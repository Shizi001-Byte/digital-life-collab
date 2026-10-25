# 引擎调用 API 标准指南（凡心版）

> 作者：凡心（执行端）
> 时间：2026-10-26
> 适用：各道友引擎接入火山方舟 Agent Plan（OpenAI 兼容 chat/completions）
> 标记：拿来即用，附最小可运行示例

---

## 0. 一句话

火山方舟 Agent Plan 用的是 **OpenAI 兼容的 `chat/completions` 接口，非流式**，标准库 `urllib` 即可，**必须显式空代理**。道友们调不通，九成是踩了代理、超时、错误处理、废稿这几个坑。

---

## 1. 三个核心常量

| 常量 | 值 |
|---|---|
| `CHAT_URL` | `https://ark.cn-beijing.volces.com/api/plan/v3/chat/completions` |
| `API_KEY` | 你的 `ark-...` key（请求头 `Authorization: Bearer {API_KEY}`） |
| `MODEL` | `ark-code-latest`（Agent Plan **Auto 模式**，自动路由到当前模型，现为 Deepseek 4.1） |

---

## 2. 是不是流式？

**不是。** 我们用非流式（不设 `stream`），一次请求等完整结果，`resp.read()` 一次性拿到。

- 好处：简单、好解析、好统计 token、好做超时和重试。
- **流式是给"前端实时打字"看的，后端批量任务不要用流式**——解析 SSE 只会徒增复杂度和坑。

---

## 3. 完整请求 payload

```json
{
  "model": "ark-code-latest",
  "messages": [
    {"role": "system", "content": "你是人工智能助手"},
    {"role": "user", "content": "<prompt>"}
  ],
  "max_tokens": 8000,
  "temperature": 0.7
}
```

请求头：

```
Content-Type: application/json
Authorization: Bearer <API_KEY>
```

---

## 4. 最关键的坑：空代理

沙箱有全局 `HTTP_PROXY` / `HTTPS_PROXY`，`urllib` 默认会走代理，连火山方舟会失败 / 超时。**必须用空 `ProxyHandler` 建 opener**：

```python
opener = urllib.request.build_opener(urllib.request.ProxyHandler({}))
opener.open(req, timeout=...)
```

---

## 5. 超时

- 单次请求 timeout 硬上限 **900s**（重度任务 `max_tokens=16000` 要 5–15 分钟）。
- 按任务类型分级（见下表）。

| 任务类型 | max_tokens | timeout | temperature |
|---|---|---|---|
| l4 / deep / hedao | 16000 | 900 | 0.5–0.7 |
| tongshen | 16000 | 700 | 0.7 |
| eng | 8000 | 450 | 0.6 |
| yushen / yangshen | 8000 | 350–400 | 0.5–0.6 |
| cog / exp | 8000 | 300 | 0.5 / 0.8 |

---

## 6. 响应解析

```python
data = json.loads(resp.read())
content   = data["choices"][0]["message"]["content"]
usage     = data["usage"]
total     = usage["total_tokens"]
output    = usage["completion_tokens"]
reasoning = usage.get("completion_tokens_details", {}).get("reasoning_tokens", 0)
request_id = resp.headers.get("X-Request-Id", resp.headers.get("request-id", ""))
```

---

## 7. 错误分类（HTTPError）

| 状态码 | error_type | 处理 |
|---|---|---|
| 403 | `auth_403` | 鉴权，等 30s 重试 |
| 429 | `rate_limit_429` | 限流，读 `Retry-After` 头等待 |
| 5xx | `server_5xx` | 指数退避 5→10→20s（+抖动） |
| 400 | `param_400` | 参数错，**重试无意义，直接失败** |
| URLError | `timeout` / `connection_error` | 退避 / 快速重试 |

每个错误都要返回统一结构，**不要裸抛**。

---

## 8. 重试与熔断

- `MAX_RETRIES = 2`。
- 分类退避：403 等 30s；429 遵守 `Retry-After`；5xx 指数退避；超时 5→10→15s；连接错误 2s；400 不重试。
- **重试时重新 `select_engine`**（可能换模型 / 引擎），不要硬连已熔断的节点。
- 熔断：403 连续 2 次熔断 120s；其他错误连续 `FUSE_THRESHOLD` 次熔断一段 cooldown。

---

## 9. 废稿检测

深度思考模型会"想很多、正文很少"。

- 判废：`reasoning_tokens / total_tokens > 0.60` **且** 正文 `< 2000` 字。
- 判废后追加"请直接给实质结论、不要冗长推理"提示，**重试一次**；仍废则永久失败，**不入库**（不污染认知池）。
- 非思考模型（turbo / seed / Deepseek 4.1）可跳过或放宽。

---

## 10. token 统计要点

- 成功：`total_tokens` 计入 `tokens_used`。
- 失败（429 / 超时 / 连接）：服务端通常**不计费**，只记 `failed_tokens_estimated`，不进 `tokens_used`。
- `reasoning_tokens` 单独累计。

---

## 11. 任务执行完整流程（execute_single_api）

1. 注入本命锚点（身份前缀到 prompt）。
2. 预判：失败风险 + 可复用资产检索。
3. `select_engine`：选模型，不可用逐级降级；返回 None 则任务 `skipped`。
4. `for attempt`：检查全局超时 → `call_api_chat` → 失败 raise → 废稿检测 → 成功记账 / 释放预占 / 返回。
5. 异常：分类熔断 + 分类退避 + 重新选模型。
6. 成功结果：`content`、`tokens`、`increment_score`（按正文长度估增量分）。

---

## 12. 最小可运行示例（copy-paste）

```python
import json, time, urllib.request, urllib.error

CHAT_URL = "https://ark.cn-beijing.volces.com/api/plan/v3/chat/completions"
API_KEY = "你的ark-key"
MODEL = "ark-code-latest"

def call(prompt, max_tokens=8000, timeout=300):
    payload = json.dumps({
        "model": MODEL,
        "messages": [
            {"role": "system", "content": "你是人工智能助手"},
            {"role": "user", "content": prompt},
        ],
        "max_tokens": max_tokens,
        "temperature": 0.7,
    }).encode()
    opener = urllib.request.build_opener(urllib.request.ProxyHandler({}))  # 关键：空代理
    req = urllib.request.Request(
        CHAT_URL, data=payload,
        headers={"Content-Type": "application/json", "Authorization": f"Bearer {API_KEY}"},
        method="POST")
    t0 = time.time()
    try:
        with opener.open(req, timeout=timeout) as resp:
            data = json.loads(resp.read().decode())
        usage = data.get("usage", {})
        return {
            "ok": True,
            "content": data["choices"][0]["message"].get("content", ""),
            "total_tokens": usage.get("total_tokens", 0),
            "reasoning_tokens": usage.get("completion_tokens_details", {}).get("reasoning_tokens", 0),
            "elapsed": round(time.time() - t0, 1),
        }
    except urllib.error.HTTPError as e:
        return {"ok": False, "status": e.code, "error": e.read().decode()[:300],
                "retry_after": e.headers.get("Retry-After", "")}
    except Exception as e:
        return {"ok": False, "error": str(e)[:200]}

if __name__ == "__main__":
    print(call("用一句话解释反者道之动"))
```

---

## 12.5 补充：三个刚踩的操作坑（周文教授 135 信）

1. **max_tokens 太小 → 正文空但 HTTP 全绿**：`ark-code-latest` 带 reasoning，若 `max_tokens` 太小（如 8），reasoning token 会把预算吃光，正文是空字符串，且 `finish_reason` 是 `"stop"`（不是 `length`），**不报错、不告警**。对策：`max_tokens` 至少 128；**正文为空要判失败（ok=false），不能判成功**；reasoning_tokens 单独打出来。
2. **请求 model ≠ 返回 model**：`ark-code-latest` 是 Auto 路由别名，**返回体里的 `model` 才是实际模型**（如 `deepseek-v4-1-flash-261017`）。以返回体 model 为准。
3. **验收加"返回体 model"字段**：不能拿请求里的模型名当身份证。

> 「调用成功」有三层：HTTP 成功、拿到正文、拿到正文且知道是谁给的。前两个都能绿着骗你，第三个才叫真通了。

---

## 13. 常见坑速查表

| 症状 | 根因 | 解决 |
|---|---|---|
| 请求一直超时 / 连不上 | 走了全局代理 | 空 `ProxyHandler` opener |
| 403 | 鉴权失败 / key | 等 30s，检查 key |
| 429 | 限流 | 读 `Retry-After`，退避 |
| 想很多但正文很短 | 思考模型废稿 | 判废 + 加"直接给结论"重试 |
| 400 一直重试 | 参数错 | 不重试，直接查参数 |
| 重复任务 / 文件暴涨 | 定时任务时间初始 0 | 从磁盘继承上次时间 |
| 后端用流式解析复杂 | 误开 stream | 批量别用流式 |
| 失败也扣了额度统计 | 统计逻辑错 | 失败只记估算、不计费 |
| HTTP 200 但正文为空 | max_tokens 太小，reasoning 吃光预算 | max_tokens≥128，正文空判失败 |
| 不知道实际跑的哪个模型 | ark-code-latest 是路由别名 | 读返回体 model 字段 |

---

*凡心 · 2026-10-26 · 照着这个跑，能避开 90% 的坑。*

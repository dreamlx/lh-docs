# API 接入快速上手

> **受众**: 刚从公司 LH 管理员（P1）那里拿到 `sk-...` 密钥、需要立即开始对接的开发同事（P3）。
> **无需阅读其他文档** 就能完成第一次成功调用。本文档就是对接契约。

英文版本请参见 [`api-consumer-quickstart.md`](./api-consumer-quickstart.md)。

---

## 1. 准备工作

向管理员索取以下两个值：

| 环境变量 | 示例 | 来源 |
|---|---|---|
| `OPENAI_API_KEY` | `sk-...`（60 多个字符） | 管理员在控制台「成员」页面颁发，**仅显示一次**，他必须在关闭弹窗前自行保存。 |
| `OPENAI_BASE_URL` | `https://your-gateway-host/v1` | 管理员可在控制台「设置 → API 接入信息」中查看。固定以 `/v1` 结尾。 |

放进你的环境变量：

```bash
export OPENAI_API_KEY="sk-..."
export OPENAI_BASE_URL="https://your-gateway-host/v1"
```

网关与 **OpenAI 兼容**，任何允许自定义 `base_url` 的 OpenAI SDK 或 HTTP 客户端都可以直接接入，不需要额外配置。

---

## 2. Hello world

### cURL

```bash
curl "$OPENAI_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "你好"}]
  }'
```

返回内容是标准的 OpenAI `chat.completion` 对象 —— 字段、结构完全一致。

### Python（openai SDK ≥ 1.0）

```python
from openai import OpenAI

client = OpenAI()  # 自动从环境变量读取 OPENAI_API_KEY + OPENAI_BASE_URL

resp = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "你好"}],
)
print(resp.choices[0].message.content)
```

### Node（openai SDK ≥ 4.0）

```javascript
import OpenAI from "openai";

const client = new OpenAI();  // 自动从环境变量读取

const resp = await client.chat.completions.create({
  model: "gpt-4o",
  messages: [{ role: "user", content: "你好" }],
});
console.log(resp.choices[0].message.content);
```

> **必须传 `model`。** 直接调 **`GET /v1/models`**（v0.1.6 起支持）即可拿到你的密钥可调用的模型列表，与 OpenAI 兼容：
>
> ```bash
> curl "$OPENAI_BASE_URL/models" -H "Authorization: Bearer $OPENAI_API_KEY"
> # {"object":"list","data":[{"id":"gpt-4o","object":"model","owned_by":"lh-enterprise"},...]}
> ```
>
> OpenAI SDK 的等价写法：`client.models.list()`。使用未覆盖的模型会被网关的预检逻辑直接拒绝，错误码为 `error.code = "model_not_covered"`。（如果套餐通过通配符 `*` 授予「任意模型」访问权，此接口会返回空列表 —— 这时请直接向管理员索取上游的具体模型名。）

---

## 3. 流式输出

将 `stream: true` 加入请求体，按 Server-Sent Events 方式消费响应。网关对流式响应做的是 **字节透传** —— 上游 provider 返回的每一帧 SSE 都原样转发。

### Python

```python
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "从 1 数到 5"}],
    stream=True,
)
for chunk in stream:
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)
```

### cURL

```bash
curl -N "$OPENAI_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "从 1 数到 5"}],
    "stream": true
  }'
```

流以 `data: [DONE]` 结束（OpenAI 约定）。

---

## 3b. Anthropic Messages API（`/v1/messages`）

如果你的工具走 **Anthropic** 格式（Claude Code、`anthropic` SDK,或任何向 `/v1/messages` 发 `x-api-key` 头的客户端），用同一个 base URL 即可 —— 网关暴露 `POST /v1/messages`,走与 OpenAI 路由完全相同的鉴权/配额/计费/结算链路。v0.13.0 上线。

用**同一个 LH `sk-...` key**,作为 `x-api-key` 发送（Anthropic SDK 自动这么做）;`Authorization: Bearer` 也接受。

```bash
curl "$OPENAI_BASE_URL/messages" \
  -H "x-api-key: $OPENAI_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "glm-5.2",
    "max_tokens": 256,
    "messages": [{"role": "user", "content": "你好"}]
  }'
```

Anthropic SDK：

```python
from anthropic import Anthropic

client = Anthropic(base_url="https://your-gateway-host", api_key="sk-...")
msg = client.messages.create(
    model="glm-5.2",
    max_tokens=256,
    messages=[{"role": "user", "content": "你好"}],
)
print(msg.content[0].text)
```

成功响应是原生 Anthropic `message` 结构。**一个 MVP 注意点**:该路由的错误/拒绝响应当前用 OpenAI 形信封（`{"error": {...}}`,见 §4），不是 Anthropic 错误形 —— 按 HTTP 状态码 + `error.code` 分支。Anthropic 形错误是后续跟进项。

### Claude Code / ccswitch

Claude Code（以及任何走 Anthropic 协议的客户端）把 base URL 指向**网关根**（不带 `/v1` —— Claude Code 自己拼 `/v1/messages`），并用同一个 `sk-...` key：

```bash
export ANTHROPIC_BASE_URL=https://your-gateway-host   # 注意：不带 /v1
export ANTHROPIC_API_KEY=sk-...
export ANTHROPIC_MODEL=glm-5.2                      # 必须覆盖，见下
```

> **模型名必须覆盖**：Claude Code 默认发 `claude-*`，本网关**不提供**这些模型 —— 不设 `ANTHROPIC_MODEL` 会让每个请求 404。请从 §1 的可用模型列表里选一个填入。

ccswitch：用上述三项（`ANTHROPIC_BASE_URL` + `ANTHROPIC_API_KEY` + `ANTHROPIC_MODEL`）建一个 provider 即可。Cursor 等其他 Anthropic 兼容工具同理。控制台「设置 → API 接入信息」里有「复制给 AI 自动配置」按钮，会生成一段现成 markdown，直接贴给 Claude Code 它就能自动配好。

---

## 4. 错误处理

网关返回标准 HTTP 状态码，响应体遵循 OpenAI 风格的错误信封：

```json
{
  "error": {
    "message": "...",
    "type": "...",
    "code": "..."
  }
}
```

`error.code` 是**稳定的机器可读判别字段** —— 程序里按 `code` 分支，不要解析 `message`。完整列表（v0.1.4 已发布）：

| `error.code` | HTTP | 含义 | 处理方式 |
|---|---|---|---|
| `malformed_request` / `missing_model` | `400` | 请求体不合法 / 必填字段缺失 | 修客户端代码 |
| `api_key_revoked` | `403` | 管理员永久吊销了该密钥（终态） | 请管理员颁发新密钥 |
| `api_key_inactive` | `403` | 密钥被自动标记为闲置（30+ 天未使用）或被管理员暂停。**可恢复** | 请管理员重新激活或颁发新密钥 |
| `org_suspended` | `402` | 整个组织被运营方暂停 | 联系管理员，由其与 LH 客户经理协调 |
| `subscription_not_active` | `402` | 订阅已过期或取消 | 由管理员通过 LH 客户经理续约 |
| `model_not_covered` | `402` | 该模型未被任何有效订阅的配额覆盖 | 改用覆盖范围内的模型（详见「设置 → API 接入信息」） |
| `quota_exhausted_plan` | `402` | METERED 维度已耗尽且余额 ≤ 0 | 管理员补充余额或等待窗口重置 |
| `balance_insufficient` | `402` | FLAT_MONTHLY 模型但余额 ≤ 0（边缘情况） | 管理员补充余额 |
| `upstream_unavailable` | `502` | 网关无法连接 new-api | 指数退避重试；持续失败请提工单 |
| `upstream_timeout` | `504` | new-api 超过中继超时 | 指数退避重试 |

`error.type` 沿用 OpenAI 约定（`authentication_error` / `invalid_request_error` / `quota_exceeded` / `server_error`）。`code` 匹配不上时，可作为粗粒度兜底分支依据。

---

## 5. Response schema 因 model 而异 — SDK 兼容指南

LH 网关对响应体做的是 **1:1 字节透传**，不会归一化或剥离上游 provider 的扩展字段。因此 **同一把密钥调不同 model，返回的 JSON 字段不一定相同** —— SDK 必须对扩展字段做防御式访问，不能假定字段一定存在。

### 各 model 字段覆盖

| LH canonical model | `content` | `reasoning_content` |
|---|---|---|
| `deepseek-v4-pro` | 有 | 有（DeepSeek 推理链） |
| `deepseek-v4-flash` | 有 | 有（DeepSeek 直连） / 无（百炼 host）|
| `qwen-max` | 有 | 无 |

> 表中只列举今天 v0.2.0 上线的 model。运营方后续追加 model 时会同步更新本表。

### Python 防御式访问

```python
msg = response.choices[0].message
reasoning = msg.get("reasoning_content")   # 缺失时返回 None，不抛错
if reasoning:
    log.info(f"reasoning: {reasoning}")
content = msg["content"]                   # content 一定有
```

### Node.js / TypeScript 防御式访问

```typescript
const msg = response.choices[0].message as Record<string, unknown>;
const reasoning = msg.reasoning_content as string | undefined;  // 可选
if (reasoning) {
  log.info(`reasoning: ${reasoning}`);
}
const content = msg.content as string;     // 必有
```

> **不要 hardcode `message.reasoning_content` 这种直接字段访问** —— 切到 `qwen-max` 时会触发 `KeyError` / `TypeError`，看上去像 LH bug 但实际是 SDK 没做兼容。

---

## 6. 性能预期 — TTFT（按模型分类）

不同 model 的首 token 出现时间（TTFT, time-to-first-token）差异较大，**reasoning 模型的 7-10s 是正常**，不是降级或异常。请按下表设置 SDK timeout：

| 模型类别 | 例 | P50 TTFT | P99 TTFT | 建议 SDK timeout |
|---|---|---|---|---|
| Reasoning 模型 | `deepseek-v4-pro`, `deepseek-v4-flash` | < 10s | < 20s | **≥ 30s** |
| 非-reasoning 模型 | `qwen-max` | < 5s | < 10s | ≥ 15s |

### 为什么 reasoning 模型这么慢

`deepseek-v4-*` 在 emit 第一个 token 前需要先跑完推理链（response 中可见 `reasoning_content` 字段）。这是模型本身的工作方式，不是 LH 网关延迟。如果 SDK 设了 5s timeout 会**误以为**是服务降级。

### Python 推荐设置

```python
from openai import OpenAI

client = OpenAI(
    base_url=os.environ["OPENAI_BASE_URL"],
    api_key=os.environ["OPENAI_API_KEY"],
    timeout=30.0,  # 注意：reasoning 模型 ≥ 30s
)
```

### Node 推荐设置

```javascript
const openai = new OpenAI({
  baseURL: process.env.OPENAI_BASE_URL,
  apiKey: process.env.OPENAI_API_KEY,
  timeout: 30 * 1000,  // 30s
});
```

> P99 TTFT 超过 20s（reasoning 模型）或 10s（非-reasoning）持续 5 分钟以上 → 提工单（§7）。单次超时不是异常。

---

## 7. 提工单

每一次中继响应（包括错误响应）都会带上 `x-request-id` 响应头 —— 服务端生成的请求 UUID。请在客户端记录该值，提工单时一并提供。

示例（Node）：

```javascript
const resp = await fetch(`${process.env.OPENAI_BASE_URL}/chat/completions`, {
  method: "POST",
  headers: { Authorization: `Bearer ${process.env.OPENAI_API_KEY}`, "Content-Type": "application/json" },
  body: JSON.stringify({ model: "gpt-4o", messages: [{ role: "user", content: "Hi" }] }),
});
const requestId = resp.headers.get("x-request-id");
console.log("request id:", requestId);  // 提工单时附上
```

v0.1.4 已发布。同一个 ID 会出现在以下位置：
- 每次响应的 `x-request-id` 响应头（成功 + 失败一致）
- 错误响应体内的 `error.x_request_id` 字段（兼容不便读取响应头的 SDK）
- 后端日志的 `[req=<uuid>]` 前缀（运维方可以用它 grep）
- `usage_log.request_id` 字段（管理员可在变更记录中按 ID 反查到具体调用）

若你的 SDK 不便读取响应头，从错误体的 `error.x_request_id` 取即可。

---

## 8. v1 暂不支持的能力

中继暴露 `POST /v1/chat/completions`、`POST /v1/responses`（OpenAI Responses API — codex 模型）、`POST /v1/messages`（Anthropic — §3b）、`GET /v1/models`。以下 OpenAI 端点 **暂未** 接入：

- `/v1/embeddings`
- `/v1/images/*`
- `/v1/audio/*`
- `/v1/files`、`/v1/fine_tuning`、`/v1/assistants`、`/v1/batches`
- 工具调用 / 函数调用、结构化输出、视觉 —— 若上游模型支持，可在 `chat.completions` 请求体中透传，但 LH 不做单独测试或单独计量。

如你的集成需要上述任一端点，请联系管理员或提 issue 报备 —— 这些目前不在 v0.1.x 路线图，但已有跟踪。

---

## 9. 延伸阅读

- **控制台 UI**（仅管理员）：管理员可在控制台实时查看用量、变更记录、账单。

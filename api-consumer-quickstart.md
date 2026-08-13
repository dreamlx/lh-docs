# API consumer quickstart

> **Audience**: developers (P3) who have just been handed an `sk-...` key by their company's LH admin (P1) and need to start integrating right away.
> **You will not need to read other documentation** to get to a working call. This doc is the integration contract.

Looking for the Chinese version? See [`api-consumer-quickstart.zh.md`](./api-consumer-quickstart.zh.md).

---

## 1. Setup

You need two values from your admin:

| Variable | Example | Where to get it |
|---|---|---|
| `OPENAI_API_KEY` | `sk-...` (60+ chars) | Your LH admin issues this from the **Members** page in the console. It's shown **once** — they must save it before closing the dialog. |
| `OPENAI_BASE_URL` | `https://your-gateway-host/v1` | Visible to the admin under **Settings → API Access**. Always ends with `/v1`. |

Drop them into your environment:

```bash
export OPENAI_API_KEY="sk-..."
export OPENAI_BASE_URL="https://your-gateway-host/v1"
```

The relay is **OpenAI-compatible**, so any OpenAI SDK or HTTP client that supports a custom `base_url` works without further configuration.

---

## 2. Hello world

### cURL

```bash
curl "$OPENAI_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

The response is the OpenAI `chat.completion` object — same fields, same shape.

### Python (openai SDK ≥ 1.0)

```python
from openai import OpenAI

client = OpenAI()  # picks up OPENAI_API_KEY + OPENAI_BASE_URL from env

resp = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
)
print(resp.choices[0].message.content)
```

### Node (openai SDK ≥ 4.0)

```javascript
import OpenAI from "openai";

const client = new OpenAI();  // env vars pick up automatically

const resp = await client.chat.completions.create({
  model: "gpt-4o",
  messages: [{ role: "user", content: "Hello" }],
});
console.log(resp.choices[0].message.content);
```

> **`model` is required.** Discover the models your key can call by calling **`GET /v1/models`** (shipped in v0.1.6). The response is OpenAI-compatible:
>
> ```bash
> curl "$OPENAI_BASE_URL/models" -H "Authorization: Bearer $OPENAI_API_KEY"
> # {"object":"list","data":[{"id":"gpt-4o","object":"model","owned_by":"lh-enterprise"},...]}
> ```
>
> Equivalent in the OpenAI SDKs: `client.models.list()`. Calls with an uncovered model are rejected by the gateway's pre-check with `error.code = "model_not_covered"`. (Plans that grant access to "any model" via wildcard quotas will return an empty list here — ask your admin directly for the upstream model name in that case.)

---

## 3. Streaming

Set `stream: true` and consume the response as Server-Sent Events. The gateway is a **byte pass-through** for streaming — every SSE frame the upstream provider produces is forwarded as-is.

### Python

```python
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Count to 5"}],
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
    "messages": [{"role": "user", "content": "Count to 5"}],
    "stream": true
  }'
```

The stream ends with the conventional `data: [DONE]` sentinel.

---

## 3b. Anthropic Messages API (`/v1/messages`)

If your tooling speaks the **Anthropic** wire format (Claude Code, the `anthropic` SDK, or any client that posts `/v1/messages` with an `x-api-key` header), point it at the same base URL — the gateway exposes `POST /v1/messages` and runs it through the identical auth / quota / billing / settle pipeline as the OpenAI routes. Shipped in v0.13.0.

Use **the same LH `sk-...` key**, sent as `x-api-key` (Anthropic SDKs do this automatically); `Authorization: Bearer` also works.

```bash
curl "$OPENAI_BASE_URL/messages" \
  -H "x-api-key: $OPENAI_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "glm-5.2",
    "max_tokens": 256,
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

Anthropic SDK:

```python
from anthropic import Anthropic

client = Anthropic(base_url="https://your-gateway-host", api_key="sk-...")
msg = client.messages.create(
    model="glm-5.2",
    max_tokens=256,
    messages=[{"role": "user", "content": "Hello"}],
)
print(msg.content[0].text)
```

The success response is the native Anthropic `message` shape. **One caveat (MVP)**: error/reject responses on this route currently use the OpenAI-shaped envelope (`{"error": {...}}`, see §4), not the Anthropic error shape — branch on the HTTP status + `error.code`. Anthropic-shaped errors are a fast-follow.

### Claude Code / ccswitch

Claude Code (and any Anthropic-protocol client) points its base URL at the **gateway root** (no `/v1` — Claude Code appends `/v1/messages` itself) and uses the same `sk-...` key:

```bash
export ANTHROPIC_BASE_URL=https://your-gateway-host   # NOTE: no /v1
export ANTHROPIC_API_KEY=sk-...
export ANTHROPIC_MODEL=glm-5.2                      # must override — see below
```

> **You must override the model name.** Claude Code sends `claude-*` by default, which this gateway does **not** serve — leaving `ANTHROPIC_MODEL` unset makes every request 404. Pick one from the supported model list in §1.

ccswitch: create a provider with the three values above (`ANTHROPIC_BASE_URL` + `ANTHROPIC_API_KEY` + `ANTHROPIC_MODEL`). Other Anthropic-compatible tools (Cursor, etc.) work the same way. The console's **Settings → API Access** card has a **"Copy AI setup"** button that emits a ready-to-paste markdown — feed it to Claude Code and it auto-configures the gateway.

---

## 4. Errors

The relay returns standard HTTP status codes. The body shape will follow an OpenAI-style envelope:

```json
{
  "error": {
    "message": "...",
    "type": "...",
    "code": "..."
  }
}
```

The `error.code` field is the **stable machine-readable discriminator** — branch on it programmatically rather than parsing `message`. The full taxonomy shipped in v0.1.4:

| `error.code` | HTTP | What it means | What to do |
|---|---|---|---|
| `malformed_request` / `missing_model` | `400` | Body invalid / required field absent | Fix client code. |
| `api_key_revoked` | `403` | Admin permanently revoked this key (terminal) | Ask admin to issue a new key. |
| `api_key_inactive` | `403` | Key auto-marked dormant (30+ d unused) OR admin parked it. **Reversible** | Ask admin to reactivate or issue a fresh key. |
| `org_suspended` | `402` | Whole org cut off by operator | Contact your admin; they escalate to LH. |
| `subscription_not_active` | `402` | Sub expired / cancelled | Admin renews via LH account manager. |
| `model_not_covered` | `402` | Model not in any active sub's quota | Use a covered model (Settings → API Access). |
| `quota_exhausted_plan` | `402` | METERED dim exhausted AND balance ≤ 0 | Admin refreshes balance or waits for window reset. |
| `balance_insufficient` | `402` | FLAT_MONTHLY model but balance ≤ 0 (edge case) | Admin refreshes balance. |
| `upstream_unavailable` | `502` | Gateway can't reach new-api | Retry with backoff; file ticket if persistent. |
| `upstream_timeout` | `504` | new-api exceeded relay timeout | Retry with backoff. |

`error.type` follows OpenAI convention (`authentication_error` / `invalid_request_error` / `quota_exceeded` / `server_error`). Useful as a coarse-grained backup if you can't match an exact `code`.

---

## 5. Filing a support ticket

Every relay response (including error responses) carries an `x-request-id` HTTP header — a server-generated UUID for the request. Capture it on the client side and include it whenever you contact support.

Example (Node):

```javascript
const resp = await fetch(`${process.env.OPENAI_BASE_URL}/chat/completions`, {
  method: "POST",
  headers: { Authorization: `Bearer ${process.env.OPENAI_API_KEY}`, "Content-Type": "application/json" },
  body: JSON.stringify({ model: "gpt-4o", messages: [{ role: "user", content: "Hi" }] }),
});
const requestId = resp.headers.get("x-request-id");
console.log("request id:", requestId);  // include this in support tickets
```

Shipped in v0.1.4. The same ID appears in:
- the response header `x-request-id` on every call (success + error)
- the `error.x_request_id` field inside the error body (for SDKs that don't surface headers)
- the backend logs as `[req=<uuid>]` prefix (operator can grep)
- the `usage_log.request_id` column (admin can find your specific call from Activity feed)

If your SDK doesn't surface headers, capture the field from the error body instead.

---

## 6. What is NOT supported (v1)

The relay exposes `POST /v1/chat/completions`, `POST /v1/responses` (OpenAI Responses API — codex models), `POST /v1/messages` (Anthropic — §3b), and `GET /v1/models`. The following OpenAI endpoints are **not yet** proxied:

- `/v1/embeddings`
- `/v1/images/*`
- `/v1/audio/*`
- `/v1/files`, `/v1/fine_tuning`, `/v1/assistants`, `/v1/batches`
- Tool / function calling, structured outputs, vision — these are passed through inside `chat.completions` if the upstream model supports them, but they are not separately tested or quota-tracked by LH.

If your integration needs one of the unsupported endpoints, file an issue or talk to your admin — these are not on the v0.1.x roadmap but are tracked.

---

## 7. Where to look next

- **Console UI** (admin only) — your admin can see usage, activity, billing in real time at the console URL.

# LH Enterprise — Customer API Docs

Public-facing API documentation for LH Enterprise customers (contracted developers, "P3"). Public so developers can read it without console login.

## Contents

- [API 接入快速上手（中文）](./api-consumer-quickstart.zh.md) — 拿到 `sk-...` 密钥后的对接指南
- [API consumer quickstart (English)](./api-consumer-quickstart.md)
- [OpenAPI spec](./api/openapi.yaml)

## Notes

- The gateway base URL (`OPENAI_BASE_URL`) is **per-customer** — get it from your LH admin (console → Settings → API Access). The `your-gateway-host` placeholder in these docs is illustrative only.
- This repo holds **only** customer-facing docs. Internal architecture / ops / ADRs live in the private source repo.

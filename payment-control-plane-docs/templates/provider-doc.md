---
name: <provider-name>
type: provider
repos:
  - 3rd-payment/controller/<name>
  - que_payment/services/<name>
status: active | deprecated
country: TH | ID | ...
operations: [payin, payout, balance, transaction-check, callback]
last_reviewed: YYYY-MM-DD
---

# Provider: `<provider-name>`

> **หนึ่งบรรทัด:** <provider ทำอะไร, payin/payout/QR, สำหรับประเทศอะไร>

## 1. Identity

- Provider type: bank gateway / aggregator / QR / peer-to-peer
- Country / currency: ...
- Onboarding date: ...
- ผู้ติดต่อ (technical): ...

## 2. Operations Supported

| Op | Supported | Notes |
|---|---|---|
| Payin (deposit) | ✅/❌ | QR / direct transfer |
| Payout (withdraw) | ✅/❌ | |
| Balance check | ✅/❌ | |
| Transaction check | ✅/❌ | |
| Callback (push) | ✅/❌ | webhook signature: ... |

## 3. Auth Model

- Method: HMAC / API key / JWT / mTLS
- Key rotation: manual / auto
- Signing fields: `<list>`
- เก็บ secret ที่: env / DB `config_payment` / vault

## 4. Endpoints (จาก provider → ใช้ใน adapter)

| Op | URL | Method | Headers สำคัญ |
|---|---|---|---|
| Create payin | https://... | POST | X-Sign, X-Merchant |
| Query | https://... | GET | |
| Withdraw | https://... | POST | |

## 5. Callback Contract

- URL ที่ provider ยิงกลับมาเรา: `POST /api/v2/callback-payin/<payment_code>`
- IP whitelist: ✅/❌ (`middleware.IPWhitelist`)
- Signature verify: ...
- Retry policy ของ provider: ...

## 6. Idempotency

- Idempotency key: ...
- ถ้า provider ยิง callback ซ้ำ → ทำยังไง

## 7. Quirks / Footguns

- ⚠️ ...

## 8. Files

- 3rd-payment side: `controller/<name>/`
  - Deposit handler: `<file>:NN`
  - Callback handler: `<file>:NN`
- que_payment side: `services/<name>/`
  - Worker: `<file>:NN`

## 9. Comparison to Sibling Providers

มี pattern อะไรเหมือน/ต่าง provider อื่นบ้าง:
- เหมือน: ...
- ต่าง: ...

## 10. Rewrite Notes

- 🟢 Keep: ...
- 🔴 Drop: ...
- 🟡 Reshape into common adapter interface: ...

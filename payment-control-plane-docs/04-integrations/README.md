---
last_reviewed: 2026-06-10
---

# 04-integrations

ระบบภายนอกและ contract ที่ระบบนี้ผูกพัน

## ที่เขียนแล้ว

- ✅ `merchant-api.md` — request/response schema ของ API ที่ merchant เรียก (**contract ห้ามพัง #1**)
- ✅ `office-webhook.md` — payload + retry ของ webhook ที่เรายิงไป merchant office (**contract ห้ามพัง #2**)

## ที่ควรเขียนเพิ่ม (เมื่อ rewrite แตะ)

- `rabbitmq.md` — connection topology (3rd ดึง URL จาก config ใน Mongo!, que ใช้ env)
- `bank-gateway.md` — `/bank-gateway/*` namespace (KBiz/SCB direct)
- `encryption-endpoint.md` — external Node.js crypto service ที่ cloudpay/visapay/peer2pay/bitpayz พึ่ง (env `ENCRYPTION_ENDPOINT`)
- `telegram-bots.md` — error bot vs business bot
- `verify-slip.md` — external slip verification (env `URL_VERIFYSLIP`)

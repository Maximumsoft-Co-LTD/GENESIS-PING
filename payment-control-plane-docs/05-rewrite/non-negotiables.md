---
last_reviewed: 2026-06-10
---

# Non-Negotiables (ของห้ามทิ้ง)

ของที่ทำงานได้ดีอยู่แล้ว — ห้ามตัดโดยไม่มี ADR:

## Observability
- ✅ OTel propagation ผ่าน AMQP headers (`amqpcarrier`) — trace ต่อเนื่อง HTTP→MQ→worker
- ✅ Severity 1-4 Telegram + dedup 5 นาที + on-call แยก chat
- ✅ Async batch event writer (1s/1000 rows → MongoDB `monitor_payment` วันนี้; backend ใหม่ตัดสินใจเป็น ADR)
- ✅ Error classifier (error → severity + remediation hint)

## Reliability / Safety
- ✅ Per-provider queue isolation (1 queue / 1 provider) — กัน noisy neighbor
- ✅ Distributed lock ก่อน mutate (3rd-payment ทำอยู่ — **rewrite ต้องขยายไปฝั่ง worker ที่วันนี้ไม่มี**)
- ✅ Unique sparse index บน `hash` (qr_payment / withdraw_statement / adjust_statement) — กลไกกันรายการซ้ำระดับ DB
- ✅ Compensating reverse เมื่อ provider call fail (คืน balance + status 13)
- ✅ Shutdown manager ของ que_payment — ขยายใช้ทั้งระบบ
- ✅ IP whitelist สำหรับ inbound callback (ขั้นต่ำ — ควรเสริม signature verify)

## Business contracts (ดู `04-integrations/`)
- ✅ Merchant API wire format ต่อ endpoint (envelope `{code,msg,data,error}`)
- ✅ Webhook success criteria: merchant ตอบ `code:0` หรือ `msg:"Success"`
- ✅ Two-phase withdraw (create → confirm) — merchant flow ผูกกับมันแล้ว
- ✅ Duplicate payin คืน order เดิม (`new_order:false`)
- ✅ `comment_timeline` audit trail บน withdraw_statement
- ✅ แยก order status กับ callback delivery status คนละ field

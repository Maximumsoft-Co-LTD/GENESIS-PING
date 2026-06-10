---
last_reviewed: 2026-06-10
---

# Glossary

ศัพท์ที่ใช้ในระบบ — ใช้ให้ตรงกันทั้ง Dev และ AI

| Term | ความหมาย |
|---|---|
| **payin / deposit** | การรับเงินเข้าจากลูกค้าปลายทาง (merchant's customer → merchant) |
| **payout / withdraw** | การโอนเงินออกจาก merchant ไปยังบัญชีปลายทาง |
| **merchant** | เว็บ/แอป ลูกค้าของเราที่ใช้ระบบนี้รับ-จ่ายเงิน |
| **service** | ชื่อ merchant (field `req.Service` ใน payin/withdraw) |
| **office** | ระบบหลังบ้านของ merchant (มี API ที่เราเรียกกลับ — field `req.OfficeAPI`) |
| **provider / payment_code** | ตัวเชื่อมกับธนาคาร/payment gateway ฝั่งภายนอก เช่น MAANPAY, ANYPAY, PEER2PAY |
| **payment_name** | ชื่อ logical ของ provider (mapping ต่อ provider; ใช้ในชื่อ queue `QUE_PAYMENT_<PAYMENT_NAME>`) |
| **order_no** | merchant reference — ห้ามซ้ำต่อ merchant |
| **slip** | หลักฐานการโอน (รูป) จากลูกค้า → ต้อง verify ผ่าน `verifySlip` |
| **callback / webhook** | ทั้ง 2 ฝั่ง: (a) provider ยิงกลับมาแจ้งสถานะ (inbound) (b) เรายิงไปแจ้ง merchant office (outbound) |
| **statement** | record การเคลื่อนไหวเงินในบัญชี — เก็บใน MongoDB |
| **bank_summary** | ยอดสรุปเงินในแต่ละบัญชีธนาคาร — ใช้ใน dashboard |
| **thorlock** | distributed lock บน Redis DB 18 (ดู `3rd-payment/service/thorlock`) — ใช้จริงเฉพาะ 3rd-payment; ฝั่ง que_payment ถูก comment ทิ้ง |
| **que_payment (collection)** | collection ใน MongoDB ที่ใช้เป็น queue สำรอง (poll mode) |
| **QUE_PAYMENT_<NAME> (queue)** | RabbitMQ queue ของแต่ละ provider — 1 queue / 1 provider |
| **monitor_payment** | MongoDB collection ที่เก็บ error/event log + trace_id (`observability/mongodb/writer.go`) — doc รุ่นก่อนเรียกผิดว่า ClickHouse `payment_events` ซึ่งไม่มีจริง |
| **severity 1-4** | ระดับ alert: 1=info (log only), 2-3=warn/error (Telegram), 4=critical (on-call) |
| **idempotency key** | ใช้ payment_code + order_no เป็นหลัก |

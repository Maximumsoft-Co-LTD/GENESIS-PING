---
last_reviewed: 2026-06-10
---

# Kill List (ของตั้งใจทิ้ง — confirm กับ owner ก่อนทุกข้อ)

## Architecture
- ❌ Provider code ซ้ำ 2 repo (59 ตัว) → adapter package เดียว
- ❌ Hybrid queue แบบปัจจุบัน — poll path แค่ re-publish เข้า MQ (`quepub/cornjob.go:202`) ไม่ใช่ fallback จริง → Outbox pattern
- ❌ PEER2PAY bypass queue (ถอนตรงจาก confirm-withdraw — `3rd/controller/withdraw.go:428-440`)
- ❌ MongoDB-as-queue polling เมื่อมี Outbox แล้ว

## Routes / Surface
- ❌ V1/V2 side-by-side (`/statement` vs `/statementV2`) — เลือก V2, audit caller, ปลด V1
- ❌ URL variants per-provider (`/callback-payout/autopeer|umpay/...`, hengpay withdraw URL) → 1 URL + dispatch by payload
- ❌ `/get-statement-withdraw-by-orderNo` GET — เก็บ POST batch
- ❌ Test endpoints ใน prod (`/update-report-test`, `/filter-report-test2`)
- ❌ Commented-out code (`routes/main.go:189-210` ซาก RouteBankTransferGateway, route เก่า speedpay `routes/main.go:18-24`)

## que_payment
- ❌ Legacy consumer V1 `StartAMQP` (entry ใช้ V2 แล้ว)
- ❌ โฟลเดอร์ `service/` (เอกพจน์) — มีแค่ sudahpay, ย้ายเข้าโครงเดียวกัน
- ❌ Sleep 5 วินาทีรอ replication หลังสร้าง bank_summary (`que/controllers/deposit.go:385`)

## Data / Schema
- ❌ Naming drift: `model` vs `models`, `repository.New` vs `NewPayment`, `CreateConnection` signature
- ❌ `is_success` ซ้ำซ้อนกับ `status` บน qr_payment
- ❌ `qr_payment_confirm` collection (เขียนจุดเดียว ไม่มีคนอ่าน — confirm ก่อนลบ)
- ❌ Per-provider credential fields ระเบิดใน `ServicePayment` struct เดียว → typed credentials per adapter
- ❌ `whitelist_ip` เป็น `interface{}`
- ❌ Webhook payload เป็น `bson.M` ไร้ struct (deposit) — ประกอบสด 2 ที่

## Security (ต้องแก้ ไม่ใช่แค่ทิ้ง)
- ❌ Outbound webhook ไม่มี signature → HMAC + key rotation
- ❌ Callback ขาเข้า ~20 provider ไม่ verify → บังคับ verify ทุกตัวที่ spec รองรับ
- ❌ maanpay callback verify ที่ถูก comment ทิ้ง (`maanpay/func.go:56-59`)
- ❌ Indo config path ข้าม X-Auth-Token (`3rd/controller/main.go:3070-3077`)
- ❌ Merchant API ไม่มี authentication จริง → API key/HMAC

## Operational
- ❌ Wallet lock `bank_summary.IsActive` ไร้ TTL — ค้างต้องแก้มือ → lease ที่หมดอายุเอง
- ❌ Retry webhook ไร้ max attempts / DLQ
- ❌ Withdraw timeout 30 วัน (`que/cronjob/_retry.go`) → configurable per provider
- ❌ Corrupted queue message ถูกทิ้งเงียบ (Nack no-requeue, no DLQ)

---
name: collection-ownership
type: feature
status: active
last_reviewed: 2026-06-10
implementations:
  - 3rd-payment: repository/main.go, repository/v2.go
  - que_payment: repository/payment.go, repository/paymentV2.go
related_workflows:
  - 03-workflows/queue-task-lifecycle.md
---

# Feature: `collection-ownership`

> **หนึ่งบรรทัด:** MongoDB 19 collections — 5 ตัวถูกเขียนจากทั้ง 2 repo (อันตรายต่อ rewrite/Outbox), 1 ตัวน่าจะตายแล้ว

## 1. Business Definition

ก่อนตัดสินใจ Outbox pattern และ migration plan ต้องรู้ว่า data ก้อนไหนถูกแชร์-เขียนข้าม repo จริง — ตารางนี้คือคำตอบ

## 2. Cross-Provider Pattern Matrix (Collection × Repo)

| Collection | 3rd-payment | que_payment | หมายเหตุ |
|---|---|---|---|
| **qr_payment** | R/W (หนัก — create, refund) | R/W (update status) | ⚠️ shared-write · core deposit · unique sparse index `hash` |
| **withdraw_statement** | R/W (หนัก) | R/W | ⚠️ shared-write · core withdraw · unique sparse index `hash` |
| **bank_summary** | R/W | R/W (**หนักกว่า** — `$inc` balance) | ⚠️ shared-write · **ledger เงินจริง** — เสี่ยงสุด |
| **logs_inc_payment** | W | W (หลัก) | ⚠️ shared-write · audit ของ balance changes |
| **service_qrpayment** | R/W (config CRUD) | R (+W 3 จุดเล็กๆ) | ⚠️ shared-write เบาๆ · ควรเป็น 3rd-only |
| que_payment | W (insert) | R/W (consume, mark done) | producer-consumer สะอาด 🟢 |
| report_payment | R | W (upsert) | สะอาด |
| adjust_statement | R/W | – | unique sparse index `hash` |
| confirm_bank_deposit | R/W | R (hengpay เช็ค) | flow ฝากตกค้าง 99 |
| confirm_statement | R | – | read-only |
| bank_account | R/W | R | bank routing |
| logs_callback_payment | R/W | R | callback audit |
| logs_callback_office | – | W | que เท่านั้น |
| logs_que_payment | – | W | que เท่านั้น (timeline ของ queue) |
| service_qrpayment_temp | W | – | staging ของ config |
| telegram_connect_provider | R/W | – | bot state |
| telegram_order_statement | R/W | – | bot logs |
| transaction_xpay_fail | W | – | xpay failure log |
| qr_payment_confirm | W (1 จุด: `3rd/controller/deposit.go:4813`) | – | ❌ **เขียนแล้วไม่มีใครอ่าน — dead candidate** |
| monitor_payment | W (event writer) | W (event writer) | observability — แยกจาก business data |

(accessor หลัก: `3rd-payment/repository/{main,v2}.go`, `que_payment/repository/{payment,paymentV2}.go`)

## 3. Common Behavior (DNA)

1. Idempotency ระดับ DB อยู่ที่ **unique sparse index บน `hash`** ของ qr_payment / withdraw_statement / adjust_statement (`3rd-payment/repository/main.go:99-103,208-213,226-231`) — 🟢 นี่คือกลไกกันซ้ำจริงของระบบ ต้องคงไว้
2. balance เปลี่ยนผ่าน `$inc` + เขียน `logs_inc_payment` คู่กันเสมอ (ledger + audit)

## 4. Variation Points (Strategy)

ไม่มี per-provider variation — collection ชุดเดียวใช้ทุก provider

## 5. Invariants

- `hash` unique ต่อ statement (กันรายการซ้ำ)
- ทุก `$inc` บน bank_summary ต้องมี row ใน logs_inc_payment
- `que_payment` row: status 0 (รอ) → 2 (กำลังทำ) → 1 (จบ)

## 6. Data Touched

(ตัวเองคือ data map — ดู section 2)

## 7. Failure Modes

| Mode | What happens | Mitigation |
|---|---|---|
| เขียน bank_summary พร้อมกัน 2 repo | race บน `$inc` — Mongo atomic ต่อ doc แต่ลำดับ business ไม่การันตี + que ไม่มี thorlock | rewrite: single writer + Outbox |
| qr_payment ถูก update ข้าม repo คนละ field | last-write-wins เงียบๆ | rewrite: แยก ownership ตาม phase ของ order |
| ลืม migrate index `hash` | รายการซ้ำทะลุ | ใส่ใน migration checklist (ทำแล้วใน doc นี้) |

## 8. Rewrite Notes

- 🟢 Keep: hash unique index, ledger+audit คู่, producer-consumer ของ que_payment collection
- 🔴 Drop: `qr_payment_confirm` (ยืนยันก่อน), shared-write บน config
- 🟡 Reshape: bank_summary → single writer (worker) ทุก mutation จาก gateway ผ่าน queue/Outbox; qr_payment/withdraw_statement → ownership ตาม phase (gateway สร้าง, worker เปลี่ยน status)
- ข้อมูลนี้สนับสนุน thesis: **Outbox จำเป็นจริง** เพราะมี shared-write 5 จุด ไม่ใช่แค่ทฤษฎี

## 9. Open Questions

- [ ] `qr_payment_confirm` ลบได้ไหม? ⚠️ TODO: verify with @owner (อาจมีระบบอื่นอ่าน)
- [ ] มีระบบอื่นนอก 2 repo เขียน collection พวกนี้ไหม (admin tool, dashboard อื่น)? — เกี่ยวกับ status 2/5 ปริศนาใน state machine ด้วย

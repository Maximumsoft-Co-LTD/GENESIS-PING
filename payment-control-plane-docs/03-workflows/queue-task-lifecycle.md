---
name: queue-task-lifecycle
type: workflow
trigger: message
status: active
involves_repos: [3rd-payment, que_payment]
last_reviewed: 2026-06-10
related_services:
  - 01-source-repos/que_payment/README.md
related_features:
  - 02-features/collection-ownership.md
  - 02-features/order-state-machine.md
---

# Workflow: queue-task-lifecycle

> **หนึ่งบรรทัด:** งาน async ทุกชนิดไหลผ่าน hybrid queue (RabbitMQ + Mongo poll) เข้า dispatcher เดียว `QueTypeSwitchV2` ที่ branch 8 ประเภท

## 1. Trigger

3rd-payment ส่งงาน 2 ทาง (บางงานทางเดียว บางงานทั้งคู่):
- **AMQP publish** → `QUE_PAYMENT_<PAYMENT_NAME>` พร้อม OTel headers (`3rd-payment/service/rabbit/que-payment.go`)
- **Insert MongoDB** collection `que_payment` status=0 (`repository/main.go:31-37`)

## 2. Actors

| Actor | Role | repo/service |
|---|---|---|
| Gateway | producer | 3rd-payment |
| AMQP consumer | เส้นทางหลัก | que_payment `rabbitmqpub/amqp.go:46-222` |
| Mongo poller | เส้นทางสำรอง ทุก 20s, pool 10, timeout 30s | que_payment `quepub/cornjob.go:38-217` |
| Dispatcher | branch by Type | `QueTypeSwitchV2` (`amqp.go:310-341`) |

## 3. Happy Path — Step by Step

1. Message/row ถูกสร้างโดย 3rd-payment
2. **เส้น AMQP**: consume (prefetch=1) → unmarshal `ReqQue` → extract OTel context → `QueTypeSwitchV2`
3. **เส้น poll**: ทุก 20s `GetListQueAllV2(PAYMENT_NAME)` ดึง status=0 → mark status=2 ("เริ่มทำรายการคิว") → worker pool 10 → `QueRunningV2` (ภายใน ⚠️ ดู Known Issues — re-publish ผ่าน `CallRabbitWithdraw`)
4. **Dispatch ตาม `Type`**:

| Type | Handler | ผลลัพธ์ |
|---|---|---|
| DEPOSIT | `DepositAgentV2` | balance +, qr_payment→1, webhook office |
| REFUND | `RefundMoneyV2` | balance −, คืน deposit ที่ fail |
| WITHDRAW / PAYOUT | `WithdrawAndPayOutV2` | lock wallet, balance −, เรียก provider |
| REPORT_DATA_CURRENT / _OLD | `UpdateReport` | upsert report_payment |
| MANUAL_CONFIRM_WITHDRAW | `ManualConfirmWithdraw` | withdraw→1 โดย admin |
| REFUND-WITHDRAW-OFFICE | `RefundWithdrawOffice` | withdraw→4 + คืน balance |

5. จบงาน → `UpdateQueByStatusV2` row → status=1 + AMQP reply `{code:0}` พร้อม trace context + Ack

## 4. Sad Paths

| ขั้นไหน fail | อาการ | ปัจจุบันทำยังไง | ถูกต้องไหม |
|---|---|---|---|
| Unmarshal fail | message เพี้ยน | Nack(requeue=false) — **ทิ้งเลย** | 🔴 ไม่มี DLQ |
| Handler error | งาน fail | classify → Telegram sev≥2 → Nack(requeue=true) | 🟡 retry ไม่จำกัด |
| RabbitMQ ล่ม | เส้นหลักหาย | Mongo poll เก็บตก (20s) + `Reconnector()` | ✅ จุดแข็งของ hybrid |
| Row ค้าง status=0/2 | งานแขวน | **ไม่มี auto-cleanup** | 🔴 ต้อง manual |

## 5. Idempotency

- Handler ระดับ business เช็ค final-state (status==1 skip) — กันการทำซ้ำจาก dual-channel (ทั้ง AMQP และ poll หยิบงานเดียวกันได้!)
- ⚠️ ไม่มี dedup ระดับ queue — ถ้างานวิ่งทั้ง 2 เส้นพร้อมกัน พึ่ง final-state check + wallet lock เท่านั้น

## 6. State Machine

`que_payment` row: `0 (รอ) → 2 (กำลังทำ — poll path เท่านั้น) → 1 (จบ)` — AMQP path ไม่ผ่าน 2

## 7. Data Touched

| Store | Collection | Op | Why |
|---|---|---|---|
| MongoDB | que_payment | INSERT(3rd) / R+UPDATE(que) | task state |
| MongoDB | logs_que_payment | INSERT | timeline ของงาน |
| RabbitMQ | QUE_PAYMENT_<NAME> + reply queue | consume/publish | งาน + ผล |
| (downstream) | ตาม Type — ดู workflow payin/payout | | |

## 8. SLA / Timing

- AMQP: เกือบทันที | Poll: สูงสุด ~20s + คิว worker (10 ตัว, timeout 30s/งาน)
- ⚠️ TODO: verify — agent พบ AMQP `Expiration` ตั้งสั้นมาก (`amqp.go:205` ฝั่ง reply) ต้องยืนยันว่ากระทบ task message ไหม

## 9. Known Issues

- ⚠️ **Poll path ไม่ได้ทำงานเอง แต่ re-publish กลับเข้า RabbitMQ** (`QueRunningV2` → `CallRabbitWithdraw` — `cornjob.go:202`) — แปลว่า "Mongo เป็น queue สำรอง" จริงๆ คือ "Mongo เป็น retry buffer ที่ feed กลับเข้า MQ" → ถ้า MQ ล่มยาว งานก็ยังไม่วิ่งอยู่ดี ⚠️ นี่ต่างจากที่ RECAP อธิบาย — สำคัญต่อ queue strategy decision (Q6)
- ⚠️ corrupted message ถูกทิ้งเงียบ (no DLQ)
- ⚠️ ไม่มี distributed lock ครอบ handler (Q9)

## 10. Rewrite Notes

- 🟢 Keep: dispatcher เดียว branch by type, per-provider queue isolation, OTel ผ่าน headers, reply contract
- 🔴 Drop: dual-channel แบบปัจจุบัน → Outbox (ตาม thesis) — ข้อมูลจาก doc นี้ยืนยันว่า hybrid ปัจจุบันไม่ได้ standalone จริง
- 🟡 Reshape: retry ไม่จำกัด → max attempts + DLQ; status=2 ของ poll path → lease + timeout

## 11. Open Questions

- [ ] Type อื่นนอกจาก 8 ตัวนี้เคยมีไหม (message เก่าใน production)?
- [ ] ทำไม poll path re-publish แทนที่จะเรียก handler ตรง — เหตุผลทาง history?

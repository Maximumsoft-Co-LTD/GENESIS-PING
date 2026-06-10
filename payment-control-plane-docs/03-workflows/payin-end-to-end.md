---
name: payin-end-to-end
type: workflow
trigger: http
status: active
involves_repos: [3rd-payment, que_payment]
last_reviewed: 2026-06-10
related_services:
  - 01-source-repos/3rd-payment/README.md
  - 01-source-repos/que_payment/README.md
related_features:
  - 02-features/provider-matrix.md
  - 02-features/order-state-machine.md
  - 02-features/collection-ownership.md
---

# Workflow: payin-end-to-end

> **หนึ่งบรรทัด:** merchant ขอ QR → ลูกค้าโอน → provider callback → worker เพิ่ม balance → แจ้ง office

## 1. Trigger

Merchant เรียก `POST /api/v2/payin` (payload ดู `04-integrations/merchant-api.md`)

## 2. Actors

| Actor | Role | repo/service |
|---|---|---|
| Merchant | initiator | external |
| Gateway | entry + provider call | 3rd-payment `controller/deposit.go` |
| Provider | สร้าง QR / แจ้งผล | external |
| Callback handler | รับผลจาก provider | 3rd-payment `controller/callback.go` |
| Worker | settle balance + แจ้ง office | que_payment `controllers/deposit.go` |
| Office | ปลายทาง webhook | external (merchant) |

## 3. Happy Path — Step by Step

**Leg A — ขอ QR (sync):**
1. **Validate** — fields + service whitelist (`deposit.go:103-135`) 
2. **Lock** — Redis lock key `{PaymentCode}:{Username}:{Amount}` 35s, ปลดด้วย defer (`deposit.go:180,195`)
3. **Duplicate check** — order เดิม (payment_code, service, username, amount) → คืน response เดิม `new_order:false` (`deposit.go:151-173`)
4. **Build order** — Status=0, CallbackStatus=3, gen OrderNo (+SHA256 บาง provider) (`deposit.go:219-240`)
5. **Health check + Provider call** — `SwitchPaymentDeposit` retry 2 ครั้ง → ได้ QR/เลขบัญชี (`deposit.go:243-351`)
6. **Persist** — insert `qr_payment` (`deposit.go:381`)
7. **Respond** — OrderNo + QR + countdown

**Leg B — provider callback (async):**
8. Provider POST `/callback-payin/:payment_code` → IP whitelist → `SwitchCallbackDeposit` per-provider parse/verify (`callback.go:112-201`)
9. ถ้า slip verify fail → update status=99 แล้วตอบ 200 (กัน provider retry) (`callback.go:211-223`)
10. **Publish** — `CallRabbitDeposit` → message Type=DEPOSIT เข้า `QUE_PAYMENT_<NAME>` + publish REPORT_DATA_CURRENT (`callback.go:249-283`)
11. ตอบ provider 200 "SUCCESS"

**Leg C — settle (que_payment):**
12. Consume → `QueTypeSwitchV2` → `DepositAgentV2` (`rabbitmqpub/amqp.go:310-312`)
13. เช็ค final-state (status==1 → skip) (`controllers/deposit.go:117-120`)
14. **Balance** — `amountInc = AmountCreate - (costs+profit)` → snapshot before_credit → `$inc` bank_summary + เขียน logs_inc_payment → mark qr_payment status=1 → snapshot after_credit (`deposit.go:348-418`) — **มี compensating reverse ถ้าขั้นไหน fail** (`deposit.go:401`)
15. **Webhook office** — goroutine fire-and-forget `UpdateAndcallbackDepositV2` → POST OfficeAPI → สำเร็จแล้ว set callback_status=1 (`deposit.go:446-600`)
16. Reply AMQP (Code=0) + Ack

## 4. Sad Paths

| ขั้นไหน fail | อาการ | ปัจจุบันทำยังไง | ถูกต้องไหม |
|---|---|---|---|
| Lock ไม่ได้ | request ซ้อน | ตอบ error ให้ merchant retry | ✅ |
| Provider call fail | ขอ QR ไม่ได้ | retry 2 แล้วตอบ 400 | ✅ |
| Callback ไม่มา | order ค้าง status=0 | **ไม่มี auto-recovery** | 🔴 deposit ไม่มี expiry |
| Publish queue fail (callback leg) | ตอบ provider 501 | รอ provider retry / manual | 🔴 จุดที่ Outbox แก้ |
| Worker error | Nack requeue + Telegram sev≥2 | retry บน broker | 🟡 ⚠️ TODO: verify message TTL — agent พบ Expiration สั้นมาก (`amqp.go:205`) |
| Office webhook fail | balance เข้าแล้ว office ไม่รู้ | cronjob retry 30 นาที (ดู `office-webhook.md`) | 🟡 ไม่มี DLQ |

## 5. Idempotency

- Leg A: duplicate check by (payment_code, service, username, amount) + unique sparse index `hash`
- Leg C: final-state check (status==1 skip) — ⚠️ ไม่มี lock ฝั่ง que (open question Q9)

## 6. State Machine

`0 → 1` หรือ `0 → 99 → 1` — ดู `02-features/order-state-machine.md`

## 7. Data Touched

| Store | Collection | Op | Why |
|---|---|---|---|
| MongoDB | qr_payment | INSERT + UPDATE ×4 | order + status + credit snapshots |
| MongoDB | bank_summary | `$inc` | ledger |
| MongoDB | logs_inc_payment, logs_callback_office | INSERT | audit |
| Redis | lock 35s | SETNX | concurrency (Leg A เท่านั้น) |
| RabbitMQ | QUE_PAYMENT_<NAME> | publish ×2 | DEPOSIT + REPORT |

## 8. SLA / Timing

- Leg A sync: ผูกกับ provider API (retry 2) — lock 35s เป็นเพดานโดยพฤตินัย
- Office webhook timeout 50-60s; retry ทุก 30 นาที window 3 ชม.

## 9. Known Issues

- ⚠️ Deposit ที่ provider ไม่ callback ค้าง status=0 ตลอดกาล
- ⚠️ Sleep 5 วินาทีหลังสร้าง bank_summary ใหม่ (`que/controllers/deposit.go:385`) — รอ replication แบบเดาเวลา
- ⚠️ Webhook ไป office เป็น goroutine fire-and-forget — ผลไม่ถูกรอ ลำดับไม่การันตี

## 10. Rewrite Notes

- 🟢 Keep: duplicate→คืน order เดิม, compensating reverse, credit snapshots
- 🔴 Drop: sleep 5s, fire-and-forget webhook
- 🟡 Reshape: callback leg → Outbox (เขียน DB + publish ใน tx เดียว), เพิ่ม order expiry

## 11. Open Questions

- [ ] countdown_time หมดแล้ว order ควร expire ไหม (ตอนนี้ไม่มีอะไรเกิดขึ้น)?

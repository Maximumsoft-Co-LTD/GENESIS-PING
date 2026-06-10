---
name: payout-end-to-end
type: workflow
trigger: http
status: active
involves_repos: [3rd-payment, que_payment]
last_reviewed: 2026-06-10
related_services:
  - 01-source-repos/3rd-payment/README.md
  - 01-source-repos/que_payment/README.md
related_features:
  - 02-features/order-state-machine.md
  - 02-features/collection-ownership.md
---

# Workflow: payout-end-to-end

> **หนึ่งบรรทัด:** สองจังหวะ create → confirm แล้วโยนเข้า queue ให้ worker หักเงิน-เรียก provider-แจ้ง office

## 1. Trigger

`POST /api/v2/create-withdraw` แล้วตามด้วย `POST /api/v2/confirm-withdraw` (two-phase โดย merchant)

## 2. Actors

| Actor | Role | repo/service |
|---|---|---|
| Merchant | initiator (2 จังหวะ) | external |
| Gateway | validate + คุมวงเงิน | 3rd-payment `controller/withdraw.go` |
| Worker | หักเงิน + เรียก provider | que_payment `controllers/withdraw.go` |
| Provider | โอนเงินจริง | external |
| Office | ปลายทาง webhook | external |

## 3. Happy Path — Step by Step

**Phase 1 — create (`withdraw.go:38-306`):**
1. Validate fields + config + ข้อจำกัด (IsNotPayout, AgentOnly) + min/max
2. Build `WithdrawStatement` Status=3, hash จาก (service+banknumber+date+amount)
3. **Balance check** จาก bank_summary (หรือ provider ตรงๆ ถ้า UseDirectBalance)
4. PAYOUT เพิ่ม: เช็ค `NextTimePayout` + วงเงิน `reportWithdraw.CanWithdraw` (`withdraw.go:230-279`)
5. Duplicate hash → reject "cannot transact again" (ห้ามซ้ำภายในวัน)
6. Insert `withdraw_statement` → ตอบ OrderNo

**Phase 2 — confirm (`withdraw.go:308-452`):**
7. โหลด statement; ถ้า status 15/16 → ตอบ "WAITWITHDRAW" (ไม่ทำซ้ำ)
8. status 3 → 2 (หรือ retry path → 5) + push comment_timeline
9. **PEER2PAY พิเศษ: เรียกถอนตรง ไม่เข้า queue** (`withdraw.go:428-440`) ⚠️
10. ที่เหลือ: Insert row เข้า collection `que_payment` (Type=WITHDRAW|PAYOUT, status=0) → ตอบ 200

**Phase 3 — worker (que_payment):**
11. ได้งานจาก **2 ทาง**: AMQP consume หรือ Mongo poll 20s (`quepub/cornjob.go:38-100`) → `WithdrawAndPayOutV2` (`controllers/withdraw.go:338`)
12. โหลด statement (retry 3×5s) + เช็ค status ∈ {2,5} + final-state check
13. เก็บ OTel trace context ใน Redis `trace:cb:{OrderNo}:{PaymentCode}` TTL 30 นาที (ไว้ match callback)
14. **Lock กระเป๋า**: `bank_summary.IsActive` 0→1 (defer ปลด) — นี่คือ lock เดียวฝั่ง que (per-service ไม่ใช่ per-order!) (`cornjob.go:167-195`)
15. **หักเงินก่อน**: `$inc -Amount` + logs_inc_payment → update status=16 + callback office
16. เรียก provider withdraw API (per-provider switch ~60 ตัว)
17. สำเร็จ → status=15 + callback office → จบรอ callback ยืนยันจาก provider (→ status=1)
18. Mark `que_payment` row status=1 + AMQP reply + Ack

## 4. Sad Paths

| ขั้นไหน fail | อาการ | ปัจจุบันทำยังไง | ถูกต้องไหม |
|---|---|---|---|
| Balance ไม่พอ | reject ตอน create | 400 | ✅ |
| InsertQue fail | confirm แล้วแต่ไม่เข้าคิว | **ค้าง status=2 ตลอด ไม่มี recovery** | 🔴 |
| Wallet ถูก lock (IsActive=1) | งานซ้อนใน service เดียวกัน | status=12 + รอ poll รอบหน้า | 🟡 serialize ทั้ง service |
| Provider API fail | เงินถูกหักแล้ว | **คืนเงิน** `$inc +Amount` + status=13 | ✅ มี compensation |
| Provider ไม่ส่ง callback ยืนยัน | ค้าง status=15 | timeout cronjob → status=4 (**30 วัน**) | 🔴 นานเกิน |
| Office webhook fail | สถานะถึง office ช้า | cronjob retry 30 นาที window 6 ชม. | 🟡 |

## 5. Idempotency

- create: unique hash (service+banknumber+date+amount) — **scope = วัน** ⚠️ ถอนซ้ำจำนวนเดียวกันวันเดียวกันโดยตั้งใจไม่ได้
- confirm: status guard (15/16 → WAITWITHDRAW)
- worker: final-state check + wallet IsActive lock

## 6. State Machine

`3 → 2|5 → 16 → 15 → 1` / fail: `→12`, `→13` (คืนเงิน), timeout `→4` — ดู `02-features/order-state-machine.md`

## 7. Data Touched

| Store | Collection | Op | Why |
|---|---|---|---|
| MongoDB | withdraw_statement | INSERT + UPDATE (status, comment_timeline) | order |
| MongoDB | bank_summary | IsActive lock + `$inc -/+` | wallet lock + ledger |
| MongoDB | que_payment | INSERT (3rd) / UPDATE (que) | task queue |
| MongoDB | logs_inc_payment | INSERT | audit |
| Redis | trace:cb:* | SET TTL 30m | OTel correlation |

## 8. SLA / Timing

- Poll fallback ทำให้ดีเลย์ได้ ~20s+ ต่องาน; wallet lock serialize งานทั้ง service
- Timeout ยืนยันจาก provider: 30 วัน (cronjob `_retry.go`)

## 9. Known Issues

- ⚠️ `bank_summary.IsActive` เป็น boolean lock ใน DB — ไม่มี TTL: ถ้า worker ตายก่อน defer ปลด → **กระเป๋าค้าง lock ต้องแก้มือ**
- ⚠️ PEER2PAY bypass queue — โค้ดถอน 2 path
- ⚠️ หักเงินก่อนเรียก provider — ถ้า crash ระหว่างกลาง เงินหายจาก ledger จนกว่า compensation จะวิ่ง

## 10. Rewrite Notes

- 🟢 Keep: two-phase create/confirm, compensation บน provider fail, comment_timeline
- 🔴 Drop: PEER2PAY special path, IsActive lock ไร้ TTL
- 🟡 Reshape: wallet lock → thorlock/lease ที่มี TTL; timeout 30 วัน → per-provider config; InsertQue → Outbox

## 11. Open Questions

- [ ] ถอนซ้ำจำนวนเดิมวันเดียวกัน (ตั้งใจ) ทำไม่ได้เพราะ hash — business ยอมรับไหม?
- [ ] ใครเคลียร์กระเป๋าที่ค้าง IsActive=1?

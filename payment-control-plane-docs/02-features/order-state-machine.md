---
name: order-state-machine
type: feature
status: active
last_reviewed: 2026-06-10
implementations:
  - 3rd-payment: controller/deposit.go, controller/callback.go, controller/withdraw.go
  - que_payment: controllers/withdraw.go, controllers/deposit.go, cronjob/_retry.go
related_workflows:
  - 03-workflows/payin-end-to-end.md
  - 03-workflows/payout-end-to-end.md
---

# Feature: `order-state-machine`

> **หนึ่งบรรทัด:** สถานะของเงินทุกบาทในระบบ — deposit ใช้ 3 สถานะ, withdraw ใช้ ~10 สถานะ (มี 2 ค่าที่หาที่มาไม่เจอ), webhook delivery แยก track ใน `callback_status`

## 1. Business Definition

state machine นี้คือ business rule ที่แท้จริงของระบบ — กระจายอยู่ใน handler/cronjob ไม่มี enum กลาง rewrite ต้อง formalize เป็น explicit state machine (REWRITE-PROMPT Phase 2 บังคับอยู่แล้ว)

## 2. Cross-Provider Pattern Matrix

State machine **เหมือนกันทุก provider** (ไม่ vary) — variation อยู่ที่ใคร trigger transition

### Deposit (`qr_payment.status`)

| Status | ความหมาย | Set โดย | Transition |
|---|---|---|---|
| 0 | กำลังทำรายการ | `3rd/controller/deposit.go:453` (create) | → 1, → 99 |
| 1 | สำเร็จ | callback handler / que `UpdateIsSuccessDepositStatementV2` (`que/controllers/deposit.go:399`) | final |
| 99 | ฝากตกค้าง รอยืนยัน manual | callback handler (`3rd/controller/callback.go:3577,3878`) เมื่อ slip verify ไม่ผ่าน | → 1 (ผ่าน `confirm-bank-deposit` โดย office) |

```
0 ──callback ok──▶ 1
└──slip verify fail──▶ 99 ──office กด confirm-bank-deposit──▶ 1
```

มี `is_success` (int32) ควบคู่กับ `status` — ถูก set พร้อมกัน ⚠️ ซ้ำซ้อน (rewrite: ยุบเหลือ field เดียว)

### Withdraw (`withdraw_statement.status`)

| Status | ความหมาย | Set โดย | Transition |
|---|---|---|---|
| 3 | สร้างรายการ | `3rd/controller/withdraw.go:585` (create-withdraw) | → 2 |
| 2 | ยืนยันแล้ว (confirm-withdraw, "ยืนยันการโอนเงิน AUTO") | `3rd/controller/withdraw.go:392` | → 16, → 12 |
| 5 | retry-confirm (confirm ซ้ำหลัง error) | `3rd/controller/withdraw.go:402` | → 16, → 12 |
| 16 | กำลังส่งรายการถอนไป provider | `que/controllers/withdraw.go:267,860` (ก่อนเรียก API) | → 15, → 13 |
| 15 | ส่งสำเร็จ รอ provider ยืนยัน | `que/controllers/withdraw.go:271,306,882` | → 1, → 4 |
| 1 | สำเร็จ | `que/controllers/withdraw.go:368` (callback ยืนยัน) | final |
| 12 | ล้มเหลว (validation/balance/API error) | `que/controllers/withdraw.go` หลายจุด | final (เงินไม่ถูกหัก หรือถูกคืนแล้ว) |
| 13 | API ล้มเหลว → คืนเงินแล้ว | `que/controllers/withdraw.go:279-300` | final |
| 4 | ยกเลิก (timeout) | `que/cronjob/_retry.go:76` | final |

```
3 ──confirm──▶ 2 ──queue──▶ 16 ──provider รับ──▶ 15 ──callback ยืนยัน──▶ 1
                 │            └─API fail─▶ 13 (คืนเงิน)   └─timeout─▶ 4
                 └─validation fail─▶ 12
(5 = วน confirm ใหม่หลัง error → เข้า 16 เหมือน 2)
```

### Callback delivery (`callback_status` — ทั้ง deposit + withdraw)

| ค่า | ความหมาย | Set โดย |
|---|---|---|
| 3 | ยังไม่ส่ง/ส่งไม่สำเร็จ (ค่าเริ่มต้น) | ตอน create (`3rd/controller/deposit.go:463`) |
| 1 | ส่งถึง office สำเร็จ | `UpdateCallbackDepositV2`/`UpdateCallbackStatusWithdraw` |

## 3. Common Behavior (DNA)

1. ทุก transition เขียน `comment_timeline` (array push) บน withdraw_statement — audit trail ในตัว 🟢 เก็บ pattern นี้
2. การเงินกับ delivery แยก track (`status` vs `callback_status`) — ถูกหลักการ 🟢
3. เงินถูกหัก/เพิ่มที่ `bank_summary` **ก่อน** ยืนยันจาก provider (หักตอนเข้า 16, คืนถ้า fail → 13)

## 4. Variation Points (Strategy)

- ค่า transition เหมือนกันทุก provider แต่ **ใคร trigger ต่าง**: บาง provider จบใน HTTP callback (3rd), ส่วนใหญ่จบใน queue worker (que)
- PEER2PAY bypass queue ทั้งหมด — เรียกถอนตรงจาก confirm-withdraw (`3rd/controller/withdraw.go:428-440`) ⚠️

## 5. Invariants

- status 1 = final เสมอ — ทุก handler เช็ค `IsSuccess==1 || Status==1` แล้ว skip (กัน replay)
- เงินใน `bank_summary` ต้อง balance กับผลรวม transition (13 ต้องคืนเท่าที่ 16 หัก)
- 99 เปลี่ยนได้ทาง manual confirm เท่านั้น

## 6. Data Touched

| Store | Object | Op |
|---|---|---|
| MongoDB | `qr_payment.status/is_success/callback_status` | W ทุก transition |
| MongoDB | `withdraw_statement.status/comment_timeline/callback_status` | W ทุก transition |
| MongoDB | `bank_summary.curent_balance` | ±เงินผูกกับ transition |

## 7. Failure Modes

| Mode | What happens | Mitigation |
|---|---|---|
| ค้างที่ 16 (provider ไม่ตอบ) | เงินถูกหักแล้ว order แขวน | cronjob `_retry.go` timeout → 4 — ⚠️ window 30 วัน (นานมาก) |
| ค้างที่ 2/5 (queue หาย) | order ไม่ถูกประมวลผล | ไม่มี auto-recovery ⚠️ — Mongo poll ช่วยได้เฉพาะ row ที่เข้า `que_payment` แล้ว |
| Double callback จาก provider | เช็ค status==1 skip — ปลอดภัยระดับ app | race ระดับ DB ดู open question (que ไม่มี lock) |

## 8. Rewrite Notes

- 🟢 Keep: comment_timeline audit, แยก delivery status, final-state check ก่อน mutate
- 🔴 Drop: `is_success` ซ้ำกับ `status`; magic numbers กระจาย
- 🟡 Reshape: enum กลาง + transition table ที่ validate ได้; timeout 30 วัน → configurable ต่อ provider
- Suggested: state machine แยก `OrderState` (สถานะเงิน) กับ `DeliveryState` (สถานะแจ้ง merchant) ชัดเจน

## 9. Open Questions

- [ ] **status 2 และ 5 ของ withdraw: ไม่พบโค้ดที่ set ค่าอื่นเข้ามา nameless guard `Status != 2 && Status != 5` มีใน que** (`que/controllers/withdraw.go:100,359`) — มี writer อื่นนอก 2 repo นี้ไหม (เช่น admin tool)? ⚠️ TODO: verify with @owner
- [ ] deposit ไม่มี error state (มีแค่ 0/1/99) — order ที่ provider ปฏิเสธค้างที่ 0 ตลอดไป? ใครเก็บกวาด?
- [ ] timeout 30 วันของ `_retry.go` — ตัวเลขนี้ตั้งใจหรือ legacy?

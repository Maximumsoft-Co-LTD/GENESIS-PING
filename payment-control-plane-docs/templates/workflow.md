---
name: <workflow-name>
type: workflow
trigger: http | callback | cron | manual | message
status: active | deprecated | proposed
involves_repos: [3rd-payment, que_payment]
last_reviewed: YYYY-MM-DD
related_services:
  - 01-source-repos/<repo>/services/<svc>.md
related_features:
  - 02-features/<feature>.md
---

# Workflow: <workflow-name>

> **หนึ่งบรรทัด:** <จุดเริ่ม → จุดจบ ของ flow นี้คืออะไร>

## 1. Trigger

ใครเริ่ม / เกิดเมื่อไหร่:
- เช่น "merchant เรียก POST /payin" / "bank ยิง callback กลับมา" / "cron ทุก 5 นาที"

Trigger payload (สรุป):
```
{ ... }
```

## 2. Actors

| Actor | Role | repo/service |
|---|---|---|
| Merchant | initiator | external |
| API Gateway | entry | 3rd-payment / routes/main.go |
| Provider Adapter | bridge to bank | que_payment / services/<name> |
| Bank | settlement | external |
| Worker | callback processor | que_payment / rabbitmqpub |

## 3. Happy Path — Step by Step

```
Merchant ──POST /payin──▶ 3rd-payment
                            │
                            ▼
                        Validate
                            │
                            ▼
                        Lock (thorlock key=...)
                            │
                            ▼
                        Provider.CreatePayin ──▶ Bank
                            │                      │
                            │◀───── orderRef ──────┘
                            ▼
                        Persist tbl_payment
                            │
                            ▼
                        Publish QUE_PAYMENT_<NAME>
                            │
                            ▼
                        Return 200 + QR
```

ทีละขั้น:
1. **Validate** — เช็คอะไรบ้าง, ทำที่ `<repo>/middleware/...:NN`
2. **Lock** — key อะไร, ปลดเมื่อไหร่ (default thorlock 180s)
3. **Provider call** — provider ไหน, timeout เท่าไหร่
4. **Persist** — collection อะไร, transaction boundary
5. **Publish** — exchange/queue/routing key
6. **Respond**

## 4. Sad Paths

| ขั้นไหน fail | อาการ | ปัจจุบันทำยังไง | ถูกต้องไหม |
|---|---|---|---|
| Validate fail | 400 | return error | ✅ |
| Lock ไม่ได้ | concurrent request | wait / fail | 🟡 |
| Provider timeout | bank ช้า | retry N ครั้ง | ⚠️ idempotent? |
| Queue down | callback หาย | ... | 🔴 |

## 5. Idempotency

- Idempotency key: `<merchant_id>:<merchant_ref>` หรือ ...
- เก็บที่ไหน: Redis / Mongo / thorlock
- ถ้าซ้ำ: return เดิม / reject / merge?

## 6. State Machine (ถ้ามี)

```
PENDING ──▶ PAID ──▶ SETTLED
   │           │
   ▼           ▼
EXPIRED     FAILED
```

ใครเปลี่ยน state อะไร — อ้างอิงโค้ด

## 7. Data Touched

| Store | Collection/Table | Op | Why |
|---|---|---|---|
| MongoDB | `tbl_payment` | INSERT | persist order |
| Redis | `lock:payin:<id>` | SETNX | concurrency |
| RabbitMQ | `QUE_PAYMENT_<NAME>` | publish | downstream |
| ClickHouse | `payment_events` | async write | event log |

## 8. SLA / Timing

- end-to-end target: < N seconds
- ขั้น slow ที่สุด: ...
- timeout settings: ...

## 9. Known Issues

- ⚠️ ...

## 10. Rewrite Notes

- 🟢 Keep: ขั้นที่ออกแบบไว้ถูกแล้ว
- 🔴 Drop: ขั้นที่ตั้งใจตัด
- 🟡 Reshape: ขั้นที่จะคงไว้แต่ทำคนละแบบ
- ผลกระทบกับ feature: `02-features/<x>.md`

## 11. Open Questions

- [ ] ...

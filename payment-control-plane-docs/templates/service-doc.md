---
name: <service-name>
repo: 3rd-payment | que_payment
type: service
status: active | deprecated | unknown
owner: <team or person>
last_reviewed: YYYY-MM-DD
source_paths:
  - <repo>/service/<name>/
related_features:
  - 02-features/<feature>.md
---

# Service: `<service-name>`

> **หนึ่งบรรทัด:** <service นี้ทำอะไร แบบที่ junior dev เข้าใจใน 10 วินาที>

## 1. Purpose — ทำไมมี service นี้

<2-4 บรรทัด: เกิดมาเพื่อแก้ปัญหาอะไร, ใครเรียก, ถ้าไม่มีจะเป็นยังไง>

## 2. Public Surface — เรียกใช้ยังไง

| Method | Function / Endpoint | Caller | หมายเหตุ |
|---|---|---|---|
| HTTP POST | `/api/v1/...` | controller X | |
| internal  | `service.DoX()` | service Y | |

Entry point: `<repo>/service/<name>/handler.go:1`

## 3. Inputs & Outputs

### Input
```json
{ ... ตัวอย่าง payload ... }
```
- field สำคัญ: ...
- field optional แต่กระทบ logic: ...

### Output
```json
{ ... }
```

### Error cases
| Error | When | Caller ทำอะไรต่อ |
|---|---|---|
| `ErrXxx` | ... | retry / fail / log only |

## 4. Core Logic — DNA

> ส่วนนี้สำคัญที่สุด — เขียนให้ "rewrite ได้โดยไม่ต้องดูโค้ดเดิม"

ขั้นตอนหลัก:
1. ...
2. ...
3. ...

Invariants (ห้ามผิด):
- ...

Side effects:
- เขียน DB: ...
- publish message ไปที่: ...
- เรียก external API: ...

## 5. Dependencies

| Type | Name | Why |
|---|---|---|
| DB | `tbl_xxx` | persist X |
| External API | KBank API | verify slip |
| Internal service | `thorlock` | distributed lock |
| Queue | `payin.callback` | downstream |

## 6. State & Concurrency

- State อยู่ที่ไหน (DB / Redis / in-memory)
- ต้อง lock อะไร? key อะไร?
- idempotent ไหม? key คืออะไร?
- ถ้าเรียกซ้ำกัน 2 ครั้งพร้อมกันจะเกิดอะไร?

## 7. Observability

- logs สำคัญ: `<file>:NN`
- metric: `payment_<name>_xxx_total`
- trace span: `<name>.Process`

## 8. Known Quirks / Footguns

- ⚠️ ...

## 9. Rewrite Notes

- 🟢 **Keep:** ส่วนที่ต้องคงไว้
- 🔴 **Drop:** ตั้งใจทิ้ง + เหตุผล
- 🟡 **Reshape:** คงไว้แต่เปลี่ยนรูป

## 10. Open Questions

- [ ] ... (รอถาม @owner)

---
name: <feature-name>
type: feature
status: active | proposed
last_reviewed: YYYY-MM-DD
implementations:
  - 3rd-payment: <path>
  - que_payment: <path>
related_workflows:
  - 03-workflows/<flow>.md
---

# Feature: `<feature-name>`

> **หนึ่งบรรทัด:** feature นี้คืออะไรในมุมธุรกิจ

## 1. Business Definition

อะไร, ทำไม, ใครได้ประโยชน์

## 2. Cross-Provider Pattern Matrix

| Provider | Variant | Notes |
|---|---|---|
| maanpay | A | uses HMAC-SHA256 |
| anypay  | A | same |
| peer2pay | B | different — uses JWT |
| ... | | |

> ตารางนี้คือสิ่งที่ rewrite ใหม่ต้อง abstract ออกมา

## 3. Common Behavior (DNA)

ส่วนที่ทุก provider ทำเหมือนกัน:
1. ...
2. ...

## 4. Variation Points (Strategy)

จุดที่ provider ต่างกัน:
- ...

## 5. Invariants

- ...

## 6. Data Touched

| Store | Object | Op |
|---|---|---|
| ... | ... | ... |

## 7. Failure Modes

| Mode | What happens | Mitigation |
|---|---|---|
| ... | ... | ... |

## 8. Rewrite Notes

- 🟢 Keep
- 🔴 Drop
- 🟡 Reshape
- Suggested abstraction:
```
type <Feature>Provider interface {
    ...
}
```

## 9. Open Questions

- [ ] ...

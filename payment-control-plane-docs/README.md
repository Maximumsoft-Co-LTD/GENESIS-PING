---
last_reviewed: 2026-06-10
---

# Payment Control Plane — Documentation

ที่เก็บความรู้ของระบบ payment 2 ตัวที่กำลังจะ rewrite:
- `3rd-payment` — HTTP front door + per-provider controller (~67 providers) + internal cronjob
- `que_payment` — RabbitMQ consumer + 3 cronjob modes + per-provider service (62 providers — **59 ตัวซ้ำกับด้านบน**)

**เป้าหมาย:** ถอด DNA ของทั้ง 2 repo ให้ Dev และ AI (Claude) เข้าใจตรงกัน ก่อนจะ rewrite ใหม่ (อาจรวมเป็น repo เดียว)

---

## เริ่มอ่านตรงไหน

| ถ้าคุณคือ... | อ่าน |
|---|---|
| คนใหม่ในทีม | `00-overview/system-context.md` → `00-overview/glossary.md` |
| Dev ที่จะ rewrite | `RECAP.md` → `02-features/` → `05-rewrite/` |
| คนที่จะใช้ AI สั่ง rewrite | `REWRITE-PROMPT.md` (copy ทั้งไฟล์) |
| Claude (AI) | `CLAUDE.md` ก่อนเสมอ |

## โครงสร้าง

```
00-overview/         ภาพรวม domain, glossary, tech stack
01-source-repos/     DNA ดิบของแต่ละ repo (per-service / per-provider)
02-features/         DNA สังเคราะห์ — feature ที่ใช้ซ้ำข้าม provider
03-workflows/        flow แบบ step-by-step ข้าม service
04-integrations/     ระบบภายนอก (bank, RabbitMQ, Redis, OTel)
05-rewrite/          เป้าหมาย rewrite (goals, non-negotiables, kill list)
06-adr/              Architecture Decision Records
templates/           template สำหรับเขียน doc ใหม่
RECAP.md             สรุป DNA + สิ่งที่ scaffold ไว้ (อ่านก่อน)
REWRITE-PROMPT.md    prompt มาตรฐานสำหรับสั่ง AI rewrite
```

## วิธีเพิ่ม doc ใหม่

1. หา template ที่ตรงประเภท (service / provider / feature / workflow / adr)
2. copy → วางในโฟลเดอร์ที่ตรงประเภท
3. เติม section ตาม template — **ห้ามลบ section ทิ้ง** ถ้าไม่มีข้อมูลใส่ `N/A — <เหตุผล>`
4. ถ้าเป็น decision สำคัญ → เพิ่ม ADR ใน `06-adr/`

## หลักการเขียน doc

- อธิบาย "ทำไม" มากกว่า "อะไร" — โค้ดบอก what อยู่แล้ว
- ใส่ link ไปยังไฟล์โค้ดต้นทางเสมอ (`repo/path/file.go:line`)
- ทุก doc ต้องมี `last_reviewed` date ใน frontmatter
- ใช้ภาษาไทยได้ — technical term เป็นอังกฤษ

## สิ่งที่ไม่เขียนในที่นี้

- โค้ดจริง (อยู่ใน repo ต้นทาง)
- credentials / API keys
- task list ที่กำลังทำ (ใช้ ClickUp/Linear)

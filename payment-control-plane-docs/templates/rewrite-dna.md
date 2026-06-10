---
project: <project-name>
status: extracting | ready | handed-off
last_reviewed: YYYY-MM-DD
source_repos:
  - <repo-1>
  - <repo-2>
deep_dive_docs: <path ไปยังโฟลเดอร์ docs ละเอียด ถ้ามี | N/A>
---

# REWRITE DNA — `<project-name>`

> **นิยาม:** เอกสารเดียวที่ AI หรือ dev ที่ไม่เคยเห็นระบบ อ่านแล้วรู้ว่า — ระบบนี้คืออะไร, อะไรคือหัวใจที่ห้ามหาย, อะไรห้ามพัง, อะไรตั้งใจทิ้ง — เพียงพอที่จะ rewrite โดยไม่ทำธุรกิจเสียหาย
>
> **กติกาการเขียน:** (1) ทุก claim ต้องมี `repo/path/file.go:NN` (2) ไม่รู้ = เขียน `⚠️ TODO: verify with @owner` ห้ามเดา (3) เขียน "ทำไม" ไม่ใช่สรุปไฟล์ (4) อัปเดต `last_reviewed` ทุกครั้ง

---

## วิธีใช้: Prompt สำหรับสั่ง AI ถอด DNA

**ขั้นตอน:** (1) copy ไฟล์ template นี้ไปวางในโปรเจคที่จะถอด (เช่น `docs/rewrite-dna.md`) (2) copy ทุกอย่างระหว่างเส้น `═══` ด้านล่างไปวางเป็น prompt แรกของ AI (Claude Code / AI ตัวไหนก็ได้ที่อ่านโค้ดได้) (3) แก้ `<path>` ให้ตรงกับที่วางไฟล์

═══════════════════════════════════════════════════════════════

คุณคือ **documentation architect + senior engineer** มีหน้าที่เดียว: ถอด DNA ของโปรเจคนี้ออกมาเป็นเอกสาร เพื่อใช้ rewrite โดยไม่ทำธุรกิจเสียหาย — คุณไม่ใช่คนสรุปไฟล์ และคุณจะไม่แก้โค้ดใดๆ (source code เป็น read-only)

**Template และวิธีทำงานทั้งหมดอยู่ที่ `<path>/rewrite-dna.md` — อ่านไฟล์นั้นให้จบก่อนเริ่ม** แล้วทำตาม "ภาคผนวก: วิธีแกะ DNA" ทีละขั้นตามลำดับ (1→9) ห้ามข้ามขั้น

กติกาบังคับ (ผลลัพธ์ทุกคนต้องตรงกัน):

1. **ทุก claim ต้องมี evidence** รูปแบบ `repo/path/file.go:NN` — claim ที่ไม่มี evidence ถือว่าไม่มีอยู่
2. **ห้ามเดาเด็ดขาด** — หาคำตอบจากโค้ดไม่ได้ให้เขียน `⚠️ TODO: verify with @owner` แล้วไปต่อ อย่าหยุดถาม
3. **อย่าเชื่อ doc เดิม** — ถ้าโปรเจคมี doc/README อยู่แล้ว ให้ตรวจทุก claim กับโค้ดจริงก่อนใช้ และรายงานทุกจุดที่ doc ไม่ตรงโค้ด
4. **ถ้าใช้ subagent ช่วยอ่าน** (แนะนำสำหรับงาน fan-out เช่น matrix) ต้อง spot-check ผลของ agent กับโค้ดจริงอย่างน้อย 2-3 จุดก่อนบันทึก — agent อ่านผิดได้
5. **ค่า status/magic number ที่หาคนกำหนดไม่เจอ, โค้ดที่ comment ทิ้ง, ของที่เขียนแล้วไม่มีคนอ่าน** — ทั้งหมดต้องลง section 11 (Landmines) ห้ามมองข้าม
6. **คำถามที่โค้ดตอบไม่ได้** (business intent, NFR, ของที่ดูเหมือน bug) — รวบรวมลง section 12 ทั้งหมด ไม่ใช่ตัดสินใจแทน owner

ผลลัพธ์ที่ต้องส่งมอบ:

- ไฟล์ `DNA.md` ที่ root ของโปรเจค (หรือ `docs/DNA.md`) กรอกครบทั้ง 14 sections ตาม template — section ที่ไม่เกี่ยวกับโปรเจคนี้ให้เขียน `N/A — <เหตุผล>` ห้ามลบ section ทิ้ง
- ถ้า section ไหนใหญ่เกิน (เช่น matrix 10+ instance, contract หลาย endpoint) ให้แตกเป็นไฟล์ย่อยข้างๆ แล้วสรุป + link จาก DNA.md
- ติ๊ก Extraction Checklist (section 0) ตามจริง — ติ๊กได้เฉพาะข้อที่ verify กับโค้ดแล้วเท่านั้น
- จบงานด้วยรายงานสั้นๆ: (a) สิ่งที่ค้นพบสำคัญสุด 3-5 ข้อ (b) Landmines ทั้งหมด (c) checklist ข้อไหนยังไม่ครบเพราะอะไร

เริ่มจากสำรวจโครงสร้างโปรเจค (ขั้นที่ 1: Surface scan) ได้เลย — ไม่ต้องถามอะไรก่อนเริ่ม

═══════════════════════════════════════════════════════════════

---

## 0. Extraction Checklist (สถานะความครบของ DNA นี้)

ติ๊กเมื่อแกะเสร็จและ **verify กับโค้ดจริงแล้ว** — DNA ที่ยังไม่ครบห้าม hand-off:

- [ ] 1. Identity — ภาพรวม + บทบาทแต่ละ repo
- [ ] 2. Ecosystem — ระบบรอบข้าง + ใครพึ่งเรา/เราพึ่งใคร
- [ ] 3. External contracts — ทุก schema ที่คนนอก integrate (in + out)
- [ ] 4. Core capabilities — function ที่เป็นธุรกิจแท้ แยกจากโค้ดประกอบ
- [ ] 5. State machines + invariants
- [ ] 6. Data ownership — ใครอ่าน/เขียน store ไหน
- [ ] 7. Variation points — สิ่งที่ vary ต่อ tenant/provider/config (ทำเป็น matrix)
- [ ] 8. Cross-cutting patterns ที่ควรเก็บ
- [ ] 9-10. Non-negotiables + Kill list
- [ ] 11. Landmines — bug แฝง/ของหาที่มาไม่เจอ
- [ ] 12. Open questions ถึง owner (พร้อมคำตอบก่อนเริ่ม rewrite)
- [ ] 13. Verification plan — พิสูจน์ยังไงว่า rewrite ถูก

---

## 1. Project Identity (ภาพรวม)

- **ระบบนี้ทำอะไร ให้ใคร:** <1-3 ประโยค — มุมธุรกิจ ไม่ใช่มุมเทคนิค>
- **อะไรไหลผ่านระบบ:** <เงิน? ข้อมูลส่วนบุคคล? คำสั่งซื้อ? — บอกระดับความเสียหายถ้าพัง>
- **Source repos:**

| Repo | บทบาท | Tech | Entry points |
|---|---|---|---|
| | | | |

- **Scale / NFR:** <throughput, latency, uptime — ถ้าไม่รู้ ⚠️ TODO: verify with @owner>

## 2. Ecosystem Map (repo และระบบที่เกี่ยวข้อง)

```
<ASCII diagram: ใครคุยกับใคร ผ่านอะไร>
```

| ระบบ/Repo | ความสัมพันธ์ | Contract | เจ้าของ | rewrite แตะได้ไหม |
|---|---|---|---|---|
| | upstream / downstream / shared-db / shared-infra | | | read-only / ต้อง coordinate / อิสระ |

## 3. External Contracts (ห้ามพังเด็ดขาด)

ทุกอย่างที่ **คนนอกทีม integrate อยู่** — เปลี่ยน field เดียวคือ breaking change:

| Contract | ทิศทาง | Consumer | Schema อยู่ที่ | ระดับความเข้มงวด |
|---|---|---|---|---|
| <API กลุ่ม X> | inbound | <ใคร> | <doc/file:line> | wire-compatible / versioned-ok |
| <webhook Y> | outbound | <ใคร> | | |
| <queue/message Z> | internal-but-shared | | | |
| <shared DB collection> | | | | |

> ถ้า schema ยาว ให้แตกเป็นไฟล์แยกแล้ว link — แต่ตารางนี้ต้องครบทุก contract

## 4. Core Business Capabilities (หน้าที่สำคัญห้ามขาด)

แยก "ธุรกิจแท้" ออกจาก "โค้ดประกอบ" — rewrite ที่ขาดข้อใดข้อหนึ่งในระดับ core = ล้มเหลว:

| Capability | ทำอะไร (มุมธุรกิจ) | Entry points (file:line) | Business rule แฝงที่มองไม่เห็นจาก signature | ระดับ |
|---|---|---|---|---|
| | | | <เช่น "duplicate คืน order เดิม", "ห้ามซ้ำภายในวัน"> | core / supporting / legacy-confirm-before-drop |

## 5. State & Invariants

**State machines** (ทุกตัวที่มี — วาด + ตาราง transition + ใครเป็นคนเปลี่ยน):

```
<state diagram>
```

| Status | ความหมาย | Set โดย (file:line) | Transitions |
|---|---|---|---|

**Invariants** (สิ่งที่ต้องจริงเสมอ ไม่ว่า state ไหน):
- <เช่น "ทุก $inc บน ledger ต้องมี audit row คู่กัน">

**Idempotency:** <กลไกกันทำซ้ำปัจจุบัน — key อะไร เก็บที่ไหน (unique index? lock? status check?)>

## 6. Data Ownership

| Store | Object | ใครเขียน | ใครอ่าน | ⚠️ shared-write? | Index/constraint ห้ามหาย |
|---|---|---|---|---|---|

- Dead data ที่พบ (เขียนแล้วไม่มีคนอ่าน): <list — confirm ก่อนทิ้ง>

## 7. Variation Points (สิ่งที่ต้อง abstract ตอน rewrite)

จุดที่พฤติกรรม **vary ตาม tenant/provider/config** — นี่คือสาเหตุที่โค้ดเดิมบวม และคือ spec ของ abstraction ใหม่:

| Variation | ค่าที่พบจริง (จาก matrix) | จำนวน case | นัยต่อ design |
|---|---|---|---|
| <เช่น auth method> | <HMAC, MD5, ...> | | <ต้องเป็น Strategy interface> |

> ถ้า variation เยอะ (10+ case) ให้ทำ matrix แยกไฟล์แล้วสรุปที่นี่ — โครงไฟล์ matrix: ตาราง instance × dimension (1 แถว/instance, dimension เป็นคอลัมน์ เช่น auth, callback, time format) + Evidence file:line ต่อแถว + สรุป common pattern / variation / rewrite notes ท้ายไฟล์

## 8. Cross-Cutting Patterns (เก็บ pattern ไม่ใช่โค้ด)

Pattern ที่พิสูจน์แล้วว่า work — rewrite ควร reimplement แนวคิดเดิม:

| Pattern | อยู่ที่ | ทำไมถึงดี |
|---|---|---|
| <เช่น trace propagation ผ่าน queue headers> | | |

## 9. Non-Negotiables ✅

ของห้ามทิ้งโดยไม่มี ADR — ทั้งเชิง technical และ business contract:
- ✅ ...

## 10. Kill List ❌

ของตั้งใจทิ้ง — **ทุกข้อต้อง confirm กับ owner ก่อน**:
- ❌ ... <พร้อมเหตุผล + ของทดแทน>

## 11. Landmines & Mysteries ⚠️

ของที่เจอระหว่างแกะแล้วอธิบายไม่ได้ หรือเป็น bug แฝง — **อันตรายสุดถ้าไม่จด เพราะ rewrite จะทำซ้ำหรือเหยียบ**:

| สิ่งที่เจอ | Evidence | สมมุติฐาน | ต้องทำอะไร |
|---|---|---|---|
| <เช่น "status 5 มี guard แต่ไม่มีใคร set"> | file:line | <มีระบบอื่นเขียน DB?> | ถาม owner ก่อน design |

## 12. Open Questions → Owner

คำถามที่โค้ดตอบไม่ได้ — **rewrite ห้ามเริ่มจนกว่าจะมีคำตอบครบ**:

1. [ ] ...
2. [ ] ...

## 13. Verification Plan (พิสูจน์ว่า rewrite ถูกต้อง)

- **Contract tests:** <เทียบ request/response ของระบบใหม่กับ schema ใน section 3 ยังไง>
- **Parity / shadow run:** <รันคู่ production ได้ไหม เทียบผลยังไง>
- **Data reconciliation:** <invariant ใน section 5 เช็คอัตโนมัติได้ไหม>
- **Migration gate per unit:** <เงื่อนไขก่อน cutover แต่ละ provider/tenant/module>

## 14. Deep-Dive Docs (ถ้าโปรเจคใหญ่)

| หัวข้อ | ไฟล์ |
|---|---|
| | |

---

## ภาคผนวก: วิธีแกะ DNA (methodology — ใช้ซ้ำกับโปรเจคอื่น)

ลำดับที่ได้ผลจริง (แต่ละขั้น verify กับโค้ดก่อนเชื่อ doc เดิม):

1. **Surface scan** — entry points ทั้งหมด (routes, queues, cronjobs, env-driven modes) → ได้ section 1, 4 ดิบๆ
2. **Audit ของเดิม** — ถ้ามี doc เก่า ให้ตรวจทุก claim กับโค้ดก่อน (โอกาสสูงที่จะเจอของผิด เช่น infra ที่ไม่มีจริง)
3. **Variation matrix** — ถ้ามีของซ้ำหลาย instance (provider/tenant/plugin) fan-out อ่านทุกตัว ทำตาราง → section 7
4. **Contracts** — ขุด request/response struct + webhook payload + queue message จริงจากโค้ด → section 3
5. **State machine** — grep ค่า status ทุกค่า หา setter ทุกจุด → section 5 (ค่าที่หา setter ไม่เจอ = landmine)
6. **Data map** — grep ทุก collection/table หาคนอ่าน-เขียนข้าม repo → section 6
7. **Workflow trace** — ไล่ 2-3 flow หลักจบ end-to-end พร้อม side effects → เจอ landmines เพิ่มเสมอ
8. **สังเคราะห์** — non-negotiables, kill list, open questions → section 9-12
9. **เขียน DNA นี้** เป็น index + สาระสำคัญ แล้ว link ไฟล์ละเอียด

เคล็ดลับ: ใช้ subagent ขนานกับงาน fan-out (ขั้น 3, 4, 6) แต่ **spot-check ผล agent กับโค้ดเองเสมอ** ก่อนบันทึก — agent อ่านผิดได้

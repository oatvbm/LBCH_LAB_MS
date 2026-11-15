# 06 — แนวทางพัฒนา LAB_MS แบบทีม AI (Claude + Codex)

> คำตอบสั้น: **ทำได้** และ LAB_MS ที่ออกแบบเป็นโมดูล (เอกสาร 05) เหมาะกับการแบ่งงานให้ AI หลายตัวทำคู่ขนานอยู่แล้ว
> แต่สิ่งที่กำหนดความเร็วจริงคือ **คนที่รีวิว ตัดสินใจ และทดสอบกับข้อมูลจริง** ไม่ใช่จำนวน agent

---

## 1. บทบาท

| บทบาท | ใคร | หน้าที่ |
|---|---|---|
| **Tech lead / Product owner** | คุณ (คน) | ตัดสินใจ, เขียน/อนุมัติ task card, รีวิวและ merge PR, คุยกับผู้ใช้ lab, UAT, deploy |
| **Implementer** | Claude Code session / Codex task (หลายตัวพร้อมกัน) | ทำงานตาม task card ใน branch ของตัวเอง เขียน test และเปิด PR |
| **Reviewer** | agent อีกตัว (ไขว้กัน) | Codex รีวิว PR ที่ Claude เขียน และ Claude (`/code-review`) รีวิว PR ที่ Codex เขียน ก่อนถึงคุณ |
| **Gatekeeper** | CI | build, unit test, integration test กับ MySQL, architecture test ต้องผ่านก่อน merge |

ทำไมต้องไขว้: โมเดลต่างกันมักเห็นจุดผิดคนละแบบ และลดงานรีวิวของคุณ

---

## 2. สิ่งที่ต้องมีก่อนเริ่มแบ่งงาน

1. **Git repo `LBCH_LAB_MS` บน GitHub (private)** — ใช้ branch/PR/CI ร่วมกันทั้ง Claude และ Codex (ตอนนี้โฟลเดอร์ `StatWarning on web` ยังไม่เป็น git repo)
2. **`AGENTS.md`** ที่ root = กติกากลางของทุก agent (Codex อ่านไฟล์นี้) และ **`CLAUDE.md`** ที่ import `@AGENTS.md` แล้วเพิ่มเฉพาะเรื่องของ Claude — กติกาจะได้มีที่เดียว
3. **R0 Platform kernel เสร็จก่อน** (ทำทีละขั้น ไม่แบ่งคู่ขนาน): โครง solution, CAS auth, ระบบสิทธิ์, layout, CI, กติกาโมดูล — ทุกเส้นทางต้องพึ่งส่วนนี้
4. **Contract ก่อนโค้ด**: SQL schema ของแต่ละโมดูล, interface ระหว่างโมดูล, HL7 parser interface, ตัวอย่าง JSON ของ API — เขียนและ merge ก่อนแยกเส้นทาง
5. **ข้อมูลทดสอบที่ปลอดภัย** (ข้อ 6)

---

## 3. การแบ่งเส้นทางงาน (lanes) สำหรับ R1

หลัง R0 เสร็จ แบ่งได้ประมาณนี้ (แต่ละเส้นทางแตะโฟลเดอร์ของตัวเองเท่านั้น):

| Lane | งาน | เครื่องมือแนะนำ | โฟลเดอร์ที่แก้ได้ |
|---|---|---|---|
| L1 | Worker ingestion: HL7 parser, OrderIngestor, ResultIngestor, CheckInSync + parity comparer | Codex | `src/Core/**`, `src/Hosts/Lbch.LabMs.Worker/**` |
| L2 | โมดูล Stat: TurnaroundCalculator, StatCaseEngine, dashboard real-time, บันทึกสาเหตุ | Claude | `src/Modules/Service/Stat/**` |
| L3 | โมดูล Critical: workflow 0→3, หน้าหอผู้ป่วย, LegacyCriticalBridge | Claude | `src/Modules/Service/Critical/**` |
| L4 | รายงาน/KPI (SQL + Dapper) + parity test กับ golden dataset | Codex | `src/Modules/*/Reports/**`, `tests/Parity/**` |
| L5 | นอก repo: QIIS API (`kpi`), ลงทะเบียน AppCode `LAB_MS` + scope ใน CAS, อัปเกรด CAS เป็น .NET 10 | Claude หรือ Codex | repo `kpi`, `LBCH_CAS` |

- เริ่มจริงไม่เกิน **3–4 lane พร้อมกัน** สำหรับผู้รีวิว 1 คน มากกว่านี้ PR จะกองรอ
- ไฟล์ส่วนกลาง (Platform kernel, schema ของ Core, `Directory.Packages.props`) แก้ผ่าน task "platform" แยกเท่านั้น เพื่อกันชนกัน
- ระยะ R2–R5 แบ่งต่อแบบเดียวกัน (เช่น ย้าย Lab_Stock lane หนึ่ง, Lab_RMS อีก lane หนึ่ง)

---

## 4. กติกาการทำงาน

1. **1 task = 1 branch = 1 worktree = 1 PR** — Claude Code session ใช้ git worktree แยกกัน, Codex cloud task สร้าง branch/PR ของตัวเอง
2. PR เล็ก (แนะนำไม่เกิน ~500 บรรทัดที่เปลี่ยน) และต้องมี test
3. agent ห้ามแก้ไฟล์นอกโฟลเดอร์ใน task card — ถ้าจำเป็นต้องแก้ ให้หยุดและบอกใน PR
4. merge ได้เมื่อ: CI ผ่าน + reviewer agent ไม่มี finding ร้ายแรงค้าง + คุณอนุมัติ
5. ทุกการตัดสินใจเชิงออกแบบที่ agent เสนอ บันทึกเป็น ADR ใน `docs/adr/` (แบบเดียวกับ Lab_Stock)
6. อัปเดต `docs/DEVELOPMENT_STATUS.md` หลัง merge ทุกครั้ง เพื่อให้ agent ตัวถัดไปรู้สถานะล่าสุด

### 4.1 Task card (ต้นแบบ)

```markdown
# TASK-R1-L2-03: TurnaroundCalculator
เป้าหมาย: คำนวณ clock_start_at และ due_at ตาม service_hour_rule
อ้างอิง: docs/01 ข้อ 2.4, docs/02 ข้อ 4.1, docs/04 ตาราง service_hour_rule
แก้ได้เฉพาะ: src/Modules/Service/Stat/Domain/**, tests/Stat.Domain.Tests/**
Contract: ITurnaroundCalculator (merge แล้วใน TASK-R1-00)
เกณฑ์ผ่าน:
- [ ] test ครบทุกกรณีใน docs/01 ข้อ 2.4 (06–08, 12–13, due ตกช่วงพัก)
- [ ] กรณีข้ามเที่ยงคืน
- [ ] C+S = 60 นาที มาจาก test_group_rule ไม่ฮาร์ดโค้ด
ไม่อยู่ในขอบเขต: UI, DB
```

---

## 5. ขั้นตอนต่อ 1 task

```text
คุณเขียน/อนุมัติ task card
   → agent วางแผน (ให้ตอบแผนสั้น ๆ ก่อนเขียนโค้ดสำหรับงานใหญ่)
   → agent เขียนโค้ด + test ใน branch ของตัวเอง → เปิด PR
   → CI
   → reviewer agent อีกตัวรีวิว → implementer แก้
   → คุณรีวิวสุดท้าย → merge → อัปเดต DEVELOPMENT_STATUS.md
```

---

## 6. ความปลอดภัยและข้อมูลผู้ป่วย (สำคัญ)

1. **ห้ามส่งข้อมูลผู้ป่วยจริงให้ AI** — ไฟล์ HL7 ตัวอย่างต้องปิดชื่อ, HN, CID, วันเกิดก่อน (ทำสคริปต์ de-identify ที่รันบนเครื่องโรงพยาบาลเอง) และตรวจนโยบายการใช้ AI บน cloud ของโรงพยาบาล
2. agent ใช้เฉพาะฐานข้อมูล development ที่มีข้อมูลสังเคราะห์ ห้ามให้ connection string ของ production
3. ไม่ใส่รหัสผ่าน/secret ในไฟล์ที่ agent อ่าน (`appsettings` ใช้ user-secrets/environment variable)
4. งานที่แตะ CAS (auth, token, scope) ต้องผ่าน security review (`/security-review`) และคุณรีวิวเองทุกบรรทัด
5. agent ไม่ deploy เอง — deploy staging/production ทำโดยคุณผ่าน script ที่ตรวจแล้ว

---

## 7. ความเร็วที่คาดได้

| ส่วนงาน | ผลของทีม AI |
|---|---|
| เขียนโค้ด, test, ย้ายโค้ดจากระบบเดิม | เร็วขึ้นมาก โดยเฉพาะเมื่อแบ่ง lane คู่ขนาน |
| รีวิวและตัดสินใจ | **เป็นคอขวด** — ขึ้นกับเวลาของคุณ |
| ยืนยันกติกากับผู้ใช้ lab, เตรียมข้อมูลทดสอบ | เท่าเดิม |
| UAT, shadow/parallel run, migrate ข้อมูลจริง | เท่าเดิม (ต้องรอข้อมูลจริงหลายวัน/สัปดาห์) |

ประมาณการคร่าว ๆ: จาก ~18 เดือน (เอกสาร 05) น่าจะเหลือ **ประมาณ 10–12 เดือน** ถ้า lane คู่ขนานทำงานได้ตามแผนและคุณมีเวลารีวิวสม่ำเสมอ — ตัวเลขนี้ต้องปรับหลังจบ R0 และ R1 จากความเร็วจริง

---

## 8. ลำดับเริ่มต้น

1. สร้าง repo `LBCH_LAB_MS` บน GitHub + ย้ายเอกสาร 01–06 เข้าไปใน `docs/`
2. เขียน `AGENTS.md` / `CLAUDE.md` (มาตรฐานจากเอกสาร 05 ข้อ 4 + กติกาข้อ 4 และ 6)
3. R0 Platform kernel แบบทีละขั้น (agent เดียว + คุณรีวิว)
4. merge contract ของ R1 (schema, interface)
5. เปิด lane L1–L4 คู่ขนาน (+ L5 ใน repo อื่น) — Claude session สร้างจากแอปนี้ได้โดยตรง แต่ละ session มี worktree แยก ส่วน Codex สั่งเป็น cloud task ต่อ lane

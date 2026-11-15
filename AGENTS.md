# AGENTS.md — กติกากลางสำหรับทุก agent (Claude, Codex) ใน LBCH_LAB_MS

ไฟล์นี้เป็น **ที่เดียว** ของกติกาที่ใช้ร่วมกันทุก agent (`CLAUDE.md` import ไฟล์นี้แล้วเพิ่มเฉพาะเรื่องของ Claude)
ถ้ากติกาในไฟล์นี้ขัดกับเอกสารใน `docs/` ให้ยึด `docs/05-platform-roadmap.md` และ `docs/06-ai-team-workflow.md` แล้วแจ้งความขัดแย้งใน PR

---

## 1. ระบบนี้คืออะไร

**LBCH Laboratory Management System (LAB_MS)** ของงานเทคนิคการแพทย์ โรงพยาบาลมะเร็งลพบุรี
แพลตฟอร์มเดียวแบบแบ่งโมดูล (modular monolith) ที่ทยอยย้ายระบบเดิมเข้ามาทีละโมดูล แบ่งเป็น 2 ด้าน + ส่วนกลาง

- **Service** (ระบบบริการ): Order & Specimen, Stat, Critical, Micro/Hemo, Outlab, HPV, Service statistics
- **Quality** (ระบบคุณภาพ): QC, Risk, Inventory, Equipment, Document control, Lab catalog, Indicators
- **Platform** (ส่วนกลาง): login CAS, ผู้ใช้, permission catalog + ชุดสิทธิ์, audit, real-time, export, layout, Worker, outbox

ระบบที่ **ไม่รวม**: BBM (ใช้ HIS), `Risk_record` ของ LabWeb, QIIS (แอปแยก รับค่าผ่าน API), CAS (บริการกลาง), Riskincident

### ชื่อที่ใช้ (ตัดสินแล้ว — ห้ามเปลี่ยน)

| เรื่อง | ค่า |
|---|---|
| ชื่อที่แสดงผู้ใช้ | LBCH Laboratory Management System |
| AppCode ใน CAS | `LAB_MS` |
| path บน IIS | `/LAB_MS` (redirect `/lab_ms` → `/LAB_MS`) |
| namespace | `Lbch.LabMs.*` |
| schema DB | `labms_<module>` เช่น `labms_platform`, `labms_stat`, `labms_critical` |

### เอกสารที่ต้องอ่านก่อนทำงาน

อ่านตามลำดับ **05 → 06 → 03 → 02 → 04 → 01** (เมื่อขัดกันให้ยึดเอกสารที่ใหม่กว่า/ลำดับก่อน — 05 สำคัญที่สุด)

| ไฟล์ | เนื้อหา |
|---|---|
| `docs/05-platform-roadmap.md` | การตัดสินใจ D1–D10, โครงสร้าง, มาตรฐาน, roadmap R0–R5 |
| `docs/06-ai-team-workflow.md` | บทบาท, lane, กติกาการทำงาน, task card, ความปลอดภัยข้อมูล |
| `docs/03-implementation-plan.md` | แผน StatWarning (= R1) |
| `docs/02-system-design.md` | ออกแบบ Stat/Critical, CAS flow |
| `docs/04-database-design.md` | schema เดิม, schema ใหม่, **data contract กับ LabWeb** |
| `docs/01-legacy-analysis.md` | business rules ของ SW2/StatImportAuto2 |
| `docs/DEVELOPMENT_STATUS.md` | สถานะล่าสุด — **อ่านทุกครั้งก่อนเริ่ม task** |
| `docs/legacy-changes.md` | ทะเบียนการแก้ระบบเดิม และความต่างจากพฤติกรรมเดิมที่ตั้งใจ |
| `docs/adr/` | ADR ทุกการตัดสินใจเชิงออกแบบ |
| `docs/tasks/` | task card |

---

## 2. มาตรฐานทางเทคนิค (เอกสาร 05 ข้อ 4)

| เรื่อง | มาตรฐาน |
|---|---|
| Runtime / UI | .NET 10 LTS, ASP.NET Core **MVC + htmx**, SignalR สำหรับ real-time (ไม่ใช้ Razor Pages, Blazor หรือ SPA) |
| Data access | EF Core 10 + **`MySql.EntityFrameworkCore` (Oracle) 10.x** สำหรับงานทั่วไป; **Dapper + SQL** สำหรับรายงาน/KPI ที่ซับซ้อน — ห้ามใช้ Pomelo |
| Provider | อ้างอิง provider ได้เฉพาะในโปรเจกต์ `*.Infrastructure` |
| Migration | **SQL migration script ต่อโมดูล** ที่ `database/<module>/NNNN_<name>.sql` บันทึกใน `schema_migrations` — **ไม่ใช้ EF migrations** |
| Database | MySQL (ปัจจุบัน 8.0.17, เป้าหมาย 8.4 LTS) — schema ใหม่ใช้ `utf8mb4` + `utf8mb4_0900_ai_ci`, `DATETIME` เต็ม (Asia/Bangkok), surrogate PK, FK จริง, `created_at/updated_at` |
| Authentication | LBCH CAS (authorization-code, server-side token store, local cookie, heartbeat, global logout) — **ห้ามสร้างหน้ากรอกรหัสผ่านของ LAB_MS เอง** |
| Authorization | CAS ตรวจแค่สิทธิ์เข้า AppCode `LAB_MS`; สิทธิ์ราย action ใช้ **permission catalog ของ LAB_MS + ชุดสิทธิ์** ที่ผู้ดูแลมอบหมาย; ตรวจฝั่ง server ทุก action (UI แค่ซ่อนปุ่ม) |
| ขอบเขตอนุมัติ | หน่วยงาน + management roles จาก CAS Profile |
| Secrets | ไม่อยู่ใน source/config แบบ plain text — ใช้ user-secrets / environment variable |
| DB account | แยกบัญชีต่อ host (web, worker) สิทธิ์เฉพาะ schema ที่ใช้ |
| Audit | ทุกการเปลี่ยนข้อมูลสำคัญเขียน `audit_log` (before/after) ใน transaction เดียวกับการเปลี่ยน |
| PDPA | CID เก็บเป็น `cid_encrypted` + `cid_last4`, mask ใน UI/export, **ห้าม log CID/token** |
| Logging | Serilog structured log |
| ทดสอบ | unit test ต่อโมดูล + integration test กับ **MySQL จริง** (Testcontainers หรือ DB ทดสอบ) + architecture test |

### โครงสร้าง solution (เอกสาร 05 ข้อ 3.2)

```text
src/Platform/   Lbch.LabMs.Platform.Kernel | .Infrastructure | .Web
src/Core/       lab core (patient, order, order test, test catalog) + HL7 contracts
src/Modules/<Service|Quality>/<Module>/   Domain | Application | Infrastructure | Web(Area)
src/Hosts/      Lbch.LabMs.Web (host เดียว) | Lbch.LabMs.Worker (Windows Service)
database/<module>/NNNN_*.sql
tests/
docs/
```

### กติกาของโมดูล (เอกสาร 05 ข้อ 3.3) — architecture test บังคับใน CI

1. แต่ละโมดูลมี schema ของตัวเอง และ **เขียนได้เฉพาะ schema ตัวเอง**
2. ข้ามโมดูลได้ผ่าน **interface สาธารณะหรือ in-process event เท่านั้น** ห้าม query ตารางของโมดูลอื่นตรง ๆ
3. Lab core อ่านได้ทุกโมดูล แต่เขียนได้เฉพาะ Worker และ API ที่ Core เปิดไว้
4. ปิด/เปิดโมดูลด้วย feature flag

### ตาราง lab core เดิม (`statwarning`) เป็นสัญญากับ LabWeb

ห้ามเปลี่ยนชื่อ/ชนิดคอลัมน์ เพิ่ม FK หรือเปลี่ยน collation ของตาราง lab core เดิม (เอกสาร 04 ข้อ 3.2 และ 5) — เพิ่ม index ได้ผ่าน DBA เท่านั้น
การเขียนตาราง lab core ต้องตรงกับ contract ในเอกสาร 04 ข้อ 5.1 ทุกคอลัมน์

---

## 3. กติกาการทำงาน (เอกสาร 06 ข้อ 4)

1. **1 task = 1 branch = 1 worktree = 1 PR** — ห้ามทำหลาย task ใน branch เดียว ห้าม commit ตรงเข้า `main`
2. **PR เล็ก** (แนะนำไม่เกิน ~500 บรรทัดที่เปลี่ยน ไม่นับไฟล์ที่ generate) และ **ต้องมี test**
3. **แก้ได้เฉพาะไฟล์/โฟลเดอร์ใน "แก้ได้เฉพาะ" ของ task card** — ถ้าจำเป็นต้องแก้นอกขอบเขต ให้หยุด แล้วระบุใน PR ว่าต้องแก้อะไรและทำไม
4. ไฟล์ส่วนกลาง (Platform kernel, schema ของ Core, `Directory.Packages.props`, `Directory.Build.props`, CI) แก้ผ่าน task "platform" เท่านั้น
5. merge ได้เมื่อ: **CI ผ่าน + reviewer agent ไม่มี finding ร้ายแรงค้าง + tech lead (คน) อนุมัติ** — agent ไม่ merge เอง
6. ทุกการตัดสินใจเชิงออกแบบที่ agent เสนอ → บันทึกเป็น ADR ใน `docs/adr/NNNN-<slug>.md`
7. หลัง merge ทุกครั้ง → อัปเดต `docs/DEVELOPMENT_STATUS.md`
8. ถ้างานแก้ระบบเดิม หรือทำให้พฤติกรรมต่างจากระบบเดิมโดยตั้งใจ (แก้ bug, เปลี่ยนนิยาม) → บันทึกใน `docs/legacy-changes.md` ใน PR เดียวกัน
9. งานใหญ่: ตอบแผนสั้น ๆ ให้ tech lead ก่อนเขียนโค้ด

### ขั้นตอนต่อ 1 task

```text
tech lead เขียน/อนุมัติ task card (docs/tasks/)
  → agent วางแผน (งานใหญ่: ตอบแผนก่อน)
  → agent เขียนโค้ด + test ใน branch ของตัวเอง → เปิด PR
  → CI → reviewer agent อีกตัว (ไขว้ Claude ↔ Codex) → implementer แก้
  → tech lead รีวิวสุดท้าย → merge → อัปเดต DEVELOPMENT_STATUS.md
```

### Task card (รูปแบบมาตรฐาน)

```markdown
# TASK-<ระยะ>-<lane>-<ลำดับ>: <ชื่อ>
เป้าหมาย: ...
อ้างอิง: docs/.. ข้อ ..
แก้ได้เฉพาะ: <glob ของไฟล์/โฟลเดอร์>
Contract: <interface/schema ที่ต้อง merge ก่อน>
เกณฑ์ผ่าน:
- [ ] ...
ไม่อยู่ในขอบเขต: ...
```

### Branch และ commit

- branch: `task/<task-id-ตัวพิมพ์เล็ก>-<slug>` เช่น `task/r0-01-solution-skeleton`
- commit message ขึ้นต้นด้วย task id เช่น `R0-01: add solution skeleton`
- PR title: `<TASK-ID>: <ชื่อ task>` และ PR body ลิงก์ task card + เกณฑ์ผ่านที่ทำครบ + สิ่งที่ยังไม่ได้ทำ

---

## 4. ความปลอดภัยและข้อมูลผู้ป่วย (เอกสาร 06 ข้อ 6) — ห้ามละเมิด

1. **ห้ามส่งหรือใส่ข้อมูลผู้ป่วยจริงลงใน repo, prompt, test หรือ fixture** — HL7 ตัวอย่างต้อง de-identify (ชื่อ, HN, CID, วันเกิด) บนเครื่องโรงพยาบาลก่อนเสมอ
2. ใช้เฉพาะฐานข้อมูล **development ที่มีข้อมูลสังเคราะห์** — **ห้ามใช้หรือขอ connection string ของ production**
3. **ห้ามใส่รหัสผ่าน/secret ในไฟล์ใด ๆ ใน repo** (`appsettings*.json` มีได้แค่ key เปล่าหรือค่า placeholder) — ใช้ `dotnet user-secrets` / environment variable
4. งานที่แตะ CAS (auth, token, scope, session, cookie) ต้องผ่าน **security review** และ tech lead รีวิวเองทุกบรรทัด
5. **agent ไม่ deploy เอง** — deploy staging/production ทำโดย tech lead ผ่าน script ที่ตรวจแล้ว
6. ห้าม log token, CID, รหัสผ่าน; query ทุกตัวต้อง parameterized
7. ห้ามคัดลอกโค้ดหรือไฟล์จากระบบเดิมที่มี credential ฝังอยู่ (เช่น `RawData` ของ StatWarning, `Web.config` ของ LabWeb) — ถ้าต้องอ้างอิง ให้สรุปเฉพาะ logic

---

## 5. กติการะหว่างเปลี่ยนผ่าน (เอกสาร 05 ข้อ 9)

1. **ระบบเดิม (LabWeb, LQMS, HPV) หยุดเพิ่มฟีเจอร์** — แก้ได้เฉพาะ bug และความปลอดภัย; ฟีเจอร์ใหม่ทำใน LAB_MS
2. เมื่อโมดูลใหม่ใช้งานได้ ให้เพิ่มเมนูในระบบเดิมที่ลิงก์มาโมดูลใหม่ (CAS SSO)
3. ตาราง lab core ใน `statwarning` เป็นสัญญากับ LabWeb จนจบ R3
4. ทุกโมดูลที่ย้ายมาต้องมี redirect จาก URL เดิม และช่วง parallel run

ทุกข้อข้างบนที่ต้องแก้ระบบเดิม → บันทึกใน `docs/legacy-changes.md`

---

## 6. คำสั่ง build / test

> จะเติมให้ครบใน R0-01 (solution skeleton) และ R0-02 (CI)

```bash
dotnet build Lbch.LabMs.sln
dotnet test Lbch.LabMs.sln
```

---

## 7. Definition of Done ของทุก task

- [ ] เกณฑ์ผ่านใน task card ครบทุกข้อ
- [ ] `dotnet build` ไม่มี warning ใหม่, `dotnet test` ผ่าน (รวม architecture test)
- [ ] ไม่มี secret/ข้อมูลผู้ป่วยใน diff
- [ ] ไม่แก้ไฟล์นอกขอบเขต task card (หรือระบุเหตุผลใน PR)
- [ ] ADR ใหม่ (ถ้ามีการตัดสินใจเชิงออกแบบ)
- [ ] `docs/legacy-changes.md` (ถ้าแตะระบบเดิมหรือเปลี่ยนพฤติกรรมจากระบบเดิม)
- [ ] PR body บอกสิ่งที่ทำ วิธีตรวจ และข้อจำกัดอย่างตรงไปตรงมา

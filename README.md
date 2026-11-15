# LBCH Laboratory Management System (LAB_MS)

ระบบบริหารงานห้องปฏิบัติการของงานเทคนิคการแพทย์ โรงพยาบาลมะเร็งลพบุรี
รวมระบบงาน lab ที่กระจายอยู่หลายโปรแกรม (StatWarning, LabWeb, LQMS, Lab_Stock, LRMS, HPV) เข้าเป็นแพลตฟอร์มเดียวแบบแบ่งโมดูล
จัดกลุ่มตาม ISO 15189:2022 เป็น **ระบบบริการ (Service)** และ **ระบบคุณภาพ (Quality)** บนส่วนกลาง **Platform**

> สถานะ: **เริ่มต้น repo — ยังไม่มีโค้ด** อยู่ระหว่างวางแผน R0 Platform kernel — ดู [docs/DEVELOPMENT_STATUS.md](docs/DEVELOPMENT_STATUS.md)

## ภาพรวม

```text
LAB_MS
├─ Service   Order & Specimen, Stat, Critical, Micro/Hemo, Outlab, HPV, Service statistics
├─ Quality   QC, Risk, Inventory, Equipment, Document control, Lab catalog, Indicators (→ QIIS)
└─ Platform  CAS login, ผู้ใช้/สิทธิ์, audit, real-time, export, layout, Worker (HL7/ABS/HIS), outbox
```

| รายการ | ค่า |
|---|---|
| Runtime | .NET 10 LTS, ASP.NET Core MVC + htmx, SignalR |
| Database | MySQL (schema `labms_*`) ผ่าน EF Core 10 + `MySql.EntityFrameworkCore` และ Dapper สำหรับรายงาน |
| Authentication | LBCH CAS, AppCode `LAB_MS` |
| Hosting | IIS `/LAB_MS` + Windows Service (Worker) |

## Roadmap

| ระยะ | ขอบเขต |
|---|---|
| **R0** | Platform kernel: solution, CAS auth, ผู้ใช้/สิทธิ์, audit, layout, CI |
| R1 | Stat + Critical (แทน StatWarning SW2 + StatImportAuto2) |
| R2 | Inventory/Equipment (Lab_Stock) + Risk (LRMS) |
| R3 | Micro / Hemoculture / Outlab (LabWeb) |
| R4 | QC (LQMS) + ส่วนที่เหลือของ LabWeb |
| R5 | HPV |

รายละเอียดใน [docs/05-platform-roadmap.md](docs/05-platform-roadmap.md)

## เอกสาร

อ่านตามลำดับ:

1. [05 — แผนรวมระบบงาน Lab](docs/05-platform-roadmap.md) — การตัดสินใจหลัก, โครงสร้าง, มาตรฐาน, roadmap
2. [06 — แนวทางพัฒนาแบบทีม AI](docs/06-ai-team-workflow.md) — บทบาท, lane, task card, ความปลอดภัยข้อมูล
3. [03 — แผนดำเนินงาน StatWarning (R1)](docs/03-implementation-plan.md)
4. [02 — ออกแบบระบบ StatWarning](docs/02-system-design.md)
5. [04 — ฐานข้อมูลเดิมและใหม่](docs/04-database-design.md)
6. [01 — วิเคราะห์ระบบ StatWarning เดิม](docs/01-legacy-analysis.md)

อื่น ๆ: [สถานะการพัฒนา](docs/DEVELOPMENT_STATUS.md) · [ทะเบียนการเปลี่ยนแปลงระบบเดิม](docs/legacy-changes.md) · `docs/adr/` · `docs/tasks/` · [schema เดิม](docs/legacy-schema/statwarning_schema.sql)

## สำหรับผู้พัฒนาและ AI agent

กติกาการทำงานทั้งหมดอยู่ใน [AGENTS.md](AGENTS.md) (Claude Code อ่านผ่าน [CLAUDE.md](CLAUDE.md))

- 1 task = 1 branch = 1 PR, merge เมื่อ CI ผ่าน + reviewer agent + tech lead อนุมัติ
- **ห้ามใส่ข้อมูลผู้ป่วยจริงหรือ secret ใน repo** — ใช้ `dotnet user-secrets` / environment variable

### เริ่มพัฒนา

> จะเติมหลัง R0-01 (solution skeleton): SDK ที่ต้องใช้, การตั้ง DB development, user-secrets ของ CAS, คำสั่ง build/test/run

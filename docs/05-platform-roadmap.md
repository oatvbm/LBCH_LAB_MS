# 05 — LBCH Laboratory Management System (LAB_MS): แผนรวมระบบงาน Lab

> สถานะ: ตัดสินใจหลักแล้ว — 2026-11-15
> ข้อมูลจากการสำรวจ `C:\programing` (ไฟล์โปรเจกต์ การตั้งค่า และเอกสารใน `Lab_Stock/docs`) ยังไม่ได้อ่านโค้ดของทุกระบบอย่างละเอียด

---

## 1. การตัดสินใจ

| # | เรื่อง | ข้อสรุป |
|---|---|---|
| D1 | ชื่อและขอบเขต | **LBCH Laboratory Management System (LAB_MS)** รองรับ 2 ด้าน: **ระบบบริการ** และ **ระบบคุณภาพ** |
| D2 | รูปแบบ | **แพลตฟอร์มเดียวแบบแบ่งโมดูล (modular monolith)** ทยอยย้ายทีละโมดูล ไม่เขียนใหม่ทั้งหมดในครั้งเดียว |
| D3 | QIIS/KPI | อนาคตใช้ทั้งโรงพยาบาล → **คงเป็นแอปแยก** LAB_MS ส่งค่าตัวชี้วัดเข้า QIIS ผ่าน API |
| D4 | CAS | ทีมดูแลเอง → เพิ่ม AppCode, API-client scope และ endpoint ได้ตามแผน |
| D5 | ระบบที่รวม | StatWarning, ค่าวิกฤต, จุลชีวะ, Hemoculture, Outlab, HPV, คลังพัสดุ (Lab_Stock), LRMS, **LQMS**, ส่วนที่เหลือของ LabWeb |
| D6 | ระบบที่ไม่รวม | **BBM** (ธนาคารเลือดย้ายไปใช้ HIS), **Risk_record** ของ LabWeb (เลิกใช้ ใช้ LRMS แทน), QIIS (แอปแยก), CAS |
| D7 | EF Core provider | **`MySql.EntityFrameworkCore` (Oracle) 10.x** — เหตุผลในข้อ 5 |
| D8 | Repository | **สร้าง repo ใหม่** แล้วยกโค้ดส่วนกลาง (auth, สิทธิ์) จาก Lab_Stock มาปรับ |
| D9 | โมดูลแรก | StatWarning + ค่าวิกฤต |
| D10 | ฐานข้อมูล | **MySQL ต่อ** (server เดิม 172.16.5.122) และวางแผนอัปเกรดจาก 8.0.17 เป็น **MySQL 8.4 LTS** ร่วมกับ DBA — ตัดสินใจ 2026-11-15 |

### 1.0 เหตุผลที่ไม่เปลี่ยนเป็น SQL Server (D10)

พิจารณาแล้วว่า SQL Server มีข้อดีเชิงเทคนิค (ตัวเชื่อม EF Core ของ Microsoft, schema หลายตัวในฐานเดียว, temporal table, การเข้ารหัสในตัว) แต่เลือก MySQL ต่อ เพราะ

- ไม่มี SQL Server Standard ที่มี license และคนดูแลสำหรับระบบนี้ ส่วนรุ่น Express จำกัดขนาด DB 10 GB ไม่พอสำหรับ LAB_MS ระยะยาว
- LabWeb ยังใช้ตารางกลางใน MySQL จนจบ R3 ถ้าเปลี่ยน Worker ต้องเขียนสองฐานข้อมูลตลอดช่วงเปลี่ยนผ่าน
- Lab_Stock, Lab_RMS และ QIIS ใช้ MySQL อยู่แล้ว และทีมคุ้นกับ MySQL/Navicat

สิ่งที่ต้องชดเชยข้อด้อยของ MySQL:
- audit/ประวัติการแก้ไข → ใช้ตาราง `audit_log` ของ platform (เก็บ before/after) แทน temporal table
- PDPA → เข้ารหัส CID ที่ระดับแอป (เก็บ `cid_encrypted` + `cid_last4`) และจำกัดสิทธิ์ DB account ตาม schema
- ตัวเชื่อม EF Core → ตามข้อ 5 (Oracle provider + Dapper สำหรับรายงาน + integration test)

### 1.1 ชื่อที่ใช้ (ตัดสินแล้ว)

- ชื่อเต็มที่แสดงผู้ใช้: **LBCH Laboratory Management System**
- ชื่อย่อ / AppCode ใน CAS: **`LAB_MS`** (เลี่ยงการสับสนกับ Learning Management System)
- path บน IIS: `/LAB_MS` (ตัวพิมพ์ใหญ่ทั้งหมด และ redirect `/lab_ms` → `/LAB_MS` ตามบทเรียน QIIS)
- repo: `LBCH_LAB_MS`, namespace: `Lbch.LabMs.*`, schema prefix: `labms_`

### 1.2 ความสัมพันธ์กับ Riskincident (ระบบของโรงพยาบาล)

`Riskincident` (RM_LBCH) เป็นระบบของโรงพยาบาล นำเข้าข้อมูลจาก **HRMS ของ สรพ.** เพื่อทำ dashboard ให้ทีม RM — **ไม่รวมใน LAB_MS**

- โมดูล Risk (LRMS) ดูแลความเสี่ยงและอุบัติการณ์ระดับ lab
- อุบัติการณ์ที่ต้องรายงานระดับโรงพยาบาลยังรายงานเข้า HRMS ตามช่องทางของ RM; ควรให้ LRMS เก็บ "เลขอ้างอิง HRMS" ของเรื่องนั้นไว้ เพื่อไม่ต้องบันทึกซ้ำและตามเรื่องข้ามระบบได้ (ต้องคุยกับทีม RM)

---

## 2. ระบบที่มีอยู่และปลายทาง

| ระบบ | โปรเจกต์ | Stack ปัจจุบัน | AppCode | ปลายทางใน LAB_MS |
|---|---|---|---|---|
| StatWarning + ค่าวิกฤต | `StatWarning on web` | VB WinForms | — | **Service › Stat, Critical** (R1) |
| คลังพัสดุ lab | `Lab_Stock` | .NET 10 MVC, `MySql.EntityFrameworkCore` 10.0.12 | `LABSTOCK` | **Quality › Inventory & Equipment** (R2) |
| LRMS | `Lab_RMS` | .NET 10 MVC, **Pomelo 8.0.3** (EF Core 8) | `LRMS` | **Quality › Risk** (R2) — ต้องเปลี่ยน provider |
| จุลชีวะ / Hemoculture / Outlab | `labweb5` | WebForms 4.7.2 | `LABWEB` | **Service › Micro, Hemo, Outlab** (R3) |
| LQMS (IQC/EQA) | `LQMS`, `LQMSv1` | WebForms 4.7.2 (DB `labdatabase`) | — | **Quality › QC** (R4) |
| LabWeb ส่วนอื่น | `labweb5` | WebForms 4.7.2 | `LABWEB` | Catalog, Document control, Reagent (R4); `Risk_record` เลิกใช้; `KPI_view` เลิกใช้ (ไปดูใน QIIS) |
| HPV | `HPVlbch`, `HPVloader`, `HPVapi` | WebForms 4.7.2 + REST + LINE/KTB | — | **Service › HPV** (R5) |
| QIIS/KPI | `kpi` | .NET 8 MVC, Pomelo 8 | `QIIS` | แอปแยก + API รับค่าตัวชี้วัด |
| CAS | `LBCH_CAS` | .NET 8 | — | บริการกลาง |
| BBM | `BBM*` | — | — | ไม่รวม (ใช้ HIS) |

> หยุดงานใน `labweb6/LbchMvc` แล้วรวมเข้ากับแผนนี้ เพื่อไม่ให้มีงานซ้ำสองทาง

---

## 3. โครงสร้างระบบ: 2 ด้าน + ส่วนกลาง

จัดกลุ่มให้สอดคล้องกับ ISO 15189:2022 (ข้อกำหนดด้านกระบวนการ และด้านระบบบริหารคุณภาพ) เพื่อให้ใช้เป็นหลักฐานตอนตรวจประเมินได้

```text
LBCH Laboratory Management System (LAB_MS)
│
├─ Service (ระบบบริการ: ก่อน–ระหว่าง–หลังการตรวจวิเคราะห์)
│   ├─ Order & Specimen   ติดตามใบสั่งตรวจและเวลาแต่ละจุด (lab core)
│   ├─ Stat               ประกันเวลา, แจ้งเตือน, สาเหตุล่าช้า          ← StatWarning
│   ├─ Critical           แจ้งค่าวิกฤต T0–T5                           ← StatWarning
│   ├─ Micro / Hemo       บันทึกผลจุลชีวะ, ติดตาม hemoculture          ← LabWeb
│   ├─ Outlab             ส่งตรวจภายนอกและติดตามผล                     ← LabWeb
│   ├─ HPV                                                             ← HPVlbch
│   └─ Service statistics ห้องเจาะเลือด, routine, จุลชีวะ               ← SW2
│
├─ Quality (ระบบคุณภาพ)
│   ├─ QC                 IQC, EQA, Sigma                              ← LQMS
│   ├─ Risk               ความเสี่ยง/อุบัติการณ์                          ← LRMS
│   ├─ Inventory          แผน, จัดซื้อ, คลัง, lot/หมดอายุ               ← Lab_Stock
│   ├─ Equipment          เครื่องมือ, สอบเทียบ, บำรุงรักษา                ← Lab_Stock
│   ├─ Document control   เอกสารคุณภาพ                                  ← LabWeb
│   ├─ Lab catalog        รายการตรวจ/ราคา                              ← LabWeb
│   └─ Indicators         คำนวณ KPI/QI แล้วส่งเข้า QIIS
│
└─ Platform (ส่วนกลาง)
    login CAS, ผู้ใช้, permission catalog + ชุดสิทธิ์, audit, แจ้งเตือน real-time,
    export Excel, หน้าตาเดียวกัน, Worker (HL7/ABS/HIS), outbox
```

### 3.1 สถาปัตยกรรม

```text
                    LBCH CAS (authentication, app access, org/management profile, API clients)
                           ▲                    ▲
          SSO / heartbeat  │                    │ API-client token (scope qiis.indicator.submit)
                           │                    │
Browser ──HTTPS──▶  LAB_MS Web  (ASP.NET Core .NET 10, IIS /LAB_MS) ─────API────▶  QIIS (แอปแยก)
                    Platform kernel + Modules (Areas)
                                   │
LIS/HIS HL7, ABS, HIS ──▶  LAB_MS Worker (Windows Service): ingestion, schedulers, outbox
                                   │
                    MySQL: labms_platform | labms_core | labms_stat | labms_critical | labms_stock | labms_risk | labms_qc ...
                           + `statwarning`, `labdatabase` เดิม (ใช้ร่วมกับ LabWeb/LQMS ระหว่างเปลี่ยนผ่าน)
```

### 3.2 โครงสร้าง solution (repo ใหม่ `LBCH_LAB_MS`)

```text
Lbch.LabMs.sln
├─ src/Platform/
│   ├─ Lbch.LabMs.Platform.Kernel          ICurrentUser, IPermissionService, IAuditLog, IClock, events
│   ├─ Lbch.LabMs.Platform.Infrastructure  CAS client, user/permission store, audit, outbox, export, DbContext base
│   └─ Lbch.LabMs.Platform.Web             auth middleware, layout/design system, SignalR hub
├─ src/Core/                              lab core (patient, order, order test, test catalog) + HL7 contracts
├─ src/Modules/
│   ├─ Service/Stat/      Domain | Application | Infrastructure | Web(Area)
│   ├─ Service/Critical/
│   ├─ Quality/Inventory/   (ย้ายจาก Lab_Stock)
│   └─ Quality/Risk/        (ย้ายจาก Lab_RMS)  ...
├─ src/Hosts/
│   ├─ Lbch.LabMs.Web       host เดียวรวมทุกโมดูล
│   └─ Lbch.LabMs.Worker    Windows Service
├─ database/<module>/NNNN_*.sql          SQL migration ต่อโมดูล
├─ tests/                                ต่อโมดูล + architecture tests
└─ docs/  (ADR, ย้ายเอกสาร 01–05 นี้เข้าไป)
```

### 3.3 กติกาของโมดูล

1. แต่ละโมดูลมี **schema ของตัวเอง** และเขียนได้เฉพาะ schema ตัวเอง
2. ข้ามโมดูลได้ผ่าน interface สาธารณะหรือ in-process event เท่านั้น ห้าม query ตารางของโมดูลอื่นตรง ๆ
3. Lab core อ่านได้ทุกโมดูล แต่เขียนได้เฉพาะ Worker และ API ที่ Core เปิดไว้
4. architecture test (เช่น NetArchTest) ตรวจกติกาข้อ 1–3 ใน CI
5. ปิด/เปิดโมดูลด้วย feature flag เพื่อ deploy ทีละส่วน

---

## 4. มาตรฐานของแพลตฟอร์ม

| เรื่อง | มาตรฐาน | ที่มา |
|---|---|---|
| Runtime / UI | .NET 10 LTS, ASP.NET Core MVC + htmx, SignalR สำหรับ real-time | Lab_Stock ADR-001 |
| Data access | EF Core 10 + `MySql.EntityFrameworkCore` 10.x สำหรับงานเขียน/อ่านทั่วไป; Dapper + SQL สำหรับรายงาน/KPI ที่ซับซ้อน | ข้อ 5 |
| Migration | SQL migration script ต่อโมดูล บันทึกใน `schema_migrations` | Lab_Stock ADR-019 |
| Authentication | CAS (SSO, heartbeat, global logout) | Lab_Stock ADR-009 |
| Authorization | CAS ตรวจสิทธิ์เข้า AppCode; สิทธิ์ราย action ใช้ permission catalog ของ LAB_MS + ชุดสิทธิ์ที่ผู้ดูแลมอบหมาย | Lab_Stock ADR-013/021/022 |
| ขอบเขตอนุมัติ | หน่วยงาน + management roles จาก CAS Profile | CAS protocol ข้อ 9–10 |
| Secrets | ไม่อยู่ใน source code / config แบบ plain text | Lab_Stock ADR-010 |
| DB account | แยกบัญชีต่อ host (web, worker) สิทธิ์เฉพาะ schema ที่ใช้ | Lab_Stock ADR-019 |
| ทดสอบ | unit test ต่อโมดูล + integration test กับ MySQL จริง (Testcontainers หรือ DB ทดสอบ) | ข้อ 5 |

### 4.1 ผลต่อการออกแบบ StatWarning (เอกสาร 02–04)

- **Permission**: `stat.view`, `stat.cause.record`, `critical.notify`, `critical.edit`, `order.delete`, `report.view`, `setup.manage` อยู่ใน permission catalog ของ LAB_MS (ไม่ใช่ใน CAS)
- **Schema**: `statwarning_app` → `labms_stat` (stat_case, stat_case_cause, delay_cause, service_hour_rule), `labms_critical` (critical_result, ward_recipient), `labms_platform` (app_user, audit_log, import_file_log, domain_event)
- **AppCode / path**: `LAB_MS`, `/LAB_MS`

---

## 5. คำแนะนำเรื่อง EF Core provider (D7)

| ตัวเลือก | สถานะ (พ.ย. 2026) | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **`MySql.EntityFrameworkCore` (Oracle)** | มีรุ่น 10.x รองรับ EF Core 10/.NET 10 และ MySQL 8.0+; Lab_Stock ใช้อยู่แล้ว (10.0.12) | ผู้ผลิต MySQL ดูแลเอง ออกรุ่นตาม EF Core ทันเวลา | ฟีเจอร์และการแปล LINQ บางอย่างน้อยกว่า Pomelo; driver `MySql.Data` |
| `Pomelo.EntityFrameworkCore.MySql` | **รุ่นล่าสุดคือ 9.0.0 ยังไม่มีรุ่นสำหรับ EF Core 10** (tracking issue ยังเปิดอยู่) | ชุมชนใหญ่ ฟีเจอร์ครบ ใช้ MySqlConnector | ติดอยู่ที่ EF Core 9 — ใช้กับ .NET 10 + EF Core 10 ไม่ได้ |
| `Microting.EntityFrameworkCore.MySql` | fork ของ Pomelo ที่มีรุ่น 10.x | API เหมือน Pomelo | ผู้ดูแลกลุ่มเล็ก ความเสี่ยงระยะยาวสำหรับระบบโรงพยาบาล |

**คำแนะนำ: ใช้ `MySql.EntityFrameworkCore` (Oracle) 10.x** เพราะ

1. เป็นตัวเดียวที่เป็นทางการและรองรับ .NET 10 / EF Core 10 แล้ว ซึ่งตรงกับ .NET 10 LTS ที่จะใช้ไปจนถึงปลายปี 2028
2. Lab_Stock ใช้และตรวจ compatibility แล้ว (ADR-018) ทีมจึงมีประสบการณ์อยู่แล้ว
3. Lab_RMS ที่ใช้ Pomelo **8.0.3** ตอนนี้ใช้ EF Core 8 อยู่บน .NET 10 ซึ่งต้องย้ายอยู่แล้วเมื่อเข้า LAB_MS

วิธีลดจุดอ่อนของ Oracle provider:
- ให้ provider ถูกอ้างอิงเฉพาะใน `Infrastructure` (เปลี่ยนภายหลังได้ถ้า Pomelo ออกรุ่น EF Core 10)
- รายงานและ KPI ที่ query ซับซ้อนให้เขียน SQL ตรงผ่าน Dapper แทนการพึ่ง LINQ
- integration test ทุก repository กับ MySQL จริง เพื่อจับ query ที่แปลไม่ได้ตั้งแต่ CI

### 5.1 เรื่องเวอร์ชันที่ควรรู้

- **.NET 8 หมดอายุการ support ไปแล้วเมื่อ 10 พ.ย. 2026** → CAS (`LBCH_CAS`), QIIS (`kpi`), iMRA และ `labweb6/LbchMvc` ยังเป็น .NET 8 ควรวางแผนอัปเกรดเป็น .NET 10 (CAS สำคัญที่สุดเพราะทุกระบบพึ่ง)
- **MySQL 8.0** หมด support แล้วเช่นกัน (server ปัจจุบัน 8.0.17) → วางแผนอัปเกรดเป็น MySQL 8.4 LTS ร่วมกับ DBA; `MySql.EntityFrameworkCore` 10.x รองรับทั้ง 8.0 และ 8.4

---

## 6. Repository ใหม่ (D8)

สร้างใหม่ดีกว่า เพราะ
- ชื่อและโครงสร้างสะท้อนระบบใหญ่ (`LBCH_LAB_MS`, `Lbch.LabMs.*`) ไม่ติดชื่อและประวัติเฉพาะของคลังพัสดุ
- วางโครง Service / Quality / Platform และกติกาโมดูลได้ตั้งแต่แรก

แต่ **ไม่เริ่มจากศูนย์**:
- ยกโค้ด CAS auth, user/permission catalog, ชุดสิทธิ์, layout และ DESIGN.md จาก Lab_Stock มาปรับเป็น Platform kernel ใน R0
- Lab_Stock และ Lab_RMS พัฒนาต่อใน repo เดิมได้จนถึง R2 แล้วค่อยย้ายเข้า (ใช้ `git subtree` ถ้าต้องการเก็บประวัติ commit)
- ย้ายเอกสาร 01–05 ชุดนี้ไปไว้ใน `docs/` ของ repo ใหม่

---

## 7. การเชื่อม QIIS

QIIS ตอนนี้ให้ผู้รับผิดชอบกรอกตัวชี้วัดเอง (Entries, Subdatasets, Workflow, Periods) ยังไม่มี API รับข้อมูล

1. เพิ่ม API ใน QIIS: `POST /api/indicators/{code}/periods/{yyyy-MM}/values` (CAS API-client token + scope `qiis.indicator.submit`) รับตัวตั้ง ตัวหาร และลิงก์รายละเอียด แล้วเข้า workflow ตรวจสอบของ QIIS ตามปกติ
2. ออกแบบ API ให้ทั่วไป (ตัวชี้วัด + period + ค่า) เพราะ QIIS จะใช้ทั้งโรงพยาบาล
3. LAB_MS คำนวณและส่งทุกต้นเดือน พร้อมเก็บหลักฐานการคำนวณ

| ตัวชี้วัด (ตามไฟล์ใน `labweb5/15. นิยาม ปี 69`) | โมดูล LAB_MS |
|---|---|
| KPI 8, QI 23–25 (ทันเวลา) | Service › Stat |
| KPI 9, QI 28–31 (ค่าวิกฤต) | Service › Critical |
| KPI 1–4, QI 1–15 (IQC/EQA/Sigma) | Quality › QC (จาก LQMS) |
| KPI 7, QI 26–27 (เครื่องพร้อมใช้, สอบเทียบ, บำรุงรักษา) | Quality › Equipment |
| KPI 5, KPI 6, QI 16–22 (รายงานผิด, ปฏิเสธ, คัดกรองผิด, หลอดผิดคน ฯลฯ) | Quality › Risk |
| KPI 11, QI 37–39 (ธนาคารเลือด) | ไม่อยู่ใน LAB_MS (ใช้ HIS) — กรอกใน QIIS หรือดึงจาก HIS ภายหลัง |
| KPI 10, 12, 13, QI 32–36 (พึงพอใจ, ร้องเรียน, บาดเจ็บ, ติดเชื้อ) | กรอกใน QIIS ตามเดิม |

> การจับคู่นี้อิงจากชื่อไฟล์ ยังไม่ได้อ่านนิยามในไฟล์ .docx — ต้องตรวจสูตรก่อนทำแต่ละตัว

---

## 8. Roadmap

```text
เดือน         1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16   17   18
R0 Kernel    ███
R1 Stat+Crit      ████ ████ ████ ███
QIIS API               ██
R2 Stock+Risk                       ████ ███
R3 Micro/Hemo/Outlab                         ████ ████ ███
R4 QC(LQMS)+LabWeb อื่น                                   ████ ████ ████
R5 HPV                                                                   ████ ████ ███
```

| ระยะ | ขอบเขต | เกณฑ์ผ่าน |
|---|---|---|
| **R0 Platform kernel** (~1 เดือน) | repo `LBCH_LAB_MS`, solution skeleton, CAS auth + heartbeat + global logout, users/permission catalog/ชุดสิทธิ์ (จาก Lab_Stock), audit, layout, CI, AppCode `LAB_MS`, ADR ชุดแรก | login ผ่าน CAS, มอบและตรวจสิทธิ์ได้, deploy staging `/LAB_MS` |
| **R1 Stat + Critical** (~4 เดือน) | เอกสาร 03 Phase 2–7: Worker ingestion (แทน StatImportAuto2), lab core, LegacyCriticalBridge | ปิด SW2 + StatImportAuto2, LabWeb ทำงานปกติ |
| **QIIS API** (คู่ขนาน R1) | endpoint รับค่าตัวชี้วัด + scope ใน CAS | ส่ง KPI 9 / QI 28–31 เดือนล่าสุดเข้า QIIS ได้ |
| **R2 Inventory/Equipment + Risk** (~2 เดือน) | ย้าย Lab_Stock และ Lab_RMS เป็นโมดูล; Lab_RMS เปลี่ยน provider; ย้ายผู้ใช้จาก `LABSTOCK`/`LRMS` ไป `LAB_MS` | URL เดิม redirect มาโมดูลใหม่, ข้อมูลครบ |
| **R3 Micro / Hemo / Outlab** (~3 เดือน) | เขียนใหม่จาก LabWeb, ค่าวิกฤต Hemoculture ส่งตรงเข้า Critical (เลิก bridge) | ปิดหน้า LabWeb ที่เกี่ยวข้อง |
| **R4 QC + LabWeb ที่เหลือ** (~3 เดือน) | QC จาก LQMS; Catalog, Document control, Reagent จาก LabWeb | ปิด LabWeb และ LQMS |
| **R5 HPV** (~3 เดือน) | ย้าย HPV + เชื่อม KTB/LINE/LIS API | ปิด HPVlbch/HPVloader |

> ประมาณการสำหรับนักพัฒนา 1–2 คนแบบไม่มี AI ช่วย ถ้าทำงานแบบทีม AI หลายเส้นทางคู่ขนานตาม [06-ai-team-workflow.md](06-ai-team-workflow.md) งานเขียนโค้ดจะเร็วขึ้นมาก แต่งาน UAT, parallel run และตรวจข้อมูลจริงยังใช้เวลาเท่าเดิม
> งานโครงสร้างพื้นฐานที่ทำคู่ขนาน: อัปเกรด CAS และ QIIS เป็น .NET 10, อัปเกรด MySQL เป็น 8.4 LTS

---

## 9. กติการะหว่างเปลี่ยนผ่าน

1. **หยุดเพิ่มฟีเจอร์ใหม่ในระบบเดิม** (LabWeb, LQMS, HPV) — แก้เฉพาะ bug และความปลอดภัย ฟีเจอร์ใหม่ทำใน LAB_MS
2. เพิ่มเมนูในระบบเดิมที่ลิงก์ไปโมดูลใหม่ทันทีที่ใช้งานได้ (CAS SSO ไม่ต้อง login ซ้ำ)
3. ตาราง lab core ใน `statwarning` ยังเป็นสัญญากับ LabWeb จนจบ R3 — เอกสาร 04 ข้อ 5
4. ทุกโมดูลที่ย้ายมาต้องมี redirect จาก URL เดิม และช่วง parallel run

---

## 10. ความเสี่ยง

| ความเสี่ยง | วิธีรับมือ |
|---|---|
| ระบบเดียวล่มกระทบทุกงาน | Worker แยก process, health check ต่อโมดูล, feature flag, deploy แบบ app_offline สั้น |
| โมดูลผูกกันจนแก้ยาก | กติกาข้อ 3.3 + architecture test ใน CI |
| ทีมเล็ก งานระบบเดิมแย่งเวลา | freeze ฟีเจอร์ระบบเดิม (ข้อ 9.1), ทำทีละระยะ |
| Lab_Stock / Lab_RMS กำลังพัฒนาอยู่ | ระหว่างนี้ใช้มาตรฐานข้อ 4 อยู่แล้ว; ย้ายใน R2 ช่วงที่ไม่มี release ใหญ่ |
| Oracle provider แปล query บางแบบไม่ได้ | Dapper สำหรับรายงาน + integration test (ข้อ 5) |
| CAS ยังเป็น .NET 8 ที่หมด support | อัปเกรด CAS เป็น .NET 10 ก่อนหรือระหว่าง R0 |

---

## 11. คำถามที่ยังเหลือ

1. ~~AppCode~~ → `LAB_MS`
2. ~~Riskincident~~ → ระบบของโรงพยาบาล (HRMS สรพ.) ไม่รวม — ต้องคุยกับทีม RM เรื่องเลขอ้างอิง HRMS ใน LRMS (ข้อ 1.2)
3. ~~ผู้พัฒนา~~ → ใช้ Claude + Codex แบบทีม — ดู [06-ai-team-workflow.md](06-ai-team-workflow.md)

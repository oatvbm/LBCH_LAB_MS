# 02 — ออกแบบระบบ StatWarning Web

> อ่านคู่กับ [01-legacy-analysis.md](01-legacy-analysis.md) (business rules เดิม) และ [03-implementation-plan.md](03-implementation-plan.md)
>
> **อัปเดต 2026-11-15:** StatWarning จะเป็น **โมดูลแรกของ LBCH Laboratory Management System (LAB_MS)** ตาม [05-platform-roadmap.md](05-platform-roadmap.md) เมื่อเอกสารนี้ขัดกับเอกสาร 05 ให้ยึดเอกสาร 05 โดยเฉพาะ
> - โครงสร้าง solution (ข้อ 2.2) → ใช้โครงสร้าง LAB_MS ในเอกสาร 05 ข้อ 3.2
> - Permission (ข้อ 5.2) → ใช้ permission catalog ของ LAB_MS (CAS ตรวจแค่สิทธิ์เข้า AppCode) ตามเอกสาร 05 ข้อ 4.1
> - AppCode `STATWARN` → `LAB_MS`, path `/LAB_MS`
> - UI ใช้ ASP.NET Core MVC (+ htmx) ให้ตรงกับ Lab_Stock/Lab_RMS แทน Razor Pages
> - EF provider `MySql.EntityFrameworkCore` และ SQL migration script ตาม Lab_Stock ADR-018/019

---

## 1. เป้าหมาย

1. **ทำได้ครบเท่าระบบเดิม**: ติดตาม Stat, บันทึกสาเหตุล่าช้า, workflow ค่าวิกฤต, KPI/สถิติ, ตั้งค่า โดย KPI ต้องให้ผลตรงกับของเดิม (ยกเว้นจุดที่แก้ bug และมีบันทึกไว้)
2. **ใช้งานผ่าน browser** ไม่ต้องติดตั้งโปรแกรม เปิดได้ทั้งเครื่องใน lab, จอแสดงผลบนผนัง และหอผู้ป่วย
3. **Login ผ่าน LBCH CAS** (SSO) และใช้สิทธิ์จาก CAS ตาม protocol กลาง
4. **Real-time push** แทนการให้ทุกเครื่อง poll ฐานข้อมูล
5. **ปลอดภัยและตรวจสอบได้**: ไม่มี credential ในโค้ด, query แบบ parameterized ทั้งหมด, มี audit trail, จัดการ CID ตาม PDPA
6. **เก็บวันเวลาครบทั้งวันที่และเวลา** เพื่อแก้ปัญหาข้ามเที่ยงคืน และย้ายเงื่อนไขเวลาที่ฮาร์ดโค้ดไปเป็น config

### นอกขอบเขตรอบแรก (ทำภายหลังได้)

- แจ้งเตือนผ่านช่องทางภายนอก (LINE Messaging API / MOPH Notify) — **LINE Notify ปิดบริการไปแล้ว ห้ามใช้**
- เชื่อม HL7 แบบ MLLP/TCP แทน shared folder
- แอปมือถือ

---

## 2. สถาปัตยกรรม

```text
                         ┌───────────────────────── LBCH CAS ─────────────────────────┐
                         │  Admin/Login (HTTPS)            API (HTTP ภายใน :57482)     │
                         └──────▲─────────────────────────────────▲────────────────────┘
                     redirect   │                                 │ exchange-code / me / heartbeat / logout
                                │                                 │ (server-to-server)
 Browser (lab, ward, TV) ──HTTPS──▶  StatWarning.Web  (ASP.NET Core, IIS /StatWarning)
        ▲                          ├─ Razor Pages + htmx (หน้าจอ)
        │  SignalR (WebSocket)     ├─ SignalR Hub  /hubs/stat
        └──────────────────────────┤─ DashboardBroadcaster (BackgroundService: อ่าน outbox → push)
                                   ├─ Reports + Excel export (ClosedXML)
                                   └─ EF Core ─────────┐
                                                       ▼
                     MySQL: `statwarning_app` (ของใหม่) + `statwarning` (lab core เดิม, ใช้ร่วมกับระบบจุลชีวะ/Outlab)
                                                       ▲
                                   ┌─ EF Core ─────────┘
 HIS/LIS shared folder (HL7) ──▶  StatWarning.Worker  (.NET Worker Service = Windows Service)
 ABS  (MySQL, read-only)      ──▶  ├─ OrderIngestor      (HL7 ORM → lab_order/test)
 Softcon HIS (SQL, read-only) ──▶  ├─ ResultIngestor     (HL7 ORU → result, critical detect)
                                   ├─ CheckInSync        (ABS Tat_abs → specimen times)
                                   ├─ StatCaseEngine     (สร้าง/ปิด stat case, คำนวณ due)
                                   ├─ NightlyJobs        (เติม check-in ที่ขาด 23:30)
 LabWeb (criticalresult เดิม) ──▶  ├─ LegacyCriticalBridge (ค่าวิกฤต Hemoculture ที่ LabWeb insert → critical_result)
                                   └─ Outbox writer      (domain_event)
```

### 2.1 เหตุผลที่เลือกเทคโนโลยีนี้

| การตัดสินใจ | ตัวเลือก | เหตุผล |
|---|---|---|
| Runtime | **.NET 10 (LTS)**, C# | ทีมคุ้นกับ .NET อยู่แล้ว, ตัวอย่างใน CAS protocol ทั้งหมดเป็น ASP.NET Core, ระบบลูกอื่น (KPI/QIIS) ก็ใช้ IIS + ASP.NET Core |
| UI | **Razor Pages + htmx + SignalR** | render ฝั่ง server ไม่ต้องมี SPA build chain, ทำ real-time ได้ด้วย SignalR, ทนกว่า Blazor Server สำหรับจอที่เปิดค้าง 24 ชม. (ไม่มีปัญหา circuit หลุดหลัง app pool recycle) |
| DB | **MySQL** (server เดิม) + EF Core (Pomelo provider): ตาราง lab core ใน `statwarning` **คงเดิม** + schema ใหม่ `statwarning_app` สำหรับข้อมูลของ StatWarning | DB เดิมใช้ร่วมกับระบบจุลชีวะ/Outlab จึงเปลี่ยนตารางกลางไม่ได้ (ดู [04](04-database-design.md)); schema ใหม่อยู่ server เดียวกันจึง join ข้าม schema ได้ |
| Background | **Worker Service** แยก process (Windows Service) | ไม่ต้องมีคน login ค้าง, restart อัตโนมัติ, deploy แยกจากเว็บ |
| Real-time | Worker เขียน `domain_event` (outbox) → เว็บอ่านทุก 3–5 วินาที → push ผ่าน SignalR | query เดียวต่อ server แทนการ query จากทุก client, worker กับ web ไม่ต้องรู้จักกัน |
| Excel | ClosedXML (server-side) | ไม่ต้องติดตั้ง Office |
| Charts | Chart.js | เบา และใช้กับ server render ได้ |
| Logging | Serilog → ไฟล์ + (ถ้ามี) Seq | ห้าม log token/CID |

> **ทำไมไม่ทำเป็นเมนูใน LabWeb?** LabWeb (WebForms, .NET Framework 4.7.2) เชื่อม CAS แล้วและใช้ DB เดียวกัน การรวมเข้าไปจะได้ login และเมนูเดียว แต่ WebForms เป็นเทคโนโลยีที่ไม่มีการพัฒนาต่อ ทำ real-time และ test ได้ยาก และจะทำให้ LabWeb ใหญ่ขึ้นอีก (worker ก็ยังต้องแยกอยู่ดี) จึงแนะนำแยกเป็นแอป ASP.NET Core โดย
> - ใช้ CAS SSO เดียวกัน → ผู้ใช้กดลิงก์จากเมนู LabWeb แล้วเข้าได้ทันทีโดยไม่ต้อง login ซ้ำ
> - ใช้ `LABWEB/CasAuth.vb` เป็นตัวอย่างอ้างอิง flow CAS ที่ใช้งานจริงแล้ว
> - map ผู้ใช้แบบเดียวกับ LabWeb (CAS → `labdatabase.ID_user.user_code`) — ดูเอกสาร 04 ข้อ 5.3
>
> ทางเลือกอื่น: ถ้าทีมอยากได้ SPA จริงจัง ให้ใช้ React + ASP.NET Core Web API แทน แต่จะเพิ่มงาน build/deploy และงานฝั่ง auth (BFF) — ไม่แนะนำสำหรับรอบแรก

### 2.2 โครงสร้าง Solution

```text
StatWarning.sln
├─ src/
│  ├─ StatWarning.Domain/          entities, enums, state machines, TurnaroundCalculator, KPI rules (ไม่มี dependency)
│  ├─ StatWarning.Application/     use cases (commands/queries), interfaces, DTO
│  ├─ StatWarning.Infrastructure/  EF Core DbContext, migrations, Hl7Parser, AbsClient, HisClient, CasClient, Outbox
│  ├─ StatWarning.Web/             Razor Pages, SignalR hub, CAS auth, reports, export
│  └─ StatWarning.Worker/          hosted services (ingestors, engine, schedulers)
├─ tests/
│  ├─ StatWarning.Domain.Tests/          TAT rules, state machines, KPI calc
│  ├─ StatWarning.Infrastructure.Tests/  HL7 parser จากไฟล์ตัวอย่างจริง, repository (Testcontainers MySQL)
│  └─ StatWarning.Parity.Tests/          เทียบผลกับระบบเดิม (golden dataset)
└─ docs/
```

---

## 3. Domain model และ Schema ใหม่

> **อัปเดตหลังได้ schema จริง:** ตาราง lab core (`patient`, `requestdata`, `requesttest`, `clinic`, `doctor`, `sample`, `testlist`, `testresultlist`, `testgroupname`, `outlab` ฯลฯ) **ต้องคงโครงสร้างเดิม** เพราะระบบอื่นใช้อยู่ worker ใหม่จะเขียนตารางเหล่านี้ให้เหมือน StatImportAuto2 ส่วนตารางใหม่ด้านล่างที่เป็นของ StatWarning เอง (stat_case, critical_result, delay_cause, ward_recipient, app_user, audit, outbox) จะอยู่ใน schema `statwarning_app`
> **DDL ฉบับร่างและ mapping อยู่ที่ [04-database-design.md](04-database-design.md)** (ตารางในข้อ 3.1 นี้คือภาพรวมแนวคิด — ถ้าต่างกัน ให้ยึดเอกสาร 04)

หลักการ: ใช้ **DATETIME เต็ม** (Asia/Bangkok ไม่มี DST) ทุกจุดที่เป็นเวลา, ใช้ enum เป็นตัวเลข/รหัส แทนข้อความไทย, มี `created_at/updated_at/updated_by` ทุกตาราง และใช้ FK จริง

### 3.1 ตารางหลัก

| ตารางใหม่ | มาจากตารางเดิม | คอลัมน์สำคัญ |
|---|---|---|
| `patient` | Patient | hn (PK), prefix, first_name, last_name, sex, birth_date, cid_encrypted, cid_last4 |
| `clinic` | Clinic | clinic_id, name, type (OPD/IPD), is_active |
| `doctor` | Doctor | doctor_id, name, is_active |
| `patient_status` | PatientStatus | id, name, **is_fast_track**, **skip_abs_checkin** (แทน list ที่ฮาร์ดโค้ด) |
| `sample_type` | sample | sample_id, name |
| `test_catalog` | testlist | test_id, name, short_code, turnaround_min, job_section, lis_test_id, is_stat_eligible, is_active |
| `test_result_item` | testresultlist | test_result_id, test_id, name, job_section, lis_code, is_critical_monitored |
| `test_group_rule` | testGroupName | short_code_combo, group_code, turnaround_min |
| `lab_order` | RequestData | lab_no (PK), hn, clinic_id, doctor_id, patient_status_id, ordered_at, is_stat, test_group, turnaround_min, checkin_at, called_at, confirmed_at, collected_at (=closeDate), abs_user_id, all_reported_at |
| `lab_order_test` | RequestTest | lab_no, test_id, sample_id, status, reported_at, tech_validated_by/at, med_validated_by/at, is_outlab |
| `stat_case` | Statwarning | id, lab_no (unique), clinic_id, request_codes, test_group, is_fast_track, clock_start_at, due_at, status, reported_at, reported_by, is_late |
| `stat_case_cause` | (คอลัมน์ cause ใน Statwarning) | stat_case_id, cause_id, comment, recorded_by, recorded_at (เก็บประวัติได้หลายครั้ง) |
| `delay_cause` | causeid2 | id, job_section, group, detail, controllability (ควบคุมได้/ไม่ได้), is_active |
| `critical_result` | criticalresult | id, lab_no, test_id, test_result_id, result_value, unit, critical_limit, flag (H/L), status, **received_at, analyzed_at, screened_at, tech_validated_at, med_validated_at, notified_at, doctor_notified_at** (DATETIME ทั้งหมด), notified_by, recipient_name, notify_comment, screen_action, doctor_action, source (LIS/MANUAL) |
| `ward_recipient` | critical_emp_receive | id, clinic_id, name, is_active |
| `service_hour_rule` | (ฮาร์ดโค้ดใน addstatwaning) | ดูหัวข้อ 4.1 |
| `app_user` | ReportUser | id, cas_emp_key (unique), display_name, lis_user_code, abs_user_id, action_pin_hash, is_active |
| `audit_log` | — | who, action, entity, entity_id, before/after (JSON), at, ip |
| `import_file_log` | — | file_name, kind (ORDER/RESULT), status, error, processed_at, checksum |
| `domain_event` | — | id, type, payload, created_at (outbox สำหรับ push real-time) |

### 3.2 State machine

**StatCase**

```text
Pending ──(ผลครบทุก test)──▶ Reported (is_late = reported_at > due_at)
   │                              │ is_late && ยังไม่มี cause ──▶ แสดงใน "รอบันทึกสาเหตุ"
   └──(ลบใบ request/ยกเลิก)──▶ Cancelled
```

**CriticalResult** (ใช้รหัสเดิม 0–3 เพื่อให้ map ข้อมูลเก่าได้)

```text
0 Detected ──NotifyWard──▶ 1 WardNotified ──CompleteTimeline──▶ 2 AwaitDoctorAck ──RecordDoctor/Close──▶ 3 Closed
```

แต่ละ transition ตรวจสิทธิ์ใน Application layer และเขียน `audit_log` ทุกครั้ง

---

## 4. Business logic ที่ปรับปรุง

### 4.1 TurnaroundCalculator (แทนเงื่อนไข 06–08 / 12–13 ที่ฮาร์ดโค้ด)

```text
service_hour_rule
  id | kind          | start  | end    | action
  1  | CLOCK_DEFER   | 06:00  | 08:00  | clock_start = 08:00   (เริ่มนับเมื่อเปิดบริการ)
  2  | CLOCK_DEFER   | 12:00  | 13:00  | clock_start = 13:00   (พักกลางวัน)
  3  | DUE_EXTEND    | 12:00  | 13:00  | due += 60 นาที        (ครบกำหนดตรงพักกลางวัน)
```

- เป็น pure function: `Calculate(collectedAt, turnaroundMin, rules) → (clockStart, due)` และเขียน unit test ทุกกรณีที่ระบบเดิมรองรับ รวมถึงกรณีข้ามเที่ยงคืน
- กฎ "C+S = 60 นาที" ย้ายไปไว้ใน `test_group_rule` แทนการ if ในโค้ด
- **แก้ bug** การไม่ reset ค่า C/S ในลูป (บันทึกเป็น known difference ตอนเทียบผลกับระบบเดิม)

### 4.2 การเตือน (Alerting)

| ระดับ | เงื่อนไข | การแสดงผล |
|---|---|---|
| Overdue | now > due | แถวแดง/ชมพู, เสียงเตือน, badge ที่เมนู |
| < 5 นาที | | สีทอง |
| < 10 นาที | | สีเหลือง |
| < 15 นาที | | สีเหลืองอ่อน |
| FastTrack | is_fast_track | ปักหมุดไว้บนสุด, ตัวอักษรแดง, ไอคอน O₂ |
| Critical รอแจ้ง | status 0 | แถบแดงด้านบนทุกหน้า + เสียง (เปิด/ปิดเสียงได้ต่อเครื่อง) |

- คำนวณเวลาที่เหลือ **ฝั่ง client จาก `due_at` เต็ม** (นับถอยหลังทุกวินาที) ส่วน server push เฉพาะตอนข้อมูลเปลี่ยน
- ใช้ toast/แถบแจ้งเตือนในหน้าแทน popup หลายหน้าต่าง

### 4.3 Ingestion ที่ทนทานขึ้น

- อ่านไฟล์ด้วย `FileSystemWatcher` + poll สำรองทุก 15 วินาที, ใช้ encoding 874, ทำงานแบบ idempotent (upsert ตาม lab_no/test_id และกันไฟล์ซ้ำด้วย checksum)
- **ทำทีละไฟล์ใน transaction** — ถ้าไฟล์ไหน error ให้ย้ายไป `error\` และบันทึกใน `import_file_log` ห้ามหยุดทั้งระบบ และห้ามมี UI ที่ต้องรอคนกด
- HL7 parser แยกเป็น class ที่ test ได้ อ่าน field ตามตำแหน่งในตาราง 2.1 ของเอกสาร 01
- Health check endpoint แสดง: ไฟล์ล่าสุดที่นำเข้า, จำนวนไฟล์ค้าง, จำนวน error และเวลาที่ sync ABS ล่าสุด → ถ้าไม่มีไฟล์เข้าเกิน X นาทีในเวลาทำการ ให้แจ้งบนหน้าเว็บ

---

## 5. Authentication / Authorization (LBCH CAS)

### 5.1 การเชื่อมต่อ (ตาม protocol กลาง)

| รายการ | ค่า |
|---|---|
| AppCode | `STATWARN` (เสนอ — ต้องลงทะเบียนกับผู้ดูแล CAS) |
| Callback | `https://<host>/StatWarning/cas/callback` (ต้องตรงตัวพิมพ์เล็ก-ใหญ่) |
| Login | redirect ไป `{CAS_ADMIN_BASE_URL}/auth/login?appCode=STATWARN&redirectUri=...&state=...` |
| Exchange | `POST {CAS_API_BASE_URL}/api/auth/exchange-code` จาก backend |
| Profile | `GET /api/auth/me?appCode=STATWARN` |
| Local session | Cookie HttpOnly, Secure, SameSite=Lax, Path=`/StatWarning`, 20 นาทีแบบ sliding |
| Token store | ฝั่ง server (`IDistributedCache`/ตาราง) ผูกกับ session; ห้ามส่ง token ไป browser |
| Heartbeat | `GET /auth/session-status` ทุก 1 นาที → `POST /api/auth/session/heartbeat`; ถ้า revoke → 401 → พาไปหน้า login |
| Logout | `POST /auth/logout` (anti-forgery) → refresh ถ้าจำเป็น → `POST /api/auth/logout` → ล้าง local ทุกกรณี |
| Canonical path | middleware redirect `/statwarning` → `/StatWarning` (บทเรียนจาก QIIS) |

### 5.2 Permission (ลงทะเบียนใน CAS)

| Permission | แทนของเดิม | ใช้กับ |
|---|---|---|
| `STATWARN.VIEW` | login ได้ | ดู dashboard, ค้นหา |
| `STATWARN.CAUSE.RECORD` | ทุกคนที่มีรหัส | บันทึกสาเหตุล่าช้า |
| `STATWARN.CRITICAL.NOTIFY` | ทุกคนที่มีรหัส | บันทึกการแจ้งค่าวิกฤต / เติมข้อมูลเวลา |
| `STATWARN.CRITICAL.EDIT` | `critical_status=1` | แก้ไขหรือปิดค่าวิกฤต |
| `STATWARN.WARD.ACK` | (ใหม่) | หอผู้ป่วยบันทึกเวลาแจ้งแพทย์และการรักษา |
| `STATWARN.ORDER.DELETE` | `Accessrights=1` | ลบใบ request หรือ test |
| `STATWARN.REPORT` | — | ดู KPI/สถิติ, export |
| `STATWARN.SETUP` | — | ตั้งค่า master |
| `STATWARN.DISPLAY` | — | โหมดจอแสดงผล (อ่านอย่างเดียว) |

ตรวจสิทธิ์ที่ backend ทุก action (`[RequirePermission]` + policy) ส่วน UI แค่ซ่อนปุ่ม

### 5.3 การลงนามบนเครื่องที่ใช้ร่วมกัน (แทน "กรอกรหัสผ่านเพื่อยืนยัน")

ระบบเดิมให้หลายคนใช้เครื่องเดียวกันและระบุตัวคนทำด้วยรหัสผ่าน ในระบบใหม่เสนอ 2 แบบ (**ต้องตัดสินใจ**):

- **A. ทุกคน login CAS ของตัวเอง** — ง่ายและปลอดภัยที่สุด แต่ต้องสลับผู้ใช้บ่อยบนเครื่องในห้อง lab
- **B. Station session + Action PIN** *(แนะนำ)* — เครื่องเปิดค้างด้วยบัญชีที่มีสิทธิ์ จากนั้นทุก action ที่ต้องลงนามจะเปิดหน้าต่างให้ **เลือกชื่อตัวเอง + กรอก PIN 6 หลัก** (เก็บแบบ hash, จำกัดการลองผิด, ตั้ง/รีเซ็ต PIN หลัง login CAS ด้วยตัวเอง) แล้วบันทึกผู้ลงนามแยกจากผู้ login ใน `audit_log`
  - แก้ปัญหารหัสซ้ำกันของระบบเดิม เพราะระบุตัวคนด้วยชื่อ ไม่ใช่รหัส

### 5.4 Mapping ผู้ใช้

- `app_user.cas_emp_key` = `empKey` จาก CAS (ใช้เป็น linkage key)
- `app_user.lis_user_code` = รหัสใน OBX-16 (techBy/reportBy) ใช้แสดงชื่อผู้รายงานและทำรายงาน OneTechMed
- หา `lis_user_code` ตอน login ด้วยวิธีเดียวกับ LabWeb: ลอง CAS `username` แล้วตามด้วย `cid` เทียบกับ `labdatabase.ID_user.user_code` (ต้องยืนยันว่าเป็นรหัสชุดเดียวกับ `reportuser.Username` — query Q8 เอกสาร 04)
- ถ้าจะ sync ผู้ใช้ล่วงหน้า ใช้ Provisioning API `GET /api/applications/STATWARN/users` (API-client token, scope `admin.user.read`)

---

## 6. หน้าจอ (Sitemap)

| เมนู | หน้า | แทนฟอร์มเดิม |
|---|---|---|
| **Stat Dashboard** | รายการรอผล (นับถอยหลัง, สี, FastTrack) + รายการรายงานแล้ววันนี้ (ทัน/ไม่ทัน) + แถบค่าวิกฤตรอแจ้ง + ปุ่มบันทึกสาเหตุ | ShowData, WarningOut, warning15, warningAddcause |
| | โหมด TV `/display` (ตัวอักษรใหญ่, อ่านอย่างเดียว, refresh อัตโนมัติ) | (ใหม่) |
| | รายละเอียดใบ Request (test, sample, เวลาแต่ละจุด) + ลบใบ/ลบ test | ShowRequestData |
| **ค่าวิกฤต** | คิวรอแจ้ง (status 0) → ฟอร์มแจ้งหอผู้ป่วย | RecordCritical, warningRecordCritical |
| | เติมข้อมูลเวลา (status 1) แบบหลายแถวพร้อมกัน | RecordCriticalCompData |
| | หอผู้ป่วยบันทึกการแจ้งแพทย์ (status 2) แยกตามคลินิก | RecordNotifyDoctor |
| | บันทึกค่าวิกฤตจุลชีวะด้วยมือ | recordCriticalBac (ทำให้ใช้งานได้จริง) |
| | ค้นหา/แก้ไขข้อมูลค่าวิกฤต + timeline T0–T5 | criticalData, EditCriData |
| **ตรวจสอบคุณภาพ** | OneTechMed | OneTechMed |
| **ค้นหา** | ประกันเวลา / สาเหตุ / FastTrack / ผู้ป่วย | searchData |
| **รายงาน** | KPI ค่าวิกฤต (T1–T5, T15, QI28–31, KPI9) รายเดือนตามปีงบประมาณ + กราฟ | StatCritical |
| | Stat ทัน/ไม่ทัน แยกตามงานหรือ test, จำนวนผู้ป่วย Stat | Stat, statInOut |
| | Routine, จุลชีวะ, ห้องเจาะเลือด | RoutineStat, StatMicrobio, StatPhlebotomy |
| **ตั้งค่า** | Test catalog, Test group rule, Clinic/ผู้รับแจ้ง, Delay cause, Service hour rule, Patient status flags | setuptest, setupProblemCause |
| **ระบบ** | สถานะการนำเข้า (import log, health), Audit log, ผู้ใช้และ PIN | (ใหม่) |

ทุกตารางรายงาน: filter ช่วงวัน/ปีงบประมาณ, export .xlsx, แสดง CID แบบ mask เท่านั้น

---

## 7. Non-functional requirements

| ด้าน | เป้าหมาย |
|---|---|
| Latency ของการแจ้งเตือน | ไฟล์ผลเข้า → dashboard อัปเดต ≤ 30 วินาที (p95) |
| ความพร้อมใช้งาน | ใช้งานได้ 24/7; worker restart อัตโนมัติ; หน้าเว็บ reconnect SignalR อัตโนมัติ และแสดงสถานะ "ขาดการเชื่อมต่อ" |
| ความปลอดภัย | HTTPS, secrets ใน environment/DPAPI ไม่อยู่ใน repo, DB user ของแอปมีสิทธิ์เฉพาะ schema ตัวเอง, ABS/HIS ใช้ user read-only แยก |
| PDPA | เข้ารหัส CID (หรือไม่เก็บถ้าไม่จำเป็น), mask ใน UI/export, ห้าม log CID/token, audit การเปิดดูและ export |
| Browser | Edge/Chrome รุ่นปัจจุบัน; จอ 1366×768 ขึ้นไป; หอผู้ป่วยใช้บน tablet ได้ |
| Observability | Serilog structured log, `/health` (DB, CAS API, shared folder, ABS), หน้า import status |
| Backup | ใช้ backup MySQL ของ server เดิม + เพิ่ม schema ใหม่ใน job |

---

## 8. Deployment

```text
Windows Server (IIS)
├─ Site: /StatWarning        → StatWarning.Web  (ASP.NET Core Hosting Bundle, เปิด WebSockets)
└─ Windows Service           → StatWarning.Worker (บัญชี service ที่อ่าน shared folder HL7 ได้)
MySQL 172.16.5.122           → schema statwarning_app (ใหม่) + statwarning (lab core เดิม, worker เขียนแทน StatImportAuto2)
```

- ใช้ config แยกตาม environment (`appsettings.Production.json` + environment variables) และ**ห้ามเขียนทับ config ทั้งไฟล์**ตอน deploy (ตามข้อกำหนด CAS)
- มี environment `Staging` ที่ลงทะเบียน callback ใน CAS แยก
- CI: build + test + publish artifact; deploy ด้วย script (Web Deploy/robocopy + app_offline)

---

## 9. การย้ายข้อมูล (Migration)

1. **ตาราง lab core ไม่ต้อง migrate** — ใช้ร่วมกันต่อ โดย worker ใหม่เขียนแทน StatImportAuto2 (ต้องผ่าน parity test ระดับคอลัมน์)
2. ETL script (C# console ใน Infrastructure) ย้ายเฉพาะข้อมูลของ StatWarning: `statwarning`, `criticalresult`, `criticalmicrobio`, `causeid2`, `critical_emp_receive`, `reportuser`, `deletedata` → `statwarning_app` ตาม mapping ในเอกสาร 04 ข้อ 4
3. รวมคอลัมน์ date และ time ที่แยกกันเป็น DATETIME; ถ้าเวลาเล็กกว่าเวลาของจุดก่อนหน้า ให้บวก 1 วัน (heuristic ข้ามเที่ยงคืน) และบันทึก `migrated_flags`
4. ไม่ย้าย password และ CID ของเจ้าหน้าที่
5. Reconcile: จำนวนแถว, KPI รายเดือนย้อนหลัง 12 เดือน เทียบกับรายงานเดิม (ยอมรับความต่างเฉพาะที่อธิบายได้ด้วย bug ที่แก้)
6. ถ้าพบว่าระบบอื่นอ่าน `statwarning`/`criticalresult` เดิม (query Q1b ในเอกสาร 04) ให้ worker เขียนสำเนาลงตารางเดิมแบบ one-way ต่อไปจนกว่าระบบนั้นจะเปลี่ยน

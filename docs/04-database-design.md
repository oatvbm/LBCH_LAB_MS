# 04 — วิเคราะห์ฐานข้อมูลเดิม และออกแบบฐานข้อมูลใหม่

> แหล่งข้อมูล: [`legacy-schema/statwarning_schema.sql`](legacy-schema/statwarning_schema.sql) (Navicat, structure only, MySQL **8.0.17**, schema `statwarning`, export 2026-11-15)
> อ่านคู่กับ [01-legacy-analysis.md](01-legacy-analysis.md) และ [02-system-design.md](02-system-design.md)
>
> **อัปเดต:** เมื่อทำเป็น LAB_MS ([05](05-platform-roadmap.md) ข้อ 4.1) schema `statwarning_app` ในเอกสารนี้จะแยกเป็น `labms_stat` (stat_case, stat_case_cause, delay_cause, service_hour_rule), `labms_critical` (critical_result, ward_recipient) และ `labms_platform` (app_user, audit_log, import_file_log, domain_event) โดยโครงสร้างตารางเหมือนเดิม

---

## 1. ข้อค้นพบสำคัญ: ฐานข้อมูลนี้ใช้ร่วมกับระบบอื่น

Schema มีทั้งหมด 33 ตารางและ 2 view แต่ **SW2 และ StatImportAuto2 ใช้งานไม่ถึงครึ่ง** ตารางที่เหลือเป็นของ **LabWeb** (`C:\programing\labweb5`) ซึ่งมีโมดูลจุลชีววิทยา, Hemoculture และ Outlab — รายละเอียดว่าใครเขียนคอลัมน์ไหนอยู่ในข้อ 5

LabWeb อ่าน `patient / requestdata / requesttest / testlist / outlab` ที่ **StatImportAuto2 เป็นผู้เขียน** (และเขียนบางคอลัมน์ของ `requesttest`, `outlab`, `criticalresult` เอง) ดังนั้น

> **ถ้าปิด StatImportAuto2 โดยที่ worker ใหม่ไม่เขียนตารางเหล่านี้ในรูปแบบเดิม ระบบอื่นจะหยุดได้รับข้อมูลทันที**

### 1.1 การจัดกลุ่มตาราง

| กลุ่ม | ตาราง | ผู้เขียน / ผู้อ่าน | ทางที่จะไปในระบบใหม่ |
|---|---|---|---|
| **A. Lab core (ใช้ร่วมกัน)** | `patient`, `requestdata`, `requesttest`, `clinic`, `doctor`, `sample`, `patientstatus`, `testlist`, `testresultlist`, `testgroupname`, `testgroup`, `outlab` (insert แถวเริ่มต้น) | Importer เขียน; SW2 และระบบอื่นอ่าน | **คงโครงสร้างเดิมไว้เป็น "สัญญา" กับระบบอื่น** — worker ใหม่ต้องเขียนให้ได้ผลเหมือนเดิมทุกคอลัมน์ ห้ามเปลี่ยนชื่อหรือชนิดคอลัมน์ (เพิ่ม index ได้) |
| **B. ของ StatWarning** | `statwarning`, `criticalresult`, `causeid2`, `critical_emp_receive`, `reportuser`, `filedirectory` | SW2 / importer (**LabWeb insert ค่าวิกฤต Hemoculture ลง `criticalresult`**) | **ย้ายไป schema ใหม่ `statwarning_app`** ตามข้อ 3; `criticalresult` ต้องมี bridge รับข้อมูลจาก LabWeb (ข้อ 5.2) |
| **C. ของ LabWeb** | `resultmb`, `resultmbdrug`, `resultmblogdata`, `mbresultlist`, `mbgradinglist`, `mbdrugresist`, `hemoculture`, `outlab` (คอลัมน์ติดตาม), `outlab_list`, `resultoutlab`, `resultinlab`, `sample_lesion`, `sample_site`, `sample_side`, คอลัมน์ `testlist.cultureStatus/mbResultMapping/TAT/TAT_unit/mbStain/2result`, `requesttest.w_result2` | ระบบอื่น | **ไม่แตะ** (SW2 อ่าน `hemoculture` และ `cultureStatus` แค่ในรายงาน) |
| **D. เลิกใช้หรือไม่พบผู้ใช้** | `causeid` (รุ่นเก่า ใช้ใน EditComment ที่ไม่มีเมนูเรียก), `checktest`, `deletedata`, `criticalmicrobio`, view `pt,ptt`, view `test microbio` | — | `deletedata` และ `criticalmicrobio` ให้ migrate ข้อมูลเข้า `statwarning_app`; ที่เหลือปล่อยไว้ (ถ้าลบต้องให้ DBA ตัดสิน) |

---

## 2. ปัญหาของ schema เดิม

| # | ปัญหา | ตัวอย่าง | ผลกระทบ |
|---|---|---|---|
| 1 | **ไม่มี Foreign Key เลย** | `requestdata.ClinicID` ↛ `clinic` | ข้อมูลกำพร้า, join ผิดโดยไม่รู้ตัว |
| 2 | **Charset/collation ปนกัน** | `utf8mb4_0900_ai_ci` (patient, requestdata, statwarning) กับ `utf8_general_ci` (criticalresult, testresultlist, causeid2, critical_emp_receive) | join ข้ามตารางต้องแปลง collation → ไม่ใช้ index และเสี่ยง error "Illegal mix of collations" |
| 3 | **แยกวันที่กับเวลาคนละคอลัมน์ หรือมีแต่เวลา** | `criticalresult.analyteDT`, `screenDT`, `notify_time`, `notifyDoctor_time` เป็น `TIME` | KPI T1–T5 ผิดเมื่อข้ามเที่ยงคืน |
| 4 | ตัวเลขและเวลาเก็บเป็น `varchar` | `testlist.testTime`, `testgroupname.TestTime`, `deletedata.DeleteDate` | เรียง/คำนวณผิด ต้อง cast ทุกครั้ง |
| 5 | **ตารางไม่มี Primary Key** | `reportuser`, `causeid2`, `clinic`, `patientstatus`, `testgroupname`, `critical_emp_receive`, `resultmblogdata` | แถวซ้ำได้, แก้ไขผิดแถว, replication มีปัญหา |
| 6 | PK ของ `criticalresult` = (labno, testid, testresultid) | — | ถ้า LIS ส่งผลแก้ไข/ผลซ้ำ insert จะ error และค่าวิกฤตครั้งที่ 2 หายไป |
| 7 | สถานะเป็นข้อความ/เลขใน varchar(255) | `notifyStatus`, `StatStatus='ด่วน'`, `requestdata.PatientStatus` เก็บ**ชื่อ**ไม่ใช่ ID | เทียบค่าผิดง่าย เปลี่ยนชื่อแล้วรายงานเพี้ยน |
| 8 | index ซ้ำซ้อน | `patient`: PK(HN) + `idx_HN`(HN); `requesttest` มี index ซ้ำ 3 ตัว | เปลืองพื้นที่ เขียนช้า |
| 9 | **ข้อมูลส่วนบุคคล** | `patient.CID`, **`critical_emp_receive.CID` (CID ของเจ้าหน้าที่)**, `reportuser.Password` plain text | ความเสี่ยงด้าน PDPA |
| 10 | รายงานไปแก้ข้อมูล | หน้า StatCritical รัน `UPDATE criticalresult SET RegisterDate/RegisterTime` **ทุกแถวในตาราง ทีละแถว** ทุกครั้งที่เปิดรายงาน | ช้าลงเรื่อย ๆ และ lock ตาราง |
| 11 | View ที่สร้างไว้ชั่วคราว | `pt,ptt` และ `test microbio` ใส่วันที่ไว้ตายตัว ชื่อมี comma/space | เกะกะ และสับสน |
| 12 | **MySQL 8.0.17** | 8.0 series หมด support (EOL) ไปเมื่อ เม.ย. 2026 | ไม่ได้ security patch → แนะนำวางแผนอัปเกรดเป็น 8.4 LTS แยกเป็นงานโครงสร้างพื้นฐาน |

---

## 3. Schema ใหม่ `statwarning_app` (เฉพาะข้อมูลของ StatWarning)

หลักการ:
- schema ใหม่อยู่บน server เดิม เพื่อให้ join กับ lab core ข้าม schema ได้ (`statwarning.requestdata`)
- ใช้ `utf8mb4` + `utf8mb4_0900_ai_ci` ทั้งหมด
- ทุกตารางมี surrogate PK, FK และ `created_at/updated_at`
- เวลาเป็น `DATETIME` เต็มทุกจุด (Asia/Bangkok)
- เก็บผู้ลงนามทั้งเป็น **user id และ snapshot ชื่อ** (ชื่อตอนลงนาม ใช้ทำรายงาน และรองรับข้อมูลเก่าที่ map เป็น user ไม่ได้)
- ตาราง lab core อ้างอิงด้วย key ธุรกิจ (`lab_no`, `hn`, `test_id`) ไม่สร้าง FK ข้าม schema เพื่อไม่ให้กระทบระบบอื่น

> DDL ด้านล่างเป็น **ร่างสำหรับ review** — ของจริงจะสร้างด้วย EF Core migration ให้ได้ผลลัพธ์ตามนี้

```sql
CREATE DATABASE statwarning_app CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
USE statwarning_app;

-- ผู้ใช้ (แทน reportuser) ------------------------------------------------
CREATE TABLE app_user (
  id                BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  cas_emp_key       VARCHAR(50)  NULL UNIQUE,          -- linkage key จาก CAS
  display_name      VARCHAR(200) NOT NULL,
  lis_user_code     VARCHAR(50)  NULL UNIQUE,          -- = reportuser.Username = OBX-16
  abs_user_id       VARCHAR(50)  NULL,
  action_pin_hash   VARCHAR(255) NULL,                 -- ถ้าเลือกแบบลงนาม B
  pin_failed_count  INT          NOT NULL DEFAULT 0,
  pin_locked_until  DATETIME     NULL,
  legacy_num        VARCHAR(50)  NULL,                 -- reportuser.num
  is_active         TINYINT(1)   NOT NULL DEFAULT 1,
  created_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- กฎเวลาให้บริการ (แทน if 06-08 / 12-13 ในโค้ด) ---------------------------
CREATE TABLE service_hour_rule (
  id              INT         NOT NULL AUTO_INCREMENT PRIMARY KEY,
  kind            ENUM('CLOCK_DEFER','DUE_EXTEND') NOT NULL,
  start_time      TIME        NOT NULL,
  end_time        TIME        NOT NULL,
  defer_to        TIME        NULL,                    -- ใช้กับ CLOCK_DEFER
  extend_minutes  INT         NULL,                    -- ใช้กับ DUE_EXTEND
  is_active       TINYINT(1)  NOT NULL DEFAULT 1,
  note            VARCHAR(255) NULL
);

-- สาเหตุล่าช้า (แทน causeid2) ------------------------------------------
CREATE TABLE delay_cause (
  id               INT          NOT NULL AUTO_INCREMENT PRIMARY KEY,
  legacy_cause_id  VARCHAR(50)  NULL,
  job_section      VARCHAR(100) NOT NULL,
  cause_group      VARCHAR(255) NOT NULL,
  cause_detail     VARCHAR(255) NOT NULL,
  controllability  ENUM('CONTROLLABLE','UNCONTROLLABLE','UNKNOWN') NOT NULL,
  sort_order       INT          NOT NULL DEFAULT 0,
  is_active        TINYINT(1)   NOT NULL DEFAULT 1,
  UNIQUE KEY uq_cause (job_section, cause_group, cause_detail)
);

-- รายการ Stat (แทน statwarning) ----------------------------------------
CREATE TABLE stat_case (
  id                   BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  lab_no               VARCHAR(20)  NOT NULL UNIQUE,
  hn                   VARCHAR(15)  NOT NULL,
  clinic_id            VARCHAR(50)  NULL,
  clinic_name          VARCHAR(255) NULL,              -- snapshot ณ เวลาสร้าง
  request_codes        VARCHAR(50)  NULL,              -- TestShotNameAll
  test_group           VARCHAR(10)  NULL,
  is_fast_track        TINYINT(1)   NOT NULL DEFAULT 0,
  collected_at         DATETIME     NOT NULL,          -- requestdata.CloseDate
  clock_start_at       DATETIME     NOT NULL,          -- หลังปรับตาม service_hour_rule
  turnaround_min       INT          NOT NULL,
  due_at               DATETIME     NOT NULL,
  status               TINYINT      NOT NULL DEFAULT 0, -- 0 Pending, 1 Reported, 9 Cancelled
  reported_at          DATETIME     NULL,
  reported_by_name     VARCHAR(200) NULL,
  reported_by_lis_code VARCHAR(50)  NULL,
  is_late              TINYINT(1)   NULL,
  cancelled_at         DATETIME     NULL,
  cancelled_by_user_id BIGINT       NULL,
  cancel_reason        VARCHAR(255) NULL,
  migrated_flags       VARCHAR(100) NULL,              -- เช่น 'CLOCK_START_INFERRED'
  created_at           DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at           DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  KEY ix_status_due (status, due_at),
  KEY ix_collected (collected_at),
  KEY ix_hn (hn),
  CONSTRAINT fk_sc_cancel_user FOREIGN KEY (cancelled_by_user_id) REFERENCES app_user(id)
);

CREATE TABLE stat_case_cause (
  id                   BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  stat_case_id         BIGINT       NOT NULL,
  delay_cause_id       INT          NULL,              -- NULL ได้สำหรับข้อมูลเก่าที่ map ไม่ได้
  legacy_cause_text    VARCHAR(500) NULL,
  comment              VARCHAR(500) NULL,
  signed_by_user_id    BIGINT       NULL,
  signed_by_name       VARCHAR(200) NOT NULL,
  session_user_id      BIGINT       NULL,              -- ผู้ที่ login อยู่ที่เครื่อง
  recorded_at          DATETIME     NOT NULL,
  is_current           TINYINT(1)   NOT NULL DEFAULT 1, -- แก้ไขแล้วเก็บประวัติแถวเก่าไว้
  KEY ix_case (stat_case_id, is_current),
  CONSTRAINT fk_scc_case  FOREIGN KEY (stat_case_id)   REFERENCES stat_case(id),
  CONSTRAINT fk_scc_cause FOREIGN KEY (delay_cause_id) REFERENCES delay_cause(id),
  CONSTRAINT fk_scc_sign  FOREIGN KEY (signed_by_user_id) REFERENCES app_user(id)
);

-- ผู้รับแจ้งค่าวิกฤตของหอผู้ป่วย (แทน critical_emp_receive; ไม่เก็บ CID) ------
CREATE TABLE ward_recipient (
  id            INT          NOT NULL AUTO_INCREMENT PRIMARY KEY,
  clinic_id     VARCHAR(50)  NOT NULL,
  cas_emp_key   VARCHAR(50)  NULL,
  display_name  VARCHAR(200) NOT NULL,
  position      VARCHAR(100) NULL,
  is_active     TINYINT(1)   NOT NULL DEFAULT 1,
  KEY ix_clinic (clinic_id, is_active)
);

-- ค่าวิกฤต (แทน criticalresult + criticalmicrobio) ------------------------
CREATE TABLE critical_result (
  id                        BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  lab_no                    VARCHAR(20)  NOT NULL,
  hn                        VARCHAR(15)  NOT NULL,
  test_id                   VARCHAR(20)  NOT NULL,
  test_result_id            VARCHAR(20)  NOT NULL,
  source                    ENUM('LIS','LABWEB','MANUAL') NOT NULL DEFAULT 'LIS',
  result_value              VARCHAR(100) NULL,
  result_unit               VARCHAR(50)  NULL,
  critical_limit            VARCHAR(50)  NULL,
  flag                      ENUM('H','L','OTHER') NOT NULL,
  status                    TINYINT      NOT NULL DEFAULT 0,  -- 0 Detected,1 WardNotified,2 AwaitDoctorAck,3 Closed
  supersedes_id             BIGINT       NULL,                 -- ผลแก้ไข/ส่งซ้ำ ชี้ไปแถวเดิม
  -- timeline (DATETIME เต็มทุกจุด)
  registered_at             DATETIME     NULL,                 -- T(-) RegisterDate
  received_at               DATETIME     NULL,                 -- T0 รับตัวอย่าง
  analyzed_at               DATETIME     NULL,                 -- T1
  screened_at               DATETIME     NULL,                 -- T2
  tech_validated_at         DATETIME     NULL,                 -- T3
  med_validated_at          DATETIME     NULL,                 -- T4
  notified_at               DATETIME     NULL,                 -- T5
  doctor_notified_at        DATETIME     NULL,
  closed_at                 DATETIME     NULL,
  -- ผู้เกี่ยวข้อง
  tech_validated_by         VARCHAR(50)  NULL,                 -- LIS code
  med_validated_by          VARCHAR(50)  NULL,
  notified_by_user_id       BIGINT       NULL,
  notified_by_name          VARCHAR(200) NULL,
  recipient_name            VARCHAR(200) NULL,
  ward_recipient_id         INT          NULL,
  notify_comment            VARCHAR(500) NULL,
  screen_action             VARCHAR(255) NULL,
  doctor_action             VARCHAR(500) NULL,
  timeline_by_user_id       BIGINT       NULL,
  timeline_by_name          VARCHAR(200) NULL,                 -- Update_NotifyData
  doctor_ack_by_user_id     BIGINT       NULL,
  doctor_ack_by_name        VARCHAR(200) NULL,                 -- Update_NotifyDoctor
  closed_by_user_id         BIGINT       NULL,
  migrated_flags            VARCHAR(100) NULL,                 -- เช่น 'NOTIFY_DATE_INFERRED'
  created_at                DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at                DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uq_detection (lab_no, test_id, test_result_id, med_validated_at),
  KEY ix_status (status),
  KEY ix_received (received_at),
  KEY ix_lab (lab_no),
  CONSTRAINT fk_cr_super  FOREIGN KEY (supersedes_id)       REFERENCES critical_result(id),
  CONSTRAINT fk_cr_ward   FOREIGN KEY (ward_recipient_id)   REFERENCES ward_recipient(id),
  CONSTRAINT fk_cr_notify FOREIGN KEY (notified_by_user_id) REFERENCES app_user(id)
);

-- ระบบ ---------------------------------------------------------------
CREATE TABLE audit_log (
  id                 BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  occurred_at        DATETIME(3)  NOT NULL,
  session_user_id    BIGINT       NULL,
  signed_by_user_id  BIGINT       NULL,
  action             VARCHAR(100) NOT NULL,     -- เช่น CRITICAL_NOTIFY, ORDER_DELETE, REPORT_EXPORT
  entity_type        VARCHAR(50)  NOT NULL,
  entity_id          VARCHAR(50)  NOT NULL,
  before_json        JSON         NULL,
  after_json         JSON         NULL,
  client_ip          VARCHAR(45)  NULL,
  KEY ix_entity (entity_type, entity_id),
  KEY ix_time (occurred_at)
);

CREATE TABLE import_file_log (
  id             BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  file_name      VARCHAR(255) NOT NULL,
  kind           ENUM('ORDER','RESULT') NOT NULL,
  checksum       CHAR(64)     NOT NULL,
  status         ENUM('OK','ERROR','DUPLICATE') NOT NULL,
  lab_no         VARCHAR(20)  NULL,
  error_message  TEXT         NULL,
  processed_at   DATETIME(3)  NOT NULL,
  duration_ms    INT          NULL,
  KEY ix_checksum (checksum),
  KEY ix_time (processed_at)
);

CREATE TABLE domain_event (           -- outbox สำหรับ push real-time
  id          BIGINT       NOT NULL AUTO_INCREMENT PRIMARY KEY,
  event_type  VARCHAR(100) NOT NULL,
  payload     JSON         NOT NULL,
  created_at  DATETIME(3)  NOT NULL,
  KEY ix_time (created_at)
);
```

### 3.1 การตัดสินใจเรื่องค่าวิกฤตซ้ำ

PK เดิมทำให้ผลที่ LIS ส่งซ้ำหรือแก้ไขภายหลัง insert ไม่ได้ ในระบบใหม่จะทำดังนี้

- ถ้าแถวเดิมยังเป็น status 0 → **อัปเดตแถวเดิม** (ยังไม่ได้แจ้งใคร)
- ถ้าแจ้งไปแล้ว (status ≥ 1) และผลหรือเวลา med validate เปลี่ยน → **สร้างแถวใหม่** ที่ `supersedes_id` ชี้แถวเดิม แล้วเข้าคิวแจ้งใหม่
- KPI นับแถวที่ `supersedes_id IS NULL` เพื่อให้ตรงกับของเดิม (**ต้องยืนยันกับหัวหน้างาน**)

### 3.2 ตาราง lab core: เพิ่มได้ แต่ห้ามเปลี่ยน

ทำได้โดยไม่กระทบระบบอื่น (ทำผ่าน DBA หลังทดสอบบน staging):

- เพิ่ม index: `requestdata(StatStatus, checkInStatus, CloseDate)`, `requesttest(labno, status)`
- ลบ index ซ้ำ (`patient.idx_HN`, `requestdata.idx_labNo`, `requesttest.idx_LabNo`/`idx_rt_lab_test_sample`) — **ทำหลังสุด** และต้องตรวจ query ของระบบอื่นก่อน

ห้ามทำจนกว่าระบบอื่นจะเปลี่ยนตาม: เปลี่ยนชื่อหรือชนิดคอลัมน์, เพิ่ม FK, เปลี่ยน collation

---

## 4. Mapping และกติกาการย้ายข้อมูล

### 4.1 `statwarning` → `stat_case` + `stat_case_cause`

| ปลายทาง | ต้นทาง | กติกา |
|---|---|---|
| lab_no | LabID | ตรวจซ้ำข้ามปีก่อน (query Q2) |
| collected_at | RequestDateTime | |
| due_at | ToReportDateTime | |
| turnaround_min | `requestdata.TimeToReport` | |
| clock_start_at | — | = due_at − turnaround_min; ถ้าไม่มี turnaround ให้ใช้ collected_at และตั้ง flag `CLOCK_START_INFERRED` |
| status | status | '0'→0; '1','2'→1 |
| is_late | ReportInTime | '1'→1, '0'→0 |
| reported_by_name | UserName | |
| stat_case_cause | Comment, causeDiv, causeGroup, causeDetail, causeType, causeBy | สร้างเมื่อ causeGroup ไม่ว่างหรือ Comment ไม่ใช่ ''/'-'; หา `delay_cause` จาก (causeDiv, causeGroup, causeDetail); ถ้าไม่พบให้เก็บข้อความใน `legacy_cause_text`; `signed_by_name` = causeBy |
| (cancel) | `deletedata` | แถวที่ถูกลบ → stat_case status 9 + audit_log |

### 4.2 `criticalresult` (+ `criticalmicrobio`) → `critical_result`

รวมวันที่และเวลาตามลำดับ timeline โดย**ถ้าเวลาน้อยกว่าเวลาของจุดก่อนหน้า ให้บวก 1 วัน** และตั้ง flag (ใช้ได้เมื่อแต่ละช่วงห่างกันไม่ถึง 24 ชม.)

| ปลายทาง | ต้นทาง |
|---|---|
| registered_at | RegisterDate + RegisterTime |
| received_at | RecieveSampleDate + RecieveSampleTime |
| analyzed_at | date(received_at) + analyteDT (+1 วันถ้า < received_at) |
| screened_at | screenDate + screenDT (ถ้าไม่มี screenDate ให้ใช้ date(analyzed_at)) |
| tech_validated_at | tvalDT (DATETIME อยู่แล้ว) |
| med_validated_at | mvalDT (DATETIME อยู่แล้ว) |
| notified_at | date(mvalDT) + notify_time (+1 วันถ้า < mvalDT) |
| doctor_notified_at | date(notified_at) + notifyDoctor_time (+1 วันถ้า < notified_at) |
| status | notifyStatus '0'..'3' |
| notified_by_name / timeline_by_name / doctor_ack_by_name | notifyby / Update_NotifyData / Update_NotifyDoctor (map เป็น user id ด้วยชื่อ ถ้าตรงกันแบบไม่กำกวม) |
| flag | ใช้ค่าจาก criticalvalue/ผลเทียบ (ข้อมูลเดิมไม่ได้เก็บ TPH/TPL ไว้ → `OTHER`) |

### 4.3 Master และผู้ใช้

| ปลายทาง | ต้นทาง | หมายเหตุ |
|---|---|---|
| delay_cause | causeid2 (status='1' → active) | causeType 'ควบคุมได้' → CONTROLLABLE |
| ward_recipient | critical_emp_receive | **ไม่ย้าย CID**; ใช้ `Name`/`emp_fname emp_name emp_lname`; จับคู่ CAS empKey ภายหลัง |
| app_user | reportuser | **ไม่ย้าย Password**; Username → lis_user_code; สิทธิ์ (Accessrights, critical_status) ย้ายไปกำหนดใน CAS |
| service_hour_rule | (ค่าในโค้ด) | seed 3 แถวตามเอกสาร 02 ข้อ 4.1 |
| config ของ worker | filedirectory | ย้ายไป appsettings |

---

## 5. Data contract กับ LabWeb (จาก source `C:\programing\labweb5\lablbch`)

LabWeb เป็น ASP.NET WebForms (VB.NET, .NET Framework 4.7.2) เชื่อม CAS แล้ว (AppCode `LABWEB`, `CasAuth.vb`) และเข้าถึง DB นี้ผ่าน connection `MicroConn` (database `Statwarning`) ในหน้า Hemoculture, ResultBac_*, outlab_record, labcatalog

### 5.1 สิทธิ์การเขียนรายตาราง/คอลัมน์

| ตาราง | Worker ใหม่ (แทน StatImportAuto2) | LabWeb | กติกา |
|---|---|---|---|
| `patient`, `clinic`, `doctor`, `sample`, `patientstatus` | insert/update | อ่าน | worker เป็นเจ้าของ |
| `requestdata` | insert/update ทุกคอลัมน์ (รวม CheckIn/Call/Confirm/**CloseDate**) | อ่าน (`RegisterDate`, `CloseDate`) | worker เป็นเจ้าของ |
| `requesttest` | insert แถว; ตอนผล LIS เข้า set `status`, `ReportDateTime`, `reportBy`, `techby`, `techDateTime`, `StatusResult='3'`; set `statusOutlab`, `statusTestGroup` | update `techby`, `techDatetime`, `reportby`, `reportDatetime`, `StatusResult` (`0`/`2`/`3`) สำหรับงานจุลชีวะ/hemoculture | **ใช้คอลัมน์ร่วมกัน** — worker ต้องทำเหมือนเดิมทุกประการ: update เฉพาะ test ที่ match `testIDLIS` ใน OBX และห้ามแตะแถวอื่น |
| `outlab` | **insert แถวเริ่มต้น** (labno, testid, sampleid) เมื่อ `testShotName='O'` | update คอลัมน์ติดตาม (Outlab, sentBy/Date, confirmOrder*, report*, Notified*) | ถ้า worker ไม่สร้างแถวนี้ หน้า outlab_record จะไม่เห็นรายการ |
| `testlist` | อ่าน | อ่าน (+ คอลัมน์ `cultureStatus`, `mbResultMapping`, `TAT`, `mbStain`, `2result`) | หน้าตั้งค่าของ StatWarning แก้ได้เฉพาะคอลัมน์ของตัวเอง (`testName`, `testShotName`, `testTime`, `testStatus`, `JobSection`, `testIDLIS`, `useStat`) |
| `testresultlist` | อ่าน (`NonCriticalTest`, `testIDLIS`) | อ่าน (หา testResultID ของ hemoculture) | — |
| **`criticalresult`** | (เดิม importer insert เมื่อ OBX TPH/TPL) | **insert/upsert ค่าวิกฤต Hemoculture (Gram stain positive)**: `criticalvalue='Positive'`, `screenAction='Gram stain'`, `tval`=user_code, **ไม่ได้ตั้ง `notifyStatus`** (เป็น NULL) | ดูข้อ 5.2 |
| `statwarning`, `reportuser`, `causeid2`, `critical_emp_receive` | — | **ไม่ใช้** | ย้ายไป `statwarning_app` ได้อย่างปลอดภัย |
| `hemoculture`, `resultmb*`, `mb*`, `outlab_list` | — | เจ้าของ | ไม่แตะ |

`HPV_uploadDataKTB` insert `RequestData` ใน database `HPVloader` (คนละ DB) จึงไม่เกี่ยวข้อง และหน้า `critical_stat.aspx` ของ LabWeb ยังเป็นหน้าว่าง

### 5.2 ค่าวิกฤต Hemoculture ที่มาจาก LabWeb

LabWeb insert ลง `criticalresult` เดิมโดยตรง ดังนั้นระบบใหม่ต้องรับข้อมูลนี้ด้วย:

- **ระยะสั้น (ไม่ต้องแก้ LabWeb)**: worker มีงาน `LegacyCriticalBridge` อ่าน `statwarning.criticalresult` แถวที่ LabWeb เขียน/แก้ (เทียบกับ checksum ล่าสุด) แล้ว upsert เข้า `statwarning_app.critical_result` ด้วย `source='LABWEB'` (ต้องเพิ่มค่านี้ใน enum)
- **ระยะยาว**: แก้ LabWeb ให้เรียก internal API ของ StatWarning (`POST /api/internal/critical-results`, ยืนยันตัวด้วย API key ฝั่ง server) แทนการ insert ตารางตรง แล้วเลิกใช้ `criticalresult` เดิม
- จุดที่ต้องยืนยัน:
  - แถวที่ LabWeb ใส่มี `notifyStatus = NULL` ซึ่ง**ไม่เข้าคิว `notifyStatus='0'` ของ SW2** แล้วปัจจุบันแจ้งหอผู้ป่วยและบันทึกผ่านช่องทางไหน? (ดู query Q7)
  - เวลาของ hemoculture ข้ามหลายวัน (load → detect positive) แต่ถูกตัดเหลือแค่ `TIME` ใน `analyteDT` → T1/T2 ของ hemoculture ในรายงานเดิมไม่มีความหมาย ระบบใหม่เก็บ DATETIME เต็มจึงแก้ได้
  - ถ้า LIS ส่ง ORU ของ test เดียวกันมาภายหลัง importer เดิมจะ insert ไม่ได้ (PK ชน) — ระบบใหม่ใช้กติกาข้อ 3.1

### 5.3 การระบุผู้ใช้ให้ตรงกันทั้งสองระบบ

- LabWeb map ผู้ใช้ CAS (`username` หรือ `cid`) → `labdatabase.ID_user.user_code` แล้วเขียน `user_code` ลง `requesttest.techby/reportby` และ `criticalresult.tval`
- SW2/importer เดิม map `requesttest.reportBy` → `reportuser.Username`
- **ข้อเสนอ**: StatWarning ใช้วิธีเดียวกับ LabWeb (CAS → `ID_user.user_code`) แล้วเก็บไว้ใน `app_user.lis_user_code` เพื่อให้รหัสผู้ใช้ตรงกันทุกระบบของ lab — **ต้องยืนยันว่า `ID_user.user_code` = `reportuser.Username` = รหัสใน OBX-16** (query Q8)

### 5.4 ข้อสังเกตด้านความปลอดภัยของ LabWeb (นอกขอบเขต แต่ควรรู้)

- `Web.config` เก็บรหัสผ่าน DB แบบ plain text และใช้บัญชี `lablbch` เดียวกับ SW2/StatImportAuto2 ซึ่งเข้าถึงได้ทุก database
- ข้อเสนอ: แยก DB user ต่อระบบ (StatWarning worker, StatWarning web, LabWeb) และให้สิทธิ์เฉพาะตาราง/คอลัมน์ตาม contract ข้อ 5.1 แล้วเปลี่ยนรหัส `lablbch` หลังย้ายระบบเสร็จ

---

## 6. Query ที่ต้องรันก่อนสรุปการออกแบบ

รันบน server จริงด้วย Navicat (อ่านอย่างเดียว) แล้วส่งผลกลับมา — **ผลเป็นตัวเลขสรุป ไม่มีข้อมูลผู้ป่วย**

### 6.1 ใครใช้ฐานข้อมูลนี้บ้าง (ยืนยันว่ามีแค่ SW2, StatImportAuto2, LabWeb)

```sql
-- Q1a: เครื่อง/บัญชีที่เชื่อมต่ออยู่ตอนนี้
SELECT user, SUBSTRING_INDEX(host, ':', 1) AS client, db, COUNT(*) AS conns
FROM information_schema.processlist
GROUP BY user, client, db ORDER BY conns DESC;

-- Q1b: query ที่เคยรันกับตารางของ StatWarning (ดูว่ามีระบบอื่นอ่านหรือไม่)
SELECT LEFT(DIGEST_TEXT, 300) AS query_pattern, COUNT_STAR, LAST_SEEN
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'statwarning'
  AND (DIGEST_TEXT LIKE '%`statwarning`%' OR DIGEST_TEXT LIKE '%`criticalresult`%'
       OR DIGEST_TEXT LIKE '%`reportuser`%' OR DIGEST_TEXT LIKE '%`causeid2`%')
ORDER BY LAST_SEEN DESC;
```

### 6.2 คุณภาพข้อมูล

```sql
-- Q2: LabID ซ้ำข้ามปีใน statwarning
SELECT COUNT(*) FROM (SELECT LabID FROM statwarning GROUP BY LabID HAVING COUNT(*) > 1) t;

-- Q3: ค่าที่มีจริงของคอลัมน์สถานะ
SELECT 'sw.status' k, status v, COUNT(*) n FROM statwarning GROUP BY status
UNION ALL SELECT 'sw.ReportInTime', ReportInTime, COUNT(*) FROM statwarning GROUP BY ReportInTime
UNION ALL SELECT 'cr.notifyStatus', notifyStatus, COUNT(*) FROM criticalresult GROUP BY notifyStatus
UNION ALL SELECT 'rd.StatStatus', StatStatus, COUNT(*) FROM requestdata GROUP BY StatStatus
UNION ALL SELECT 'rd.PatientStatus', PatientStatus, COUNT(*) FROM requestdata GROUP BY PatientStatus;

-- Q4: ค่าวิกฤตที่น่าจะข้ามเที่ยงคืน (เวลาแจ้งน้อยกว่าเวลา med validate)
SELECT COUNT(*) AS rows_total,
       SUM(notify_time < TIME(mvalDT)) AS notify_before_mval,
       SUM(analyteDT < RecieveSampleTime) AS analyte_before_receive,
       SUM(screenDT < analyteDT) AS screen_before_analyte
FROM criticalresult;

-- Q5: ขนาดตาราง
SELECT table_name, table_rows, ROUND((data_length + index_length)/1024/1024,1) AS mb
FROM information_schema.tables WHERE table_schema = 'statwarning' ORDER BY mb DESC;

-- Q6: causeid2 และ reportuser ซ้ำ
SELECT COUNT(*) total, COUNT(DISTINCT jobsection, causeGroup, causeDetail) uniq FROM causeid2;
SELECT COUNT(*) total, COUNT(DISTINCT Username) uniq_user, COUNT(DISTINCT Password) uniq_pwd FROM reportuser WHERE UseStatus = '1';
```

```sql
-- Q7: ค่าวิกฤตที่ LabWeb ใส่ (Hemoculture) อยู่ในสถานะไหน
SELECT COALESCE(notifyStatus, 'NULL') AS notifyStatus, COUNT(*) n,
       SUM(mvalDT IS NOT NULL) has_mval, SUM(notify_time IS NOT NULL) has_notify
FROM criticalresult WHERE criticalvalue = 'Positive' GROUP BY notifyStatus;

-- Q8: รหัสผู้ใช้ของ LabWeb (ID_user) กับ SW2 (reportuser) ตรงกันแค่ไหน — นับอย่างเดียว
SELECT COUNT(*) AS reportuser_active,
       SUM(EXISTS (SELECT 1 FROM labdatabase.ID_user u
                   WHERE u.user_code COLLATE utf8mb4_0900_ai_ci = r.Username)) AS matched_in_id_user
FROM statwarning.reportuser r WHERE r.UseStatus = '1';
```

> Q6 บรรทัดสุดท้ายนับเฉพาะจำนวน ไม่ได้แสดงรหัสผ่าน ถ้า `uniq_pwd < total` แปลว่ามีผู้ใช้ที่รหัสซ้ำกันอยู่จริง ซึ่งยืนยัน bug การลงนามด้วยรหัสผ่านของระบบเดิม

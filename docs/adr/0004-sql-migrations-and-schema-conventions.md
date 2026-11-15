# ADR-0004: SQL migration ต่อโมดูลและข้อตกลงการตั้งชื่อ schema

- สถานะ: ตัดสินแล้ว (2026-11-15)
- ผู้ตัดสิน: tech lead
- ที่มา: docs/05 ข้อ 3.2, 4; docs/04 ข้อ 3, 3.2; task card R0 ข้อ Q3, Q5; Lab_Stock ADR-019

## บริบท

- DB server เดียวใช้ร่วมกับ LabWeb, LQMS และระบบอื่น การเปลี่ยน schema ต้องควบคุมเวลาได้และตรวจทานได้โดย DBA
- แต่ละโมดูลเป็นเจ้าของ schema ของตัวเอง (ADR-0001) จึงต้อง migrate แยกต่อโมดูลได้
- Lab_Stock ใช้ SQL migration + `schema_migrations` แล้วได้ผลดี และพบว่าการผสม EF migrations กับ SQL script ทำให้สร้างตารางซ้ำ

## การตัดสินใจ

### Migration

1. schema เปลี่ยนด้วย **SQL script** ที่ `database/<module>/NNNN_<snake_name>.sql` (เลข 4 หลักต่อโมดูล) — **ไม่ใช้ EF Core migrations**
2. บันทึกการรันใน `labms_platform.schema_migrations` (module, version, name, checksum SHA-256, applied_at, applied_by)
3. script ที่รันแล้ว **ห้ามแก้** — runner ต้องปฏิเสธเมื่อ checksum เปลี่ยน; แก้ไขด้วย script ใหม่เท่านั้น
4. รัน migration ด้วย console **`tools/Lbch.LabMs.DbMigrate`** โดยคนตอน deploy — **web/worker ห้าม migrate อัตโนมัติตอน start**
5. บัญชี DB ของ migrator แยกจากบัญชี web/worker (ADR-0007) และมีสิทธิ์ DDL เฉพาะ schema `labms_*`
6. script ต้องรันได้บน MySQL 8.0 และ 8.4 (ADR-0002) และรันซ้ำได้อย่างปลอดภัยเท่าที่ทำได้ (`IF NOT EXISTS`)
7. การเปลี่ยนตาราง lab core เดิม (`statwarning.*`) **ไม่อยู่ใน migration ของ LAB_MS** — ทำผ่าน DBA และบันทึกใน `docs/legacy-changes.md`

### ข้อตกลงการตั้งชื่อ

| เรื่อง | ข้อตกลง | ตัวอย่าง |
|---|---|---|
| schema | `labms_<module>` ตัวพิมพ์เล็ก | `labms_platform`, `labms_stat` |
| ตาราง | snake_case **เอกพจน์** | `app_user`, `audit_log`, `stat_case` |
| คอลัมน์ | snake_case; เวลา `*_at` (`DATETIME`), flag `is_*`, FK `<table>_id` | `created_at`, `is_active`, `stat_case_id` |
| PK | surrogate `id` (`BIGINT` หรือ `INT` AUTO_INCREMENT) | |
| constraint/index | `pk_`, `fk_<table>_<ref>`, `uq_<table>_<cols>`, `ix_<table>_<cols>` | `ix_stat_case_status_due` |
| charset | `utf8mb4` + `utf8mb4_0900_ai_ci` ทุกตาราง | |

- ทุกตารางมี `created_at`, `updated_at`; ข้อมูลที่คนแก้ได้มีผู้แก้ (`updated_by_user_id`) หรือบันทึกผ่าน `audit_log`
- เวลาเก็บเป็น `DATETIME` เต็มตามเวลาไทย (Asia/Bangkok ไม่มี DST) — ห้ามแยกวันที่กับเวลาคนละคอลัมน์
- สถานะเก็บเป็นรหัส (`TINYINT`/`ENUM`/`VARCHAR` รหัสอังกฤษ) ไม่ใช่ข้อความไทย
- ใช้ FK จริงภายใน schema ของโมดูล; อ้างถึง lab core เดิมหรือโมดูลอื่นด้วย key ธุรกิจ (`lab_no`, `hn`, `test_id`) **ไม่สร้าง FK ข้าม schema**

## ทางเลือกที่ไม่เลือก

- **EF Core migrations**: ผูก schema กับ model ทำให้ DBA ตรวจ diff ยาก และเสี่ยงสร้างตารางซ้ำกับ script (บทเรียน Lab_Stock ADR-019)
- **auto-migrate ตอน start**: เปลี่ยน schema บน server ที่ใช้ร่วมกันโดยไม่มีคนควบคุม และหลาย instance อาจรันพร้อมกัน
- **ชื่อตารางพหูพจน์แบบ Lab_Stock (`audit_logs`)**: เอกสาร 04/05 ใช้เอกพจน์อยู่แล้ว ตอนย้าย Lab_Stock ใน R2 ค่อย map

## ผลที่ตามมา

- R0-03 สร้าง migration runner, `tools/Lbch.LabMs.DbMigrate` และ `database/platform/0001_platform_baseline.sql`
- R0-09 deploy script ต้องมีขั้นรัน migrator ก่อนเปิดเว็บ
- DDL ร่างในเอกสาร 04 ข้อ 3 ต้องเขียนใหม่เป็น script ต่อโมดูลตามข้อตกลงนี้ใน R1

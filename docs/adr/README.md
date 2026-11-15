# Architecture Decision Records (ADR)

บันทึกการตัดสินใจเชิงออกแบบของ LAB_MS เพื่อให้คนและ agent ทุกตัวอ้างอิงที่เดียว (AGENTS.md ข้อ 3.6)

## กติกา

- 1 การตัดสินใจ = 1 ไฟล์ `NNNN-<slug>.md` (เลข 4 หลักเรียงต่อกัน ไม่นำเลขกลับมาใช้)
- agent **เสนอ** ADR ได้ (สถานะ `เสนอ`) แต่เปลี่ยนเป็น `ตัดสินแล้ว` ได้เมื่อ tech lead อนุมัติใน PR เท่านั้น
- ADR ที่ตัดสินแล้ว **ห้ามแก้เนื้อหาการตัดสินใจย้อนหลัง** — ถ้าเปลี่ยนใจ ให้เขียน ADR ใหม่ แล้วเปลี่ยนสถานะฉบับเดิมเป็น `แทนที่โดย ADR-XXXX` (แก้คำผิด/ลิงก์ได้)
- ADR ที่ยกมาจาก Lab_Stock (`C:\programing\Lab_Stock\docs\DECISIONS.md`) ให้ระบุเลขต้นทางในหัวข้อ "อ้างอิง"
- เขียนภาษาไทย ชื่อเทคนิค/identifier เป็นภาษาอังกฤษ

## สถานะ

`เสนอ` → `ตัดสินแล้ว` → (`แทนที่โดย ADR-XXXX` | `เลิกใช้`)

## รูปแบบ

```markdown
# ADR-NNNN: <ชื่อการตัดสินใจ>

- สถานะ: เสนอ | ตัดสินแล้ว (YYYY-MM-DD) | แทนที่โดย ADR-XXXX
- ผู้ตัดสิน: tech lead
- ที่มา: docs/.. ข้อ .. / Lab_Stock ADR-..

## บริบท
ปัญหาและข้อจำกัดที่ทำให้ต้องตัดสินใจ

## การตัดสินใจ
สิ่งที่เลือก เขียนให้ตรวจได้ (ใช้คำว่า "ต้อง" / "ห้าม")

## ทางเลือกที่ไม่เลือก
ทางเลือกและเหตุผลสั้น ๆ

## ผลที่ตามมา
สิ่งที่ต้องทำต่อ ข้อจำกัด และวิธีตรวจ (เช่น architecture test)
```

## ทะเบียน

| ADR | เรื่อง | สถานะ |
|---|---|---|
| [0001](0001-modular-monolith.md) | Modular monolith และกติกาโมดูล | ตัดสินแล้ว |
| [0002](0002-mysql-database.md) | ใช้ MySQL ต่อ, เป้าหมาย 8.4 LTS และฐาน development | ตัดสินแล้ว |
| [0003](0003-ef-core-provider-and-dapper.md) | EF Core provider `MySql.EntityFrameworkCore` + Dapper | ตัดสินแล้ว |
| [0004](0004-sql-migrations-and-schema-conventions.md) | SQL migration ต่อโมดูลและข้อตกลงการตั้งชื่อ schema | ตัดสินแล้ว |
| [0005](0005-cas-authentication.md) | Authentication ผ่าน LBCH CAS | ตัดสินแล้ว |
| [0006](0006-permission-catalog-and-permission-sets.md) | Permission catalog และชุดสิทธิ์กำหนดโดยระบบ | ตัดสินแล้ว |
| [0007](0007-secrets-db-accounts-and-personal-data.md) | Secrets, บัญชี DB และข้อมูลส่วนบุคคล | ตัดสินแล้ว |
| [0008](0008-ai-team-workflow-and-ci.md) | การทำงานแบบทีม AI, CI และการยกโค้ดจาก Lab_Stock | ตัดสินแล้ว |

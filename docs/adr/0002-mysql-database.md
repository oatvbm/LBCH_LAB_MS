# ADR-0002: ใช้ MySQL ต่อ, เป้าหมาย 8.4 LTS และฐาน development

- สถานะ: ตัดสินแล้ว (2026-11-15)
- ผู้ตัดสิน: tech lead
- ที่มา: docs/05 ข้อ 1 (D10), 1.0, 5.1; docs/04 ข้อ 2 (#12); task card R0 ข้อ Q1; Lab_Stock ADR-012, ADR-019

## บริบท

- ข้อมูลเดิมอยู่บน MySQL 8.0.17 (172.16.5.122) และ LabWeb ใช้ตาราง lab core ใน `statwarning` ร่วมกันจนจบ R3
- Lab_Stock, Lab_RMS, QIIS ใช้ MySQL อยู่แล้ว ทีมคุ้นกับ MySQL/Navicat
- ไม่มี SQL Server Standard ที่มี license และคนดูแล ส่วน Express จำกัด 10 GB
- MySQL 8.0 หมด support (EOL เม.ย. 2026) ต้องวางแผนอัปเกรด

## การตัดสินใจ

1. LAB_MS ใช้ **MySQL** บน server เดิม โดยสร้าง schema ใหม่ `labms_*` ข้าง schema เดิม (`statwarning`, `labdatabase`) เพื่อ join ข้าม schema ได้ระหว่างเปลี่ยนผ่าน
2. เป้าหมาย production คือ **MySQL 8.4 LTS** — อัปเกรดเป็นงานแยกร่วมกับ DBA (legacy-changes LC-014) และ **ไม่บล็อก R0/R1**
3. โค้ดต้องทำงานได้ทั้ง **8.0 และ 8.4** จนกว่า production จะอัปเกรด — ห้ามใช้ฟีเจอร์ที่มีเฉพาะ 8.4
4. **ฐาน development**: MySQL **8.4 ใน Docker** บนเครื่องผู้พัฒนา ข้อมูลสังเคราะห์เท่านั้น
5. **Integration test**: Testcontainers (MySQL 8.4) บนเครื่อง dev; CI รันกับ **ทั้ง 8.0 และ 8.4** (matrix)
6. ชดเชยข้อด้อยของ MySQL เมื่อเทียบ SQL Server:
   - ประวัติการแก้ไข → ตาราง `audit_log` ของ platform (before/after JSON) แทน temporal table
   - PDPA → เข้ารหัส CID ระดับแอป (ADR-0007) และจำกัดสิทธิ์ DB account ตาม schema
7. **ห้ามใช้ฐาน production สำหรับพัฒนาหรือทดสอบ** และห้ามให้ agent ใช้ connection string ของ production

## ทางเลือกที่ไม่เลือก

- **SQL Server**: ข้อดีเชิงเทคนิค (provider ของ Microsoft, temporal table, หลาย schema) แต่ไม่มี license/คนดูแล และต้องเขียนสองฐานข้อมูลระหว่างเปลี่ยนผ่าน (docs/05 ข้อ 1.0)
- **instance MySQL 8.0.17 แบบ AppServ แบบ Lab_Stock (ADR-019)**: ต้องดูแล instance เอง ต้องเริ่มด้วย script หลังรีสตาร์ต และไม่ตรงกับเป้าหมาย 8.4 — เครื่อง dev มี Docker แล้ว

## ผลที่ตามมา

- R0-02: CI matrix MySQL 8.0 / 8.4
- R0-03: `docs/DEVELOPMENT_SETUP.md` อธิบายการเปิด MySQL dev ใน Docker
- collation: schema ใหม่ใช้ `utf8mb4_0900_ai_ci`; query ที่ join ตารางเดิมที่เป็น `utf8_general_ci` ต้องระบุ `COLLATE` ชัดเจนและวัด performance (docs/03 ความเสี่ยง)

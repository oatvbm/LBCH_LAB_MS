# ADR-0003: EF Core provider `MySql.EntityFrameworkCore` + Dapper สำหรับรายงาน

- สถานะ: ตัดสินแล้ว (2026-11-15)
- ผู้ตัดสิน: tech lead
- ที่มา: docs/05 ข้อ 1 (D7), 5; Lab_Stock ADR-018

## บริบท

LAB_MS ใช้ .NET 10 LTS + EF Core 10 (สถานะ ณ พ.ย. 2026):

| Provider | สถานะ |
|---|---|
| `MySql.EntityFrameworkCore` (Oracle) 10.x | ทางการ รองรับ EF Core 10 / .NET 10 และ MySQL 8.0+; Lab_Stock ใช้ 10.0.12 แล้ว |
| `Pomelo.EntityFrameworkCore.MySql` | รุ่นล่าสุด 9.0.0 ยังไม่รองรับ EF Core 10 |
| `Microting.EntityFrameworkCore.MySql` | fork ของ Pomelo ผู้ดูแลกลุ่มเล็ก — เสี่ยงระยะยาว |

## การตัดสินใจ

1. งานเขียน/อ่านทั่วไปใช้ **EF Core 10 + `MySql.EntityFrameworkCore` 10.x** (ปักเวอร์ชันใน `Directory.Packages.props`)
2. รายงาน/KPI และ query ที่ซับซ้อนใช้ **Dapper + SQL แบบ parameterized** — ไม่พึ่งการแปล LINQ ของ provider
3. **อ้างอิง provider (`MySql.*`) และ Dapper ได้เฉพาะในโปรเจกต์ `*.Infrastructure`** — Domain/Application/Web ห้ามอ้างโดยตรง
4. ห้ามใช้ Pomelo หรือ fork ของ Pomelo
5. **ไม่ใช้ EF Core migrations** — schema จัดการด้วย SQL script (ADR-0004); DbContext ใช้ map ตารางที่มีอยู่เท่านั้น
6. ทุก repository/query ต้องมี integration test กับ MySQL จริง (ADR-0002) เพื่อจับ query ที่ provider แปลไม่ได้ตั้งแต่ CI

## ทางเลือกที่ไม่เลือก

- **Pomelo**: ติดอยู่ที่ EF Core 9 ใช้กับ .NET 10/EF Core 10 ไม่ได้
- **Microting fork**: API เหมือน Pomelo แต่ความเสี่ยงด้านผู้ดูแลสูงสำหรับระบบโรงพยาบาล
- **Dapper อย่างเดียว**: เขียน CRUD และ unit of work เองมากเกินไป

## ผลที่ตามมา

- Lab_RMS (Pomelo 8.0.3) ต้องเปลี่ยน provider เมื่อย้ายเข้า R2
- ถ้า Pomelo ออกรุ่น EF Core 10 ในอนาคต เปลี่ยนได้ใน Infrastructure โดยไม่กระทบชั้นอื่น (ต้องเขียน ADR ใหม่)
- R0-02: architecture test ตรวจข้อ 3

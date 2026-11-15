# ADR-0006: Permission catalog และชุดสิทธิ์กำหนดโดยระบบ

- สถานะ: ตัดสินแล้ว (2026-11-15)
- ผู้ตัดสิน: tech lead
- ที่มา: docs/05 ข้อ 4, 4.1; docs/02 ข้อ 5.2 (ถูกแทนที่ด้วยข้อนี้); Lab_Stock ADR-013, ADR-021, ADR-022

## บริบท

- เอกสาร 02 เดิมเสนอให้ลงทะเบียน permission ราย action ใน CAS (`STATWARN.*`) แต่ LAB_MS มีหลายโมดูลและ permission จะเพิ่มทุกระยะ การให้ผู้ดูแล CAS แก้ทุกครั้งเป็นคอขวด
- Lab_Stock ใช้แนวทาง "CAS ตรวจสิทธิ์เข้าแอป + permission catalog และชุดสิทธิ์ภายในแอป" แล้วได้ผลดี
- ระบบเดิมตรวจสิทธิ์แค่ซ่อนปุ่มบน client (docs/01 ข้อ 3.1.5)

## การตัดสินใจ

1. **CAS ตรวจเฉพาะสิทธิ์เข้า AppCode `LAB_MS`**; สิทธิ์ราย action อยู่ใน **permission catalog ของ LAB_MS**
2. **Permission catalog กำหนดโดยระบบ** — แต่ละโมดูลประกาศ permission ของตัวเองในโค้ด (`ILabMsModule`) และลง DB ผ่าน migration (ADR-0004); รูปแบบชื่อ `<module>.<resource>.<action>` ตัวพิมพ์เล็ก เช่น `platform.users.manage`, `stat.cause.record`, `critical.notify`
3. **ชุดสิทธิ์ (permission set)** กำหนดโดยระบบเช่นกัน แบ่งตามบทบาทงาน — ผู้ดูแล **มอบหมาย/ถอน** ชุดสิทธิ์ให้ผู้ใช้ได้ แต่ **สร้างชุดใหม่หรือแก้ permission ในชุดไม่ได้**; เปลี่ยน catalog/ชุดสิทธิ์ผ่านการพัฒนา + migration ที่ตรวจทานได้
4. การมอบหมายมีขอบเขตหน่วยงานได้ (`NULL` = ไม่จำกัด); ขอบเขตอนุมัติใช้หน่วยงาน + management roles จาก CAS Profile
5. **ตรวจสิทธิ์ฝั่ง server ทุก action** ด้วย `[RequirePermission("...")]` + authorization policy; UI ซ่อนปุ่มได้แต่ไม่ใช่การตรวจสิทธิ์; ไม่มีสิทธิ์ → `403` + audit
6. ผู้ดูแล (`platform.users.manage`) ค้นผู้ใช้ที่ CAS เปิดสิทธิ์ `LAB_MS` ผ่าน CAS API client (scope `admin.user.read`) แล้วเพิ่ม/ซิงก์เป็นรายคน; ใช้ `empKey` เป็น key; การซิงก์ไม่เปิดบัญชีที่ LAB_MS ปิดไว้
7. ทุกการเปลี่ยนผู้ใช้/การมอบหมายตรวจสิทธิ์ซ้ำและเขียน `audit_log` ใน transaction เดียวกัน
8. ระบบป้องกัน: ปิดบัญชีตัวเอง, ถอนชุดสิทธิ์ผู้ดูแลจากบัญชีที่กำลังใช้งาน, และสถานะที่ไม่เหลือผู้ดูแลที่เข้าแอปได้
9. ผู้ดูแลคนแรกสร้างด้วย SQL bootstrap ที่ tech lead รันเอง — **ไม่มีชื่อบุคคลหรือ empKey ใน repo**

## ทางเลือกที่ไม่เลือก

- **permission ราย action ใน CAS (docs/02 ข้อ 5.2)**: ต้องพึ่งผู้ดูแล CAS ทุกครั้งที่เพิ่ม action และ CAS ไม่รู้ขอบเขตหน่วยงานของงาน lab
- **ให้ผู้ดูแลสร้าง role/แก้ matrix เอง**: ตรวจย้อนหลังยาก และเสี่ยงให้สิทธิ์เกินโดยไม่ผ่านการทบทวน

## ผลที่ตามมา

- permission ของ StatWarning เปลี่ยนจาก `STATWARN.*` เป็น `stat.view`, `stat.cause.record`, `critical.notify`, `critical.edit`, `order.delete`, `report.view`, `setup.manage` (docs/05 ข้อ 4.1) — ลงทะเบียนใน R1-00
- R0-06 สร้างตาราง `app_user`, `permission`, `permission_set`, `permission_set_permission`, `user_permission_set` และหน้าผู้ดูแล
- ผู้ใช้ที่ย้ายจาก `LABSTOCK`/`LRMS` ใน R2 ต้อง map ชุดสิทธิ์เดิมเข้าชุดสิทธิ์ของ LAB_MS

# ADR-0001: Modular monolith และกติกาโมดูล

- สถานะ: ตัดสินแล้ว (2026-11-15)
- ผู้ตัดสิน: tech lead
- ที่มา: docs/05 ข้อ 1 (D1, D2, D5, D6, D8), 3, 3.2, 3.3, 10

## บริบท

งาน lab กระจายอยู่หลายระบบ (StatWarning, LabWeb, LQMS, Lab_Stock, LRMS, HPV) ต่างเทคโนโลยี ต่าง login และใช้ DB ร่วมกันโดยไม่มีขอบเขตชัด
ทีมพัฒนาเล็ก (tech lead 1 คน + agent) ดูแลหลาย service แยกกันไม่ไหว แต่ถ้ารวมเป็นก้อนเดียวไม่มีขอบเขต จะแก้ยากและแบ่งงานให้ agent คู่ขนานไม่ได้

## การตัดสินใจ

1. LAB_MS เป็น **แอปเดียวแบบแบ่งโมดูล (modular monolith)**: web host เดียว `Lbch.LabMs.Web` (IIS `/LAB_MS`) + worker เดียว `Lbch.LabMs.Worker` (Windows Service)
2. ทยอยย้ายระบบเดิมทีละโมดูลตาม roadmap R0–R5 — **ไม่เขียนใหม่ทั้งหมดในครั้งเดียว**
3. จัดโมดูลเป็น 2 ด้าน + ส่วนกลาง: `Service`, `Quality`, `Platform` และ `Core` (lab core) ตามโครงสร้าง docs/05 ข้อ 3.2
4. แต่ละโมดูลแบ่งเป็น `Domain | Application | Infrastructure | Web(Area)` และมี assembly สาธารณะ (`*.Contracts`) สำหรับให้โมดูลอื่นเรียก
5. **กติกาโมดูล** (ต้องบังคับด้วย architecture test ใน CI):
   1. โมดูลมี schema DB ของตัวเอง (`labms_<module>`) และ **เขียนได้เฉพาะ schema ตัวเอง**
   2. ข้ามโมดูลได้ผ่าน interface สาธารณะ (`*.Contracts`) หรือ in-process domain event เท่านั้น — **ห้าม** reference `Domain`/`Infrastructure` ของโมดูลอื่น และห้าม query ตารางของโมดูลอื่น
   3. Lab core อ่านได้ทุกโมดูล แต่เขียนได้เฉพาะ Worker และ API ที่ Core เปิดไว้
   4. `Platform.Kernel` ไม่มี dependency ไปยัง Infrastructure, Web, EF Core หรือโมดูลใด
6. เปิด/ปิดโมดูลด้วย feature flag ใน config (`Modules:<name>:Enabled`) เพื่อ deploy ทีละส่วน — โมดูลที่ปิดไม่ลง route, เมนู และ permission
7. ระบบที่ **ไม่รวม**: BBM, `Risk_record` ของ LabWeb, QIIS (แอปแยก รับค่าผ่าน API), CAS, Riskincident

## ทางเลือกที่ไม่เลือก

- **Microservices / แอปแยกต่อระบบ**: ต้องดูแลหลาย deploy, หลาย login, หลาย DB account — เกินกำลังทีม
- **Monolith ไม่แบ่งโมดูล**: แบ่งงานให้ agent คู่ขนานไม่ได้ (ไฟล์ชนกัน) และผูกกันจนย้าย/ปิดทีละส่วนไม่ได้
- **ต่อยอด LabWeb (WebForms)**: เทคโนโลยีไม่พัฒนาต่อ ทำ real-time และ test ยาก (docs/02 ข้อ 2.1)

## ผลที่ตามมา

- R0-02 ต้องมี architecture test (NetArchTest) ตรวจข้อ 5 ตั้งแต่ยังไม่มีโมดูลธุรกิจ
- ระบบเดียวล่มกระทบทุกงาน → Worker แยก process, health check ต่อโมดูล, feature flag, deploy แบบ `app_offline` สั้น (docs/05 ข้อ 10)
- ไฟล์ส่วนกลาง (Platform, schema Core, `Directory.*.props`, CI) แก้ผ่าน task "platform" เท่านั้น (ADR-0008)
- ตาราง lab core เดิมใน DB `statwarning` ยังเป็นสัญญากับ LabWeb จนจบ R3 — ดู docs/04 ข้อ 5

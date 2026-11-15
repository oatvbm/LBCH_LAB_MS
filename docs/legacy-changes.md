# ทะเบียนการเปลี่ยนแปลงที่เกี่ยวกับระบบเดิม (Legacy changes)

> เริ่ม 2026-11-15 — ต้องอัปเดตใน PR เดียวกับงานที่ทำให้เกิดการเปลี่ยนแปลง (AGENTS.md ข้อ 3.8)

ไฟล์นี้มี 2 ส่วน

- **ส่วน A — การแก้ระบบเดิม/ระบบภายนอก**: ทุกครั้งที่ LAB_MS ต้องให้ระบบอื่นเปลี่ยน (LabWeb, LQMS, HPV, Lab_Stock, Lab_RMS, CAS, QIIS, ฐานข้อมูล `statwarning`/`labdatabase`, บัญชี DB, IIS) เพื่อให้ทีม ระบบเดิม และ DBA ตามเรื่องได้ที่เดียว
- **ส่วน B — ความต่างจากพฤติกรรมเดิมที่ตั้งใจ (known differences)**: จุดที่ LAB_MS ให้ผลต่างจากระบบเดิมโดยตั้งใจ (แก้ bug, เปลี่ยนนิยาม) ใช้อธิบายผล parity test / shadow run และให้หัวหน้างานรับรอง

## กติกา

1. **ระบบเดิมหยุดเพิ่มฟีเจอร์** (เอกสาร 05 ข้อ 9.1) — การแก้ระบบเดิมที่ยอมรับได้มีเพียง: แก้ bug, ความปลอดภัย, ลิงก์/redirect มายัง LAB_MS, และการเปลี่ยนที่จำเป็นต่อการเปลี่ยนผ่าน (contract, bridge, ปิดระบบ) — ถ้าเป็นอย่างอื่นต้องมี ADR
2. **ห้ามแก้ตาราง lab core เดิมใน `statwarning`** นอกเหนือจาก index (เอกสาร 04 ข้อ 3.2) และต้องทำผ่าน DBA หลังทดสอบบน staging
3. การแก้ระบบเดิมทำใน repo ของระบบนั้น (ไม่ใช่ใน repo นี้) — ที่นี่บันทึกเฉพาะทะเบียนและลิงก์
4. ห้ามใส่ credential, connection string หรือข้อมูลผู้ป่วยในไฟล์นี้
5. รหัสรายการไม่นำกลับมาใช้ซ้ำ รายการที่ยกเลิกให้เปลี่ยนสถานะเป็น `ยกเลิก` พร้อมเหตุผล

**สถานะ**: `วางแผน` → `ขออนุมัติ` → `กำลังทำ` → `ทดสอบ staging` → `ใช้งานแล้ว` (หรือ `ยกเลิก`)
**ประเภท**: `contract` (ข้อตกลงข้อมูล/API) · `link` (เมนู/redirect) · `security` · `bug` · `infra` · `retire` (ปิดระบบ/ส่วนของระบบ)

---

## ส่วน A — การแก้ระบบเดิม/ระบบภายนอก

| รหัส | ระบบ | การเปลี่ยนแปลง | ประเภท | ระยะ | อ้างอิง | ผู้รับผิดชอบ | สถานะ | วันที่ใช้งาน / PR |
|---|---|---|---|---|---|---|---|---|
| LC-001 | CAS (`LBCH_CAS`) | ลงทะเบียน AppCode `LAB_MS` + callback (localhost, staging, production — ตรงตัวพิมพ์ `/LAB_MS/cas/callback`) | contract | R0 | 05 ข้อ 1.1, 02 ข้อ 5.1 | tech lead + ผู้ดูแล CAS | วางแผน | |
| LC-002 | CAS | API client สำหรับ LAB_MS พร้อม scope `admin.user.read` (ค้นผู้ใช้ที่มีสิทธิ์ `LAB_MS`) | contract | R0 | Lab_Stock ADR-021 | ผู้ดูแล CAS | วางแผน | |
| LC-003 | CAS | อัปเกรด .NET 8 → .NET 10 (.NET 8 หมด support 10 พ.ย. 2026) | infra | คู่ขนาน R0–R1 (lane L5) | 05 ข้อ 5.1, 10 | ผู้ดูแล CAS | วางแผน | |
| LC-004 | MySQL 172.16.5.122 | สร้าง schema `labms_platform` และบัญชี DB แยก web/worker สิทธิ์เฉพาะ schema | infra | R0 | 05 ข้อ 4 | DBA | วางแผน | |
| LC-005 | QIIS (`kpi`) | เพิ่ม API `POST /api/indicators/{code}/periods/{yyyy-MM}/values` + scope `qiis.indicator.submit` ใน CAS | contract | คู่ขนาน R1 | 05 ข้อ 7 | lane L5 | วางแผน | |
| LC-006 | LabWeb (`labweb5`) | เพิ่มเมนูลิงก์ไป LAB_MS › Stat / Critical (CAS SSO) | link | R1 | 05 ข้อ 9.2 | | วางแผน | |
| LC-007 | DB `statwarning` | worker ใหม่เขียนตาราง lab core แทน StatImportAuto2 (ตาม contract เอกสาร 04 ข้อ 5.1) และปิด StatImportAuto2 + SW2 | retire | R1 | 03 Phase 7, 04 ข้อ 5 | tech lead | วางแผน | |
| LC-008 | DB `statwarning` | `LegacyCriticalBridge` อ่าน `criticalresult` ที่ LabWeb insert (Hemoculture) → `labms_critical.critical_result` (อ่านอย่างเดียว ไม่แก้ LabWeb) | contract | R1 | 04 ข้อ 5.2 | lane L3 | วางแผน | |
| LC-009 | DB `statwarning` | เพิ่ม index `requestdata(StatStatus, checkInStatus, CloseDate)`, `requesttest(labno, status)` | infra | R1 | 04 ข้อ 3.2 | DBA | วางแผน | |
| LC-010 | ทุกระบบที่ใช้บัญชี `lablbch` | เปลี่ยนรหัสผ่าน DB ทั้งหมดที่เคยฮาร์ดโค้ดใน SW2/StatImportAuto2/LabWeb และแยกบัญชีต่อระบบ | security | หลัง go-live R1 | 03 Phase 7, 04 ข้อ 5.4 | DBA | วางแผน | |
| LC-011 | Lab_Stock, Lab_RMS | ย้ายเป็นโมดูล, redirect URL เดิม, ย้ายผู้ใช้จาก AppCode `LABSTOCK`/`LRMS` → `LAB_MS` | retire + link | R2 | 05 ข้อ 8 | | วางแผน | |
| LC-012 | LabWeb | ค่าวิกฤต Hemoculture เรียก API ของ Critical โดยตรงแทน insert `criticalresult` แล้วเลิก bridge | contract | R3 | 04 ข้อ 5.2, 05 ข้อ 8 | | วางแผน | |
| LC-013 | LabWeb `Risk_record`, `KPI_view` | เลิกใช้ (ใช้ LRMS / QIIS แทน) | retire | R2–R4 | 05 ข้อ 2 | | วางแผน | |
| LC-014 | MySQL server | อัปเกรด 8.0.17 → 8.4 LTS (ทดสอบทุกระบบที่ใช้ server นี้) | infra | คู่ขนาน | 05 D10, 5.1 | DBA | วางแผน | |

> รายการ LC-001–LC-014 ตั้งจากเอกสาร 01–06 เพื่อให้เห็นภาพรวมตั้งแต่ต้น — เมื่อเริ่มทำจริงให้เติมผู้รับผิดชอบ วันที่ และลิงก์ PR/ticket

---

## ส่วน B — ความต่างจากพฤติกรรมเดิมที่ตั้งใจ (known differences)

| รหัส | โมดูล | พฤติกรรมเดิม | พฤติกรรมใหม่ | ผลต่อรายงาน/KPI | อ้างอิง | ผู้รับรอง | สถานะ |
|---|---|---|---|---|---|---|---|
| KD-001 | Stat | `completeRequestData` ไม่ reset ตัวแปร C/S ในลูป → Lab No ถัดไปในรอบเดียวกันได้ 60 นาทีทั้งหมด | กฎ C+S = 60 นาทีอยู่ใน `test_group_rule` คิดต่อ Lab No | `due_at` / is_late ของบางเคสต่างจากเดิม | 01 ข้อ 3.2.3, 02 ข้อ 4.1 | หัวหน้างาน | วางแผน |
| KD-002 | Stat | เวลาที่เหลือคิดจาก TimeOfDay ไม่ดูวันที่ | ใช้ `DATETIME` เต็ม | เคสข้ามเที่ยงคืนต่างจากเดิม | 01 ข้อ 3.2.1 | หัวหน้างาน | วางแผน |
| KD-003 | Critical | KPI T1–T5, T15 คิดด้วย `TIME` | คิดด้วย `DATETIME` เต็ม | KPI เคสข้ามเที่ยงคืนต่างจากเดิม | 01 ข้อ 3.2.2, 04 ข้อ 2.3 | หัวหน้างาน | วางแผน |
| KD-004 | Critical | ผลที่ LIS ส่งซ้ำ/แก้ไข insert ไม่ได้ (PK ชน) ค่าวิกฤตครั้งที่ 2 หาย | status 0 → อัปเดตแถวเดิม; status ≥ 1 → แถวใหม่ `supersedes_id`; KPI นับ `supersedes_id IS NULL` | จำนวนค่าวิกฤตอาจเพิ่ม | 04 ข้อ 3.1 | หัวหน้างาน | ต้องยืนยัน |
| KD-005 | Stat | `UpdateReportStatwarning` ล้าง comment/cause ทุกครั้งที่ปิดรายการ | เก็บสาเหตุใน `stat_case_cause` เป็นประวัติ ไม่ล้าง | รายงานสาเหตุครบขึ้น | 01 ข้อ 3.2.8 | หัวหน้างาน | วางแผน |
| KD-006 | Critical | QI31 ชื่อเมนู 10 นาที แต่ SQL ใช้ 25 นาที | ใช้ค่าที่หัวหน้างานยืนยัน (config) | อาจต่างจากรายงานเดิม | 01 ข้อ 2.9 | หัวหน้างาน | ต้องยืนยัน |
| KD-007 | Platform | ลงนามด้วยรหัสผ่านอย่างเดียว (หาผู้ใช้จาก `WHERE Password = ...`) | CAS login + (ถ้าเลือกแบบ B) เลือกชื่อ + PIN | ชื่อผู้ลงนามถูกคนขึ้น | 01 ข้อ 2.11, 02 ข้อ 5.3 | tech lead | รอตัดสินใจ |

> ทุกความต่างในส่วน B ต้องถูกอ้างถึงในรายงาน parity / shadow run และต้องมีผู้รับรองก่อน go-live

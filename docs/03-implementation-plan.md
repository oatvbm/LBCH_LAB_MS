# 03 — แผนการดำเนินงานพัฒนา StatWarning Web

> **อัปเดต 2026-11-15:** แผนนี้คือระยะ **R1** ของ LAB_MS ตาม [05-platform-roadmap.md](05-platform-roadmap.md) — Phase 1 (โครงสร้างพื้นฐาน) ในเอกสารนี้จะทำเป็น **R0 Platform kernel** ใน repo ใหม่ `LBCH_LAB_MS` (ยกโค้ด auth/สิทธิ์จาก Lab_Stock มาปรับ) และใช้ AppCode `LAB_MS` แทน `STATWARN`

> สมมติฐาน: นักพัฒนา 1–2 คน + ผู้ใช้หลักจากห้อง lab (key user) 1–2 คนช่วยทดสอบ
> ระยะเวลารวมโดยประมาณ **18–22 สัปดาห์** (ถ้ามีนักพัฒนา 2 คน ทำ Phase 3–5 คู่ขนานได้ จะลดเหลือประมาณ 14–16 สัปดาห์)

---

## ภาพรวม Phase

```text
สัปดาห์   1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22
P0 เตรียม  ██ ██
P1 ฐาน           ██ ██ ██
P2 Worker                 ██ ██ ██ ██            (shadow run ต่อเนื่องถึง go-live)
P3 Stat                               ██ ██ ██
P4 Critical                                    ██ ██ ██
P5 รายงาน                                               ██ ██ ██
P6 ตั้งค่า/Admin                                                  ██
P7 UAT/Go-live                                                       ██ ██ ██ ██
```

---

## Phase 0 — เตรียมการและตัดสินใจ (สัปดาห์ 1–2)

| งาน | ผลลัพธ์ |
|---|---|
| ~~ขอ schema dump~~ ✅ ได้แล้ว → วิเคราะห์ใน [04-database-design.md](04-database-design.md) | |
| ขอข้อมูล master (testlist, testresultlist, testgroupname, causeid2, clinic) + รัน query สำรวจ Q1–Q8 ในเอกสาร 04 | ผลลัพธ์ query |
| ~~ระบุระบบอื่นที่ใช้ DB ร่วมกัน~~ ✅ คือ LabWeb → data contract อยู่ในเอกสาร 04 ข้อ 5 | |
| ยืนยันว่าค่าวิกฤต Hemoculture จาก LabWeb (notifyStatus = NULL) ปัจจุบันแจ้งหอผู้ป่วยผ่านช่องทางไหน | กติกาสำหรับ LegacyCriticalBridge |
| เก็บไฟล์ HL7 ตัวอย่าง order และ result (ปิดชื่อผู้ป่วย/CID) ≥ 50 ไฟล์ รวมกรณีพิเศษ (C+S, outlab, micro, critical, ข้ามเที่ยงคืน) | `tests/fixtures/hl7/` |
| ขอ export KPI/สถิติย้อนหลัง 12 เดือนจากระบบเดิม | golden dataset สำหรับ parity test |
| ลงทะเบียน AppCode `STATWARN`, callback (prod + staging), permissions ตามเอกสาร 02 ข้อ 5.2 กับผู้ดูแล CAS | ค่า config CAS |
| ขอ DB user ใหม่: app user ของ `statwarning_v2`, read-only user ของ ABS และ HIS | credential ใน secret store |
| จัดเตรียม server IIS + service account ที่เข้าถึง shared folder HL7 | server พร้อม |
| **ตัดสินใจ**: (1) stack (2) วิธีลงนามบนเครื่องร่วม A/B (3) เกณฑ์ QI31 คือ 10 หรือ 25 นาที (4) สิทธิ์และการ login ของจอ TV/หอผู้ป่วย | บันทึกการตัดสินใจ (ADR) |

**เกณฑ์ผ่าน Phase**: มี fixtures ครบ, ได้ค่า CAS, ได้ตัดสินใจครบทั้ง 4 ข้อ

---

## Phase 1 — โครงสร้างพื้นฐาน (สัปดาห์ 3–5)

- สร้าง solution ตามเอกสาร 02 ข้อ 2.2, ตั้ง CI (build + test), Serilog, health checks
- EF Core DbContext + migration แรกของ schema ใหม่ทั้งหมด
- CAS: login → callback → exchange → profile → local cookie, heartbeat `/auth/session-status`, global logout, canonical path middleware, permission policy
- Layout หลัก (เมนูตาม sitemap, แสดงชื่อผู้ใช้, แถบสถานะการเชื่อมต่อ), หน้า 401/403/session หมดอายุ ตามข้อความมาตรฐาน CAS
- `audit_log` + interceptor
- SignalR hub + DashboardBroadcaster (อ่าน outbox) — ทำเป็น skeleton ไว้ก่อน

**เกณฑ์ผ่าน**: login/logout ผ่าน CAS ใน staging ได้ทุก scenario ใน CAS checklist ข้อ 14 และข้อ 23 (เปิดหลายระบบ, logout จากอีกระบบแล้วถูกเตะออกภายใน 1 นาที)

---

## Phase 2 — Worker / Ingestion (สัปดาห์ 6–9)

| ลำดับ | งาน |
|---|---|
| 2.1 | `Hl7Parser` (ORM/ORU, cp874) + unit test จาก fixtures |
| 2.2 | OrderIngestor: upsert patient/clinic/doctor/sample/order/test **ลงตาราง lab core เดิมในรูปแบบเดียวกับ StatImportAuto2 ทุกคอลัมน์** (รวม `outlab`, `statusTestGroup`, `checkInStatus`), lookup CID จาก HIS, idempotent, ย้ายไฟล์ไป backup/error, `import_file_log` |
| 2.3 | Test grouping + turnaround (`test_group_rule`) — แก้ bug C+S |
| 2.4 | CheckInSync จาก ABS + กติกา `skip_abs_checkin` + nightly job 23:30 |
| 2.5 | `TurnaroundCalculator` + `service_hour_rule` + unit test ครบทุกกรณีในเอกสาร 01 ข้อ 2.4 และกรณีข้ามเที่ยงคืน |
| 2.6 | StatCaseEngine: สร้าง case, ปิด case เมื่อผลครบ, is_late |
| 2.7 | ResultIngestor: อัปเดตผล, tech/med validate, ตรวจจับค่าวิกฤต (TPH/TPL + `is_critical_monitored`) |
| 2.8 | Outbox events: `StatCaseCreated/Reported`, `CriticalDetected` |
| 2.8b | `LegacyCriticalBridge`: อ่านค่าวิกฤต Hemoculture ที่ LabWeb insert ลง `criticalresult` เดิม → `critical_result` (source `LABWEB`) |
| 2.8c | Integration test กับ LabWeb บน staging: เปิดหน้า Hemoculture / ResultBac / outlab_record แล้วข้อมูลที่ worker ใหม่เขียนต้องแสดงและบันทึกได้ตามปกติ |
| 2.9 | **Shadow run**: ให้ worker อ่านไฟล์ชุดเดียวกับระบบเดิม (copy ไฟล์ไปอีกโฟลเดอร์) และเขียนลง **สำเนาของ schema lab core** (เช่น `statwarning_shadow`) ไม่ใช่ตารางจริง แล้วรายงานเปรียบเทียบรายวัน: (ก) ทุกคอลัมน์ของ requestdata/requesttest/patient/outlab (ข) due_at, is_late, critical ที่ตรวจพบ |

**เกณฑ์ผ่าน**: shadow run 1 สัปดาห์: ตาราง lab core ตรงกัน 100% (ยกเว้นจุดที่ตั้งใจแก้และระบบอื่นรับรองแล้ว) และผล stat/critical ตรงกับระบบเดิม ≥ 99% โดยทุกความต่างอธิบายได้

> ห้ามให้ worker ใหม่ย้ายไฟล์จากโฟลเดอร์ของระบบเดิมระหว่าง shadow run — ต้องใช้สำเนาไฟล์เท่านั้น

---

## Phase 3 — Stat Dashboard (สัปดาห์ 10–12)

- Dashboard: รายการรอผลแบบนับถอยหลังฝั่ง client, สีตามระดับ, FastTrack ปักบนสุด, รายการรายงานแล้ววันนี้, เสียงเตือน (ตั้งค่าได้ต่อเครื่อง)
- Push real-time ผ่าน SignalR, reconnect อัตโนมัติ
- บันทึกสาเหตุล่าช้า (dropdown ลำดับชั้น 4 ระดับ) + ลงนามตามแบบที่เลือกใน Phase 0
- รายละเอียดใบ Request + ลบใบ/ลบ test (ใช้สิทธิ์ `ORDER.DELETE` และบันทึก audit)
- โหมด TV `/display`
- ค้นหา: ประกันเวลา / สาเหตุ / FastTrack / ผู้ป่วย + export

**เกณฑ์ผ่าน**: key user ใช้ dashboard คู่กับ SW2 ใน staging ได้ 3 วันโดยไม่มีรายการที่หลุด

---

## Phase 4 — Workflow ค่าวิกฤต (สัปดาห์ 13–15)

- คิว status 0 + แถบแจ้งเตือนทุกหน้า + ฟอร์มแจ้งหอผู้ป่วย (เลือกผู้รับจาก `ward_recipient` หรือพิมพ์ใหม่)
- เติมข้อมูลเวลา (analyzed/screened/actions) แบบหลายแถวพร้อมกัน
- หน้าหอผู้ป่วย: บันทึกเวลาแจ้งแพทย์และการรักษา → ปิด (status 3)
- บันทึกค่าวิกฤตจุลชีวะด้วยมือ (ทำให้ใช้งานได้จริง)
- ค้นหา/แก้ไข (สิทธิ์ `CRITICAL.EDIT`) + timeline T0–T5 ต่อรายการ

**เกณฑ์ผ่าน**: ทดสอบ end-to-end ครบ 0→3 ทุกงาน (เคมี, โลหิต, จุลชีวะ) และ timeline มีวันเวลาเต็มทุกจุด

---

## Phase 5 — รายงานและ KPI (สัปดาห์ 16–18)

- ทำ KPI เป็น query/service ชุดเดียวที่ใช้ร่วมกัน (ไม่ทำซ้ำแบบ QI28/QI28_1): T1–T5, T15, KPI9, QI28–31 รายเดือนตามปีงบประมาณ + กราฟ
- รายงานค่าวิกฤต: แยกตามงาน / test / หน่วยงาน / ใน-นอกเวลา / ประเภทผู้ป่วย / Blood smear
- สถิติ Stat, Routine, จุลชีวะ, ห้องเจาะเลือด, OneTechMed
- Export .xlsx (ClosedXML) ทุกรายงาน
- **Parity test**: รันกับข้อมูลที่ migrate มา เทียบกับ golden dataset จาก Phase 0

**เกณฑ์ผ่าน**: KPI 12 เดือนย้อนหลังตรงกับระบบเดิม หรือความต่างมีเหตุผลรองรับเป็นลายลักษณ์อักษร (เช่น เคสข้ามเที่ยงคืน)

---

## Phase 6 — ตั้งค่าและ Admin (สัปดาห์ 19)

- CRUD: Test catalog, Test group rule, Clinic + ผู้รับแจ้ง, Delay cause, Service hour rule, Patient status flags
- หน้าผู้ใช้: map LIS user code, ตั้ง/รีเซ็ต PIN
- หน้าสถานะระบบ: import log, health, audit log

---

## Phase 7 — UAT, Migration, Go-live (สัปดาห์ 19–22)

| ขั้น | รายละเอียด |
|---|---|
| UAT | ทดสอบตาม scenario กับ key user ทุกงาน + หอผู้ป่วยนำร่อง 1–2 แห่ง |
| Dry-run migration | รัน ETL บน staging, reconcile, จับเวลาที่ใช้ |
| อบรม | คู่มือสั้น 1–2 หน้า/บทบาท (เจ้าหน้าที่ lab, หัวหน้างาน, หอผู้ป่วย, admin) |
| Parallel run | 1–2 สัปดาห์ ระบบเดิมยังทำงานหลัก ระบบใหม่ทำงานคู่และเทียบผลรายวัน |
| Cut-over | แจ้งผู้ใช้ LabWeb (งานจุลชีวะ/Outlab) ล่วงหน้า → หยุด StatImportAuto2 → migrate delta ของข้อมูล StatWarning → เปิด worker ใหม่อ่านโฟลเดอร์จริงและเขียน lab core → ตรวจว่าระบบอื่นยังได้ข้อมูลใหม่ → เปิดเว็บ |
| Rollback plan | เก็บ SW2/StatImportAuto2 ไว้ 1 เดือน; ถ้าพบปัญหาร้ายแรง ให้หยุด worker ใหม่แล้วเปิด importer เดิม (ไฟล์ยังอยู่ใน backup) |
| หลัง go-live | **เปลี่ยนรหัสผ่าน DB ทั้งหมดที่เคยฮาร์ดโค้ดใน binary เดิม**, ถอน SW2 ออกจากทุกเครื่อง, archive DB เดิมแบบ read-only |

---

## ความเสี่ยงและวิธีรับมือ

| ความเสี่ยง | ผลกระทบ | วิธีรับมือ |
|---|---|---|
| ไม่มีไฟล์ HL7 ตัวอย่างจริง | parser หรือ ETL ผิด | ทำ Phase 0 ให้เสร็จก่อนเขียนโค้ด และใช้ shadow run |
| **DB `statwarning` ใช้ร่วมกับ LabWeb** (จุลชีวะ/Hemoculture/Outlab) | worker ใหม่เขียน lab core ผิดเพี้ยน → LabWeb พัง; ค่าวิกฤต Hemoculture หลุดจากระบบใหม่ | ยึด data contract ในเอกสาร 04 ข้อ 5, parity test ระดับคอลัมน์, shadow run ลง schema สำเนา, integration test กับ LabWeb (2.8c), LegacyCriticalBridge |
| MySQL 8.0.17 หมด support แล้ว | ช่องโหว่ไม่ได้รับ patch | เสนออัปเกรดเป็น 8.4 LTS เป็นงานแยกร่วมกับ DBA (ทดสอบระบบอื่นด้วย) — ไม่บล็อกโครงการนี้ |
| collation ปนกัน (utf8 กับ utf8mb4) | query join ช้าหรือ error | schema ใหม่ใช้ utf8mb4 ทั้งหมด; query ที่ join ตารางเก่าที่เป็น utf8 ให้ระบุ `COLLATE` ชัดเจนและวัด performance |
| KPI ไม่ตรงกับรายงานเดิมที่ส่งผู้บริหารไปแล้ว | ไม่เชื่อถือระบบใหม่ | parity test + บันทึกความต่างที่เกิดจาก bug เดิม + ให้หัวหน้างานรับรอง |
| CAS ใช้งานไม่ได้ | login ไม่ได้ | dashboard ที่ login ค้างอยู่ยังใช้ได้ (นโยบาย availability ตาม CAS ข้อ 6.3), มี break-glass local admin 1 บัญชีที่ปิดไว้ปกติ |
| จอ lab เปิดค้าง 24 ชม. แล้ว session หมดอายุ | dashboard ดับกลางเวร | โหมด display ใช้สิทธิ์ `DISPLAY` + sliding session + reconnect; ทดสอบเปิดค้าง 72 ชม. |
| Shared folder/เครือข่ายขัดข้อง | ข้อมูลไม่เข้า | health check + แจ้งเตือนบน dashboard เมื่อไม่มีไฟล์เข้าเกินกำหนด |
| ผู้ใช้ไม่คุ้นกับการลงนามแบบใหม่ | ใช้งานช้าลง | ออกแบบให้ใช้ keyboard ได้ทั้งหมด (เลือกชื่อ + PIN + Enter) และทดสอบกับ key user ตั้งแต่ Phase 3 |

---

## สิ่งที่ต้องตัดสินใจ / ขอข้อมูลจากผู้ใช้

1. **Stack**: ยืนยันใช้ ASP.NET Core (.NET 10) + Razor Pages/htmx + MySQL ตามที่เสนอหรือไม่
2. **การลงนาม**: แบบ A (login รายคน) หรือ B (station + PIN — แนะนำ)
3. **QI31**: เกณฑ์ที่ถูกต้องคือ 10 หรือ 25 นาที
4. **จอ TV/หอผู้ป่วย**: หอผู้ป่วยต้อง login CAS รายคน หรือใช้บัญชีประจำหอ
5. ต้องเก็บ CID ในระบบนี้ต่อไปหรือไม่ (ถ้าไม่ใช้ ควรเลิกดึงจาก HIS)
6. Server สำหรับ staging/production และชื่อ host ที่จะใช้
7. ~~ระบบไหนใช้ DB ร่วมกัน~~ → LabWeb; ยืนยันด้วย query Q1 ว่าไม่มีระบบอื่นอีก
8. กติกาค่าวิกฤตที่ LIS ส่งซ้ำหรือแก้ผลหลังแจ้งไปแล้ว (เอกสาร 04 ข้อ 3.1)
9. ค่าวิกฤต Hemoculture จาก LabWeb: ใช้ bridge อ่านตารางเดิมไปก่อน แล้วค่อยแก้ LabWeb ให้เรียก API ของ StatWarning (เอกสาร 04 ข้อ 5.2) — ตกลงไหม
10. ยืนยันว่าจะแยกเป็นแอป ASP.NET Core (แนะนำ) ไม่ใช่เพิ่มเป็นเมนูใน LabWeb

---

## ขั้นตอนถัดไปทันที

1. รัน query Q1–Q8 ในเอกสาร 04
2. ขอไฟล์ HL7 ตัวอย่างและข้อมูล master (Phase 0)
3. ส่งคำขอลงทะเบียน AppCode `STATWARN` กับผู้ดูแล CAS
4. ตอบคำถาม 10 ข้อข้างบน → จากนั้นเริ่ม scaffold solution (Phase 1) ได้ทันที

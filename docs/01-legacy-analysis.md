# 01 — วิเคราะห์ระบบ StatWarning เดิม (SW2 + StatImportAuto2)

> แหล่งที่อ่าน (อยู่ที่ `C:\programing\StatWarning on web\RawData`, ไม่ได้คัดลอกเข้า repo เพราะมีรหัสผ่าน DB และไฟล์ binary): `RawData/SW2` (client WinForms), `RawData/StatImportAuto2` (ตัวนำเข้าข้อมูล), `RawData/LBCH_CAS_Integration_Protocol_for_Developers.md`, `RawData/KPI_Logout_Integration_Guide_TH.md`
> วันที่วิเคราะห์: 2026-11-15

---

## 1. ภาพรวมองค์ประกอบ

| องค์ประกอบ | เทคโนโลยี | หน้าที่ |
|---|---|---|
| **SW2** (v1.0.3) | VB.NET WinForms, .NET Framework 4.7.2, MySql.Data, ClosedXML + Excel Interop, ติดตั้งด้วย Inno Setup ทีละเครื่อง | หน้าจอเจ้าหน้าที่: ติดตาม Stat แบบ real-time, บันทึกสาเหตุล่าช้า, workflow รายงานค่าวิกฤต, สถิติ/KPI, ตั้งค่า |
| **StatImportAuto2** | VB.NET WinForms (ต้องเปิดค้างไว้บนเครื่อง PC), Timer 15 วินาที | อ่านไฟล์ HL7 จาก HIS → เขียนลง MySQL, ดึงเวลา check-in จาก ABS, สร้างรายการ StatWarning, ตรวจผลเสร็จ, ตรวจจับค่าวิกฤต |
| **MySQL `Statwarning`** | 172.16.5.122:3306 | ฐานข้อมูลหลัก |
| **ABS** (`abs_3_1`, MySQL 172.17.6.86) | ระบบคิวห้องเจาะเลือด | ตาราง `Tat_abs`: checkIn / call / confirm / close datetime, userid |
| **Softcon HIS** (`LbchPhoenixDB`, SQL Server 172.16.5.111) | HIS | อ่าน CID และวันเกิดของผู้ป่วยจาก HN |
| **Shared folder HL7** | ตั้งค่าในตาราง `Filedirectory` | ไฟล์ order (ORM) และไฟล์ผล (ORU) เข้ารหัส TIS-620 (code page 874) |

### Data flow ปัจจุบัน

```text
HIS/LIS ──(ไฟล์ HL7 order)──▶ StatImportAuto2.ImportRequest ──▶ Patient / Clinic / Doctor / Sample / RequestData / RequestTest
                                     │ (lookup CID, BirthDT) ◀── Softcon HIS (SQL Server)
                                     ▼
                              completeRequestData  ──▶ RequestData.testShotNameAll / testGroup / timeToReport
                                     ▼
ABS Tat_abs ───────────────▶ RecordReceiveSample ──▶ RequestData.CheckIn/Call/Confirm/CloseDate
                                     ▼
                              addstatwaning        ──▶ StatWarning (status 0 = รอผล)
LIS ──(ไฟล์ HL7 result)──▶ ImportReport          ──▶ RequestTest.status=1, ReportDateTime, techBy, reportBy
                                     │ OBX flag TPH/TPL ──▶ criticalresult (notifyStatus 0)
                                     ▼
                              UpdateReportStatwarning ──▶ StatWarning.status=1, ReportInTime (0 ทัน / 1 ไม่ทัน)
                              updateRequestAllReport  ──▶ RequestData.ReportDateTime
                     (23:30 ทุกวัน) updateCheckInDateLost ──▶ เติม check-in = RegisterDate ที่ยังว่าง

SW2 (ทุกเครื่อง) ── poll ทุก 30 วินาที / 3 นาที ──▶ MySQL ──▶ ตาราง Stat, popup เตือน, popup ค่าวิกฤต
```

---

## 2. Business rules ที่ต้องคงไว้ (สกัดจากโค้ด)

### 2.1 การนำเข้า Order (HL7)

| Segment | Field | ความหมาย |
|---|---|---|
| PID-2 | HN |
| PID-5 | ชื่อ `คำนำหน้า^ชื่อ^^นามสกุล` |
| PID-8 | เพศ F/M |
| PV1-2 | OP → OPD, IP → IPD |
| PV1-9 / PV1-11 | แพทย์ `id^...` / คลินิก `id^name` |
| PV1-29 | PatientStatus (เช่น รถนั่ง on oxygen, เจาะเลือดจากหอผู้ป่วย) |
| ORC-4 | Lab No |
| ORC-7 | วันเวลาสั่ง `yyyyMMddHHmmss` |
| ORC-12 | แพทย์ผู้สั่ง |
| ORC-20 | `1` = **ด่วน (Stat)** |
| OBR-4 | `testID^testName` |
| OBR-15 | sample `id^name` |
| OBR-34 / OBR-35 | register lab / `lesion^site^side` |

- คลินิก แพทย์ ประเภทผู้ป่วย และ sample ที่ยังไม่มี จะถูกเพิ่มลง master อัตโนมัติ
- ไฟล์ที่อ่านแล้วถูกย้ายไป `backup data\{yyyyMMdd}\`

### 2.2 การจัดกลุ่มรายการตรวจและเวลาประกัน (`completeRequestData`)

- เอา `testShotName` ของทุก test ใน Lab No มาเรียงต่อกัน เช่น `CH` แล้วหา `TestGroup` จากตาราง `testGroupName.testShotNameAll`; ถ้าไม่พบ ให้เป็น `R`
- `timeToReport` = ค่า `testTime` สูงสุดของ test ในใบนั้น (นาที)
- มีทั้ง `C` และ `S` → `timeToReport = 60` นาที
- Test group `O` = ส่งตรวจภายนอก (outlab) → บันทึกตาราง `Outlab`

### 2.3 การรับตัวอย่าง (`RecordReceiveSample`)

- ดึงเวลาจาก ABS `Tat_abs` ด้วย `LISLabNumber`
- ถ้าไม่พบใน ABS และเป็นกรณี "เจาะเลือดจากหอผู้ป่วย", "เจาะเลือดมาเอง/เก็บตัวอย่างมาเอง", "ไม่ระบุ", "เก็บตัวอย่างระบบทางเดินหายใจ", "เพิ่ม LAB" หรือกลุ่ม `UF`/`I` → ใช้ `RegisterDate` แทนเวลาทุกจุด
- เวลา 23:30 ของทุกวัน: เติม check-in = RegisterDate ให้รายการที่ยังว่าง

### 2.4 การสร้างรายการ Stat (`addstatwaning`)

เงื่อนไข: `StatStatus='ด่วน'` และ testGroup ไม่ใช่ `O`/`I` และ check-in แล้ว และ `closeDate` ไม่ว่าง และยังไม่อยู่ใน StatWarning

- **เวลาเริ่มนับ** = `closeDate` (เวลาที่ห้องเจาะเลือดปิดคิว)
- **ปรับตามช่วงเวลาให้บริการ** (ฮาร์ดโค้ด):
  - เริ่มระหว่าง 06:00–08:00 → เริ่มนับที่ 08:00
  - เริ่มระหว่าง 12:00–13:00 → เริ่มนับที่ 13:00
  - เวลาครบกำหนดตกในช่วง 12:00–13:00 → บวกเพิ่ม 60 นาที
- `ToReportDateTime` = เวลาเริ่มนับ + `timeToReport`
- **FastTrack** = PatientStatus เป็น "รถนั่ง (on oxygen)" หรือ "เปลนอน (on oxygen)"

### 2.5 การปิดรายการ Stat (`UpdateReportStatwarning`)

- เมื่อ RequestTest ทุกตัวของ Lab No มี `status=1` → StatWarning `status=1`, `ReportDateTime` = เวลาผลตัวสุดท้าย, ผู้รายงาน = ชื่อจาก `ReportUser` ที่ map กับ `reportBy`
- `ReportInTime = 1` (ไม่ทัน) ถ้า ReportDateTime > ToReportDateTime

### 2.6 การเตือนบนหน้าจอ (SW2 `ShowData`)

- โหลดตารางทุก 30 วินาที และเช็ค popup ทุก 3 นาที
- สีของแถว: เกินเวลา = ชมพู; เหลือ < 5 นาที = ทอง; < 10 นาที = Khaki; < 15 นาที = LemonChiffon; FastTrack = ตัวอักษรแดงและอยู่บนสุด
- Popup `WarningOut`: รายการเกิน/ใกล้เกินเวลา
- Popup `warningAddcause`: รายการไม่ทันเวลาที่ยังไม่บันทึกสาเหตุ
- Popup `warningRecordCritical`: ค่าวิกฤตที่ยังไม่แจ้ง

### 2.7 การบันทึกสาเหตุล่าช้า (`EditComment2`)

- ใช้ catalog `causeid2` แบบลำดับชั้น: **งาน (jobsection) → กลุ่มสาเหตุ → รายละเอียด → ประเภท (ควบคุมได้/ควบคุมไม่ได้)**
- บันทึกใน StatWarning: `comment, causeType, causeGroup, causeDetail, causeDiv, causeBy`

### 2.8 Workflow ค่าวิกฤต (`criticalresult.notifyStatus`)

```text
[0] ตรวจพบ ──แจ้งหอผู้ป่วย──▶ [1] แจ้งแล้ว ──เติมข้อมูลเวลา──▶ [2] รอบันทึกการแจ้งแพทย์ ──▶ [3] ปิดเรื่อง
     (Import OBX TPH/TPL        (RecordCritical:            (RecordCriticalCompData:       (RecordNotifyDoctor:
      และ test ไม่อยู่ใน          ผู้รับแจ้งจาก                  analyteDT, screenDT,             notifyDoctor_time,
      NonCriticalTest)            critical_emp_receive,         screenAction, doctorAction)      doctorAction หรือปิดตรง)
                                  notify_time, comment)
```

- ค่าวิกฤตจุลชีวะ (MB010–MB012) บันทึกเองผ่าน `recordCriticalBac` แต่โค้ดยังไม่เสร็จ
- การแก้ไขข้อมูลค่าวิกฤต (`EditCriData`) ต้องใช้สิทธิ์ `critical_status = 1`

### 2.9 นิยาม KPI ค่าวิกฤต (ต้องให้ผลตรงกับของเดิม)

| ตัวชี้วัด | นิยาม |
|---|---|
| T1 | analyteDT ≤ RecieveSampleTime + 15 นาที |
| T2 | screenDT ≤ analyteDT + 20 นาที |
| T3 | tval_time ≤ screenDT + 2 นาที |
| T4 | mval_time ≤ tval_time + 2 นาที |
| T5 / KPI9 | notify_time ≤ mval_time + 3 นาที (รายงานค่าวิกฤตทันเวลา 3 นาที) |
| T15 | notify_time ≤ RecieveSampleTime + 40 นาที |
| QI29 | เคมีคลินิก: mval − screenDT < 5 นาที |
| QI30 | โลหิตวิทยา: mval − screenDT < 25 นาที |
| QI31 | จุลชีววิทยา: mval − screenDT < 25 นาที (ชื่อเมนูเขียนว่า 10 นาที แต่ SQL ใช้ 25 นาที — **ต้องยืนยัน**) |
| QI28 | รวมทุกงาน (ตามเกณฑ์ของแต่ละงาน) |

- รายงานแบ่งตาม **ปีงบประมาณ** (เริ่มเดือนตุลาคม) แยกรายเดือน
- รายงานอื่น: แยกตามงาน / รายการทดสอบ / หน่วยงาน / ใน-นอกเวลาราชการ / ประเภทผู้ป่วย / อัตราการดู Blood smear

### 2.10 สถิติและการค้นหาอื่น ๆ

- จำนวนผู้ป่วย Stat แยกตามวันที่/แผนก, Stat ทันเวลา/ไม่ทันเวลา แยกตามงานหรือรายการตรวจ
- Routine: ผู้รับบริการแยกตามแผนก (OPD/IPD) / งานบริการ, ห้องเจาะเลือดที่ไม่มีรายการเจาะ
- จุลชีวะ: แยกตามแผนก, specimen, specimen c/s, รายการตรวจ
- ห้องเจาะเลือด: แยกตามประเภทผู้ป่วย
- **OneTechMed**: ผลที่ tech validate และ med validate โดยคนเดียวกัน (สำหรับตรวจสอบคุณภาพ)
- การค้นหา: ข้อมูลประกันเวลา, เหตุผลที่เกินเวลา, เหตุผลทั้งหมด, Fast track, ค้นผู้ป่วย
- Export Excel ได้ทุกหน้า

### 2.11 ผู้ใช้และสิทธิ์ (`ReportUser`)

| Field | ใช้ทำอะไร |
|---|---|
| Username / Password | ใช้ login (เก็บ password แบบ plain text) |
| Name | ชื่อที่บันทึกเป็นผู้ทำรายการ |
| Accessrights = 1 | ลบใบ request หรือ test ได้ |
| SeniorStatus | อ่านค่ามาแต่ไม่พบว่าใช้ |
| critical_status = 1 | แก้ไขหรือปิดข้อมูลค่าวิกฤตได้ |
| ABS_userid | map ผู้ใช้กับ ABS |
| UseStatus | เปิด/ปิดบัญชี |

**รูปแบบ "ลงนามด้วยรหัสผ่าน"**: ทุก action สำคัญจะเปิดหน้าต่าง `password` ให้กรอก **รหัสผ่านอย่างเดียว** แล้วหาผู้ใช้จาก `WHERE Password = ...` เพื่อระบุตัวคนทำ (รองรับหลายคนใช้เครื่องเดียวกันใน lab)

---

## 3. ปัญหาของระบบเดิม

### 3.1 ความปลอดภัย (ร้ายแรง)

1. **Credential ฮาร์ดโค้ดในโค้ดและ binary** ทั้ง 3 ฐานข้อมูล รวมถึง `root` ของ ABS และ `LbchDBRead` ของ HIS → **ควรเปลี่ยนรหัสผ่านฐานข้อมูลทั้งหมดหลังย้ายระบบ** เพราะ binary กระจายอยู่ทุกเครื่อง
2. รหัสผ่านผู้ใช้เก็บแบบ plain text
3. **SQL Injection** อยู่ทั่วไป (login, ลบใบ request, บันทึกค่าวิกฤต)
4. การระบุตัวคนด้วยรหัสผ่านอย่างเดียว → ถ้าสองคนตั้งรหัสซ้ำกัน ระบบจะบันทึกชื่อผิดคน และเดารหัสของคนอื่นได้ง่าย
5. ตรวจสิทธิ์แค่ซ่อนปุ่มบน client
6. มี CID ผู้ป่วยในตาราง Patient แต่ไม่มีการควบคุมการเข้าถึงหรือ audit (เรื่อง PDPA)

### 3.2 ความถูกต้องของข้อมูล (bug)

1. **การคำนวณเวลาที่เหลือใช้แค่เวลาของวัน (TimeOfDay) ไม่ดูวันที่** (`ShowData.addDataStat/checkTime`) → ผิดเมื่อข้ามเที่ยงคืน
2. `criticalresult` เก็บวันที่และเวลาแยกคอลัมน์ (`RecieveSampleTime`, `mval_time`, `notify_time` ...) และ KPI ใช้ `TIMEDIFF/ADDTIME` บน TIME → **KPI ผิดเมื่อข้ามเที่ยงคืน**
3. `completeRequestData`: ไม่ reset ตัวแปร `tmpCheckC/tmpCheckS` ในลูป → ถ้ามี Lab No ใดมีทั้ง C+S แล้ว **Lab No ถัดไปในรอบเดียวกันจะได้ 60 นาทีทั้งหมด**
4. `recordCriticalBac`: query ไม่ join `patient/testresultlist` (เป็น cartesian product) และไม่ใช้ HN ที่กรอก
5. ตัวแปร global (`tmpSendCriticalHN`, `...Result`, `...Value`) ไม่ถูก clear → list โตไปเรื่อย ๆ และข้อมูลใน popup เลื่อนไม่ตรงแถว
6. `RecordCritical` บันทึก `notify_time` เป็นเวลาอย่างเดียว ไม่มีวันที่
7. สถานะเป็นข้อความภาษาไทย (`'ด่วน'`, PatientStatus) ทำให้การเทียบค่าเปราะ
8. `UpdateReportStatwarning` ล้าง comment/cause ทุกครั้งที่ปิดรายการ

### 3.3 การใช้งานและการดูแลระบบ

1. Importer เป็นโปรแกรม desktop ที่ต้องมีคน login ค้างไว้ และ `MessageBox` ใน background task จะค้างรอคนกด OK → การนำเข้าหยุด
2. ทุกเครื่องที่เปิด SW2 จะ query DB ทุก 30 วินาที (โหลดเพิ่มตามจำนวนเครื่อง)
3. ต้องติดตั้งและอัปเดตทีละเครื่อง และต้องมี Microsoft Office เพราะใช้ Excel Interop
4. Popup หลายหน้าต่างแบบ TopMost รบกวนการทำงาน
5. Logic ซ้ำกันมาก (`QI28` กับ `QI28_1` ...), `StatCritical.vb` ยาว 4,256 บรรทัด และใช้ global state ข้ามฟอร์ม
6. ไม่มี log กลาง ไม่มี health check ไม่มี audit trail ไม่มี test

---

## 4. สิ่งที่ยังไม่มีใน RawData (ต้องขอเพิ่ม)

1. ~~Schema จริงของฐานข้อมูล~~ — **ได้แล้ว** ([`legacy-schema/statwarning_schema.sql`](legacy-schema/statwarning_schema.sql)) ดูผลวิเคราะห์ใน [04-database-design.md](04-database-design.md) ซึ่งพบว่า **DB นี้ใช้ร่วมกับระบบอื่น** (จุลชีวะ/Hemoculture/Outlab)
   - ยังต้องขอข้อมูล master (testlist, testresultlist, testgroupname, causeid2, clinic) และผลของ query สำรวจในเอกสาร 04 ข้อ 6
   - ~~ระบบอื่นที่ใช้ DB นี้~~ — **คือ LabWeb** (`C:\programing\labweb5`) วิเคราะห์ data contract แล้วในเอกสาร 04 ข้อ 5
2. **ไฟล์ HL7 ตัวอย่าง** ทั้ง order และ result (ปิดข้อมูลผู้ป่วยแล้ว) อย่างน้อย 30–50 ไฟล์ที่ครอบคลุมกรณีพิเศษ
3. โครงสร้างตาราง `Tat_abs` ของ ABS และสิทธิ์อ่านแบบ read-only
4. รายงาน KPI ย้อนหลังที่ออกจากระบบเดิม เพื่อใช้เทียบผล (golden dataset)
5. รายชื่อผู้ใช้และการ map `ReportUser.Username` (รหัส LIS ใน OBX-16) ↔ `empKey` ใน CAS

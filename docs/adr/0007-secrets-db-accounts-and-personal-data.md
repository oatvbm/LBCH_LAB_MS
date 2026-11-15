# ADR-0007: Secrets, บัญชี DB และข้อมูลส่วนบุคคล

- สถานะ: ตัดสินแล้ว (2026-11-15)
- ผู้ตัดสิน: tech lead
- ที่มา: docs/05 ข้อ 1.0, 4; docs/06 ข้อ 6; docs/02 ข้อ 7 (PDPA); docs/04 ข้อ 2 (#9), 5.4; docs/01 ข้อ 3.1; Lab_Stock ADR-010, ADR-019

## บริบท

- ระบบเดิมฝังรหัสผ่าน DB ใน binary/`Web.config` และใช้บัญชี `lablbch` เดียวที่เข้าถึงได้ทุก database (docs/01 ข้อ 3.1, docs/04 ข้อ 5.4)
- ฐานข้อมูลมี CID ผู้ป่วยและ CID เจ้าหน้าที่โดยไม่มีการควบคุม (PDPA)
- ทีมพัฒนาใช้ AI agent ซึ่งอ่านไฟล์ใน repo และอาจส่งเนื้อหาไปประมวลผลบน cloud

## การตัดสินใจ

### Secrets

1. **ห้ามมี secret ใน Git** — connection string ที่มีรหัสผ่าน, ค่าลับของ CAS/API client, encryption key, SMTP credential
2. development ใช้ **`dotnet user-secrets`**; staging/production ใช้ **environment variable** ของ app pool/service (หรือ secret store ที่ฝ่าย IT กำหนด)
3. `appsettings*.json` ใน repo มีได้เฉพาะ key เปล่าหรือ placeholder; deploy **ห้ามเขียนทับ config ทั้งไฟล์**
4. CI ต้องมีขั้นตรวจ secret ใน diff (R0-02)

### บัญชี DB

5. แยกบัญชีต่อ host — `labms_web`, `labms_worker`, `labms_migrator` — สิทธิ์เฉพาะ schema ที่ใช้:
   - web/worker: `SELECT/INSERT/UPDATE/DELETE` เฉพาะ `labms_*` ที่โมดูลใช้ + สิทธิ์ตาม contract บน `statwarning` (docs/04 ข้อ 5.1) เท่าที่จำเป็น
   - migrator: DDL เฉพาะ `labms_*`
   - ABS/HIS: บัญชี read-only แยก
6. **ห้ามใช้บัญชี `lablbch`** ของระบบเดิม; หลัง go-live ของแต่ละระยะให้เปลี่ยนรหัสผ่านที่เคยฮาร์ดโค้ด (legacy-changes LC-010)

### ข้อมูลส่วนบุคคล (PDPA)

7. CID ผู้ป่วยเก็บเป็น **`cid_encrypted` + `cid_last4`** (เข้ารหัสระดับแอป, key อยู่นอก DB และนอก repo) — หรือไม่เก็บถ้าไม่จำเป็น (ตัดสินใน R1)
8. **ไม่เก็บ CID เจ้าหน้าที่** และไม่ย้ายรหัสผ่านของระบบเดิม
9. UI และ export แสดง CID แบบ mask เท่านั้น; audit การเปิดดูข้อมูลผู้ป่วยแบบละเอียดและการ export
10. **ห้าม log** token, CID, รหัสผ่าน, ข้อมูลผู้ป่วยแบบระบุตัว; query ทุกตัวต้อง parameterized

### AI agent และข้อมูลทดสอบ (docs/06 ข้อ 6)

11. **ห้ามใส่ข้อมูลผู้ป่วยจริงใน repo, prompt, test หรือ fixture** — HL7 ตัวอย่างต้อง de-identify (ชื่อ, HN, CID, วันเกิด) ด้วยสคริปต์ที่รันบนเครื่องโรงพยาบาลก่อน
12. agent ใช้เฉพาะฐาน development ที่มีข้อมูลสังเคราะห์ — **ห้ามให้ connection string ของ production**
13. agent ไม่อ่านหรือคัดลอกไฟล์ของระบบเดิมที่มี credential ฝังอยู่

## ทางเลือกที่ไม่เลือก

- **config แบบ plain text บน server + ไม่ commit**: ยังรั่วผ่าน backup/สำเนาได้ และไม่มีการบังคับใน CI
- **บัญชี DB เดียวสำหรับทุก host**: bug หรือช่องโหว่ใน host หนึ่งกระทบทุก schema

## ผลที่ตามมา

- R0-01: `appsettings*.json` ไม่มีค่า secret; README อธิบาย user-secrets
- R0-02: secret scanning ใน CI
- R0-03/R0-09: เอกสารระบุชื่อ key ที่ต้องตั้ง (ไม่ใส่ค่า) และสิทธิ์ของแต่ละบัญชี DB (legacy-changes LC-004)
- การเลือกวิธีเก็บ encryption key ของ CID (DPAPI, environment variable หรือ key store) ตัดสินเป็น ADR แยกใน R1

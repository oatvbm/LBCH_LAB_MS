# ADR-0005: Authentication ผ่าน LBCH CAS

- สถานะ: ตัดสินแล้ว (2026-11-15)
- ผู้ตัดสิน: tech lead
- ที่มา: docs/05 ข้อ 4; docs/02 ข้อ 5.1; docs/06 ข้อ 6.4; Lab_Stock ADR-009, ADR-024 และ `docs/AUTH_LBCH_CAS.md`

## บริบท

- ทุกระบบของโรงพยาบาลใช้ LBCH CAS (SSO + global logout) และ LabWeb, Lab_Stock เชื่อมแล้ว
- protocol ของ CAS เป็น authorization-code flow ของโรงพยาบาลเอง **ไม่ใช่ OIDC หรือ CAS มาตรฐาน**
- Lab_Stock มี implementation ที่ทดสอบกับ CAS จริงแล้ว (login, state ต่อ attempt, heartbeat, global logout, switch account)

## การตัดสินใจ

1. LAB_MS ใช้ **LBCH CAS เป็นทางเดียวในการยืนยันตัวตน** ด้วย AppCode **`LAB_MS`** — **ห้ามสร้างหน้ากรอกรหัสผ่านของ LAB_MS เอง**
2. CAS ตอบเพียง "ใครเป็นใคร และเข้า AppCode `LAB_MS` ได้หรือไม่"; สิทธิ์ราย action อยู่ใน LAB_MS (ADR-0006)
3. Flow (ตาม protocol และ Lab_Stock):
   - redirect ไป `{CAS_ADMIN_BASE_URL}/auth/login?appCode=LAB_MS&redirectUri=...&state=...` — `state` สุ่มใหม่ต่อ attempt, อายุ 10 นาที, ใช้ครั้งเดียว, cookie ยืนยันแยกต่อ attempt (รองรับหลายแท็บ)
   - callback `https://<host>/LAB_MS/cas/callback` (ตรงตัวพิมพ์กับที่ลงทะเบียน) รับเฉพาะ `code` + `state`
   - backend แลก code ที่ `POST {CAS_API_BASE_URL}/api/auth/exchange-code` แล้วอ่าน `GET /api/auth/me?appCode=LAB_MS`
   - ใช้ **`empKey` เป็น linkage key** ของผู้ใช้ (ไม่ใช้รหัสบุคลากรหรือ username)
4. **Token อยู่ฝั่ง server เท่านั้น** (token store ผูกกับ local session) — ห้ามส่ง token ไป browser, URL, log หรือ browser storage
5. Local cookie: `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/LAB_MS`; idle timeout แบบ sliding ตั้งค่าได้ (ค่าเริ่มต้น 20 นาทีตามตัวอย่างใน protocol) — **ค่าใช้งานจริงต้องตกลงกับฝ่าย IT/เจ้าของระบบก่อนเปิด production**
6. **ความสดของ session** (จาก Lab_Stock ADR-024):
   - หน้าเว็บ poll `GET /auth/session-status` ทุก 1 นาทีขณะแท็บแสดงอยู่ → backend เรียก heartbeat ของ CAS; ถ้า revoke → `401` → ล้างหน้าและพาไป login
   - request ที่เปลี่ยนข้อมูล (ไม่ใช่ GET/HEAD) ตรวจ heartbeat กับ CAS ทุกครั้ง; GET/HEAD ใช้ผลที่สำเร็จได้ไม่เกิน 1 นาที; ไฟล์ static ไม่ตรวจ
   - CAS ตอบ 400 จาก exchange/refresh = ถูกปฏิเสธ (login ใหม่); 5xx/เชื่อมต่อไม่ได้ = ยืนยันไม่ได้ (`503`) **ไม่ล้าง session**
7. **Logout**: `POST /auth/logout` (anti-forgery) → refresh ถ้าจำเป็น → `POST /api/auth/logout` (global logout) → **ล้าง token/session/cookie ทุกกรณี** แม้ CAS ล้มเหลว
8. **เปลี่ยนบัญชี**: เมื่อบัญชีไม่มีสิทธิ์ `LAB_MS` ใช้ `POST /auth/switch-account` (global logout ของบัญชีนั้นก่อน) แทนลิงก์ไป `/auth/login` ตรง ๆ ซึ่งจะวนกลับบัญชีเดิม
9. **Canonical path**: middleware redirect `/lab_ms` (ตัวพิมพ์ใดก็ได้) → `/LAB_MS`; ห้ามสร้าง callback URL จาก `Host` header ที่ไม่เชื่อถือ
10. แยก `CAS_ADMIN_BASE_URL` (browser) กับ `CAS_API_BASE_URL` (backend) ต่อ environment ใน config/environment variable
11. audit: login สำเร็จ/ล้มเหลว (เป็นหมวดสาเหตุ), logout และผล global revoke — **ไม่บันทึก code, state, token**
12. ทุกการเปลี่ยนโค้ดที่แตะ auth/token/session/cookie ต้องผ่าน `/security-review` และ tech lead รีวิวทุกบรรทัด (docs/06 ข้อ 6.4)

## ทางเลือกที่ไม่เลือก

- **ใช้ไลบรารี OIDC/CAS มาตรฐาน**: protocol ของ LBCH CAS ไม่ใช่มาตรฐานเหล่านั้น
- **เขียน adapter ใหม่ทั้งหมด**: Lab_Stock ทดสอบกับ CAS จริงแล้ว ยกมาปรับเร็วและเสี่ยงน้อยกว่า
- **เก็บ token ใน cookie/browser**: เพิ่มความเสี่ยงรั่วไหล ขัด protocol

## ผลที่ตามมา

- R0-05 ยกโค้ดจาก Lab_Stock (`LbchCasClient`, `CasSessionMiddleware`, `AuthController`, `CasLoginAttemptStore` และ views `Auth/*`) แล้วเปลี่ยน AppCode, path และ namespace — Lab_Stock ใช้ callback `/auth/callback` ต้องเปลี่ยน route เป็น `/cas/callback`
- ต้องลงทะเบียน AppCode `LAB_MS` และ callback ทุก environment กับผู้ดูแล CAS (legacy-changes LC-001)
- token store และ Data Protection keys ต้องอยู่รอดเมื่อ IIS recycle (เก็บ key ใน path ถาวรหรือ DB) — กำหนดใน R0-05/R0-09
- break-glass local admin เมื่อ CAS ล่ม (docs/03 ความเสี่ยง) **ไม่อยู่ใน ADR นี้** — ถ้าต้องการให้เขียน ADR แยก
- นโยบาย login ของจอ TV/หอผู้ป่วยและการลงนามบนเครื่องร่วม (A/B) ตัดสินใน R1

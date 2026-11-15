# R0 — Platform kernel (task cards)

> สถานะ: **รออนุมัติจาก tech lead** — ร่าง 2026-11-15
> อ้างอิงหลัก: docs/05 ข้อ 3.2, 3.3, 4, 6, 8 (แถว R0) · docs/06 ข้อ 2–4, 6 · docs/03 Phase 1 · docs/02 ข้อ 5 · Lab_Stock ADR-009/010/013/018/019/021/022/024

## ภาพรวม

**เป้าหมาย R0**: มีแพลตฟอร์มกลางที่ทุก lane ของ R1 ต่อยอดได้ — solution skeleton, CAS auth + heartbeat + global logout, ผู้ใช้/permission catalog/ชุดสิทธิ์ (ยกจาก Lab_Stock), audit, layout, CI, AppCode `LAB_MS`, ADR ชุดแรก

**เกณฑ์ผ่าน R0** (เอกสาร 05 ข้อ 8):
- [ ] login ผ่าน CAS ได้ (รวม heartbeat และ global logout ตาม CAS checklist ข้อ 14 และ 23 — logout จากอีกระบบแล้วถูกเตะออกภายใน 1 นาที)
- [ ] ผู้ดูแลมอบชุดสิทธิ์ได้ และ server ตรวจสิทธิ์ราย action ได้ (ไม่มีสิทธิ์ → 403 + audit)
- [ ] deploy staging ที่ `/LAB_MS` ได้ (tech lead เป็นคน deploy)
- [ ] CI: build + unit + integration (MySQL จริง) + architecture test ผ่านบน `main`

**วิธีทำ**: ทีละ task ตามลำดับ ไม่แบ่ง lane คู่ขนาน (เอกสาร 06 ข้อ 2.3) — agent เดียว + tech lead รีวิว

```text
R0-00 ADR ─▶ R0-01 skeleton ─▶ R0-02 CI + arch test ─▶ R0-03 DB + migration ─▶ R0-04 kernel + audit
                                                                                     │
   R0-09 staging ◀─ R0-08 SignalR/outbox ◀─ R0-07 layout ◀─ R0-06 users/สิทธิ์ ◀─ R0-05 CAS auth
```

**สิ่งที่ tech lead ต้องเตรียม (นอก repo)** — บันทึกใน `docs/legacy-changes.md`

| สิ่งที่ต้องมี | ใช้ใน | อ้างอิง |
|---|---|---|
| ~~GitHub private repo `LBCH_LAB_MS` + push `main` แรก~~ ✅ `oatvbm/LBCH_LAB_MS` (2026-11-15) — เหลือตั้ง branch protection ของ `main` | R0-02 | 06 ข้อ 2.1 |
| CAS: AppCode `LAB_MS` + callback localhost/staging (LC-001) | R0-05 | |
| CAS: API client scope `admin.user.read` (LC-002) | R0-06 | |
| DB development (ข้อมูลสังเคราะห์) + บัญชี DB แยก web/worker บน staging (LC-004) | R0-03, R0-09 | |
| Server IIS staging + Hosting Bundle .NET 10 + WebSockets | R0-09 | 02 ข้อ 8 |

---

## TASK-R0-00: ADR ชุดแรก

เป้าหมาย: บันทึกการตัดสินใจที่ทำแล้วในเอกสาร 05/06 เป็น ADR เพื่อให้ agent ทุกตัวอ้างอิงที่เดียว
อ้างอิง: docs/05 ข้อ 1 (D1–D10), 4, 5; docs/06 ข้อ 4; Lab_Stock `docs/DECISIONS.md`
แก้ได้เฉพาะ: `docs/adr/**`, `docs/DEVELOPMENT_STATUS.md`
Contract: —
เกณฑ์ผ่าน:
- [ ] `docs/adr/README.md` (รูปแบบ ADR + สถานะ: เสนอ/ตัดสินแล้ว/แทนที่แล้ว)
- [ ] ADR-0001 modular monolith + กติกาโมดูล (D2, 05 ข้อ 3.3)
- [ ] ADR-0002 MySQL ต่อ + เป้าหมาย 8.4 LTS (D10)
- [ ] ADR-0003 `MySql.EntityFrameworkCore` 10.x + Dapper สำหรับรายงาน, provider อยู่ใน Infrastructure เท่านั้น (D7)
- [ ] ADR-0004 SQL migration script ต่อโมดูล + `schema_migrations` ไม่ใช้ EF migrations
- [ ] ADR-0005 CAS authentication (ยก Lab_Stock ADR-009/024 ปรับเป็น `LAB_MS`)
- [ ] ADR-0006 permission catalog + ชุดสิทธิ์กำหนดโดยระบบ (ยก Lab_Stock ADR-013/021/022)
- [ ] ADR-0007 secrets นอก source + DB account แยกต่อ host
- [ ] ADR-0008 ทีม AI: 1 task = 1 branch = 1 PR, รีวิวไขว้ (06)
ไม่อยู่ในขอบเขต: โค้ด

## TASK-R0-01: Solution skeleton

เป้าหมาย: solution ที่ build ได้ตามโครงสร้างเอกสาร 05 ข้อ 3.2 พร้อม host ว่าง 2 ตัว
อ้างอิง: docs/05 ข้อ 3.2, 4; docs/03 Phase 1; docs/02 ข้อ 7 (observability)
แก้ได้เฉพาะ: `Lbch.LabMs.sln`, `global.json`, `Directory.Build.props`, `Directory.Packages.props`, `.editorconfig`, `src/Platform/**`, `src/Hosts/**`, `tests/Lbch.LabMs.Platform.*.Tests/**`, `README.md` (หัวข้อ "เริ่มพัฒนา"), `AGENTS.md` (ข้อ 6 คำสั่ง build/test)
Contract: ADR-0001, ADR-0003 (R0-00)
เกณฑ์ผ่าน:
- [ ] `global.json` ปัก .NET SDK 10.0.x; `Directory.Build.props`: nullable, `TreatWarningsAsErrors`, implicit usings; Central Package Management ใน `Directory.Packages.props`
- [ ] โปรเจกต์: `Lbch.LabMs.Platform.Kernel` (ไม่มี dependency ภายนอก), `.Platform.Infrastructure`, `.Platform.Web`, `Lbch.LabMs.Web` (host), `Lbch.LabMs.Worker` (Worker Service, รองรับ Windows Service)
- [ ] Web host: `PathBase` = `/LAB_MS`, Serilog (file + console, ห้าม log token/CID), `/health` (liveness), config แยก environment ไม่มี secret ใน `appsettings*.json`
- [ ] Worker host: Serilog + heartbeat log, `UseWindowsService()`
- [ ] โปรเจกต์ test ว่าง (xUnit) อย่างน้อย 1 test ต่อโปรเจกต์ที่ผ่าน
- [ ] `dotnet build` / `dotnet test` ผ่าน ไม่มี warning
ไม่อยู่ในขอบเขต: DB, CAS, UI, โมดูลธุรกิจ, `src/Core/**`

## TASK-R0-02: CI + architecture tests

เป้าหมาย: CI เป็น gatekeeper ตามเอกสาร 06 ข้อ 1 และบังคับกติกาโมดูลตั้งแต่วันแรก
อ้างอิง: docs/05 ข้อ 3.3 (ข้อ 4), 10; docs/06 ข้อ 1, 4.4
แก้ได้เฉพาะ: `.github/**`, `tests/Lbch.LabMs.ArchitectureTests/**`, `Directory.Packages.props`, `Lbch.LabMs.sln`
Contract: R0-01
เกณฑ์ผ่าน:
- [ ] GitHub Actions: restore → build → unit test → integration test (MySQL service container) → architecture test บน PR และ push `main`
- [ ] integration test รันกับ MySQL **8.0 และ 8.4** (matrix) เพราะ production ยังเป็น 8.0.17
- [ ] architecture test (NetArchTest): Kernel ไม่อ้าง Infrastructure/Web/EF; `MySql.*` อ้างได้เฉพาะ `*.Infrastructure`; โมดูลอ้างโมดูลอื่นได้เฉพาะ assembly `*.Contracts`/Application สาธารณะ (เขียน test ให้ผ่านกับ solution ปัจจุบันและพร้อมรับโมดูลใหม่)
- [ ] ขั้นตรวจ secret ใน diff (เช่น gitleaks) ใน CI
- [ ] PR template ที่มี checklist Definition of Done ตาม AGENTS.md ข้อ 7
ไม่อยู่ในขอบเขต: deploy อัตโนมัติ (agent ไม่ deploy — 06 ข้อ 6.5), branch protection (tech lead ตั้งเองบน GitHub)

## TASK-R0-03: ฐานข้อมูลและ migration runner

เป้าหมาย: รัน SQL migration ต่อโมดูลได้ และมี integration test กับ MySQL จริง
อ้างอิง: docs/05 ข้อ 3.2 (`database/<module>/NNNN_*.sql`), 4; docs/04 ข้อ 3 (หลักการ schema); Lab_Stock ADR-019
แก้ได้เฉพาะ: `database/platform/**`, `src/Platform/Lbch.LabMs.Platform.Infrastructure/Persistence/**`, `tools/Lbch.LabMs.DbMigrate/**`, `tests/Lbch.LabMs.Platform.Infrastructure.Tests/**`, `Directory.Packages.props`, `Lbch.LabMs.sln`, `docs/DEVELOPMENT_SETUP.md`
Contract: ADR-0002, ADR-0004
เกณฑ์ผ่าน:
- [ ] migration runner: อ่าน `database/<module>/NNNN_*.sql` ตามลำดับ, บันทึก `schema_migrations` (module, version, checksum, applied_at), ปฏิเสธเมื่อ checksum ของ script ที่รันแล้วเปลี่ยน, รันใน transaction ต่อ script เท่าที่ MySQL ทำได้
- [ ] console `tools/Lbch.LabMs.DbMigrate` (รันโดยคนตอน deploy) — **host ไม่ migrate อัตโนมัติตอน start**
- [ ] `database/platform/0001_platform_baseline.sql` สร้าง schema `labms_platform` (utf8mb4_0900_ai_ci) + `schema_migrations`
- [ ] `LabMsDbContextBase` (Oracle provider, connection จาก config/secret, ไม่มี EF migrations)
- [ ] integration test ด้วย Testcontainers (MySQL 8.4) : runner รันซ้ำแล้ว idempotent, checksum เปลี่ยนแล้ว fail
- [ ] `docs/DEVELOPMENT_SETUP.md`: เปิด MySQL dev ด้วย Docker, ตั้ง user-secrets, รัน migrator
ไม่อยู่ในขอบเขต: ตารางของผู้ใช้/สิทธิ์/audit (R0-04, R0-06), schema ของโมดูลอื่น, การแตะ DB `statwarning`

## TASK-R0-04: Kernel abstractions + audit + module registration

เป้าหมาย: สัญญากลางที่ทุกโมดูลใช้ และ audit log ที่เขียนใน transaction เดียวกับการเปลี่ยนข้อมูล
อ้างอิง: docs/05 ข้อ 3.2 (Kernel), 3.3 (ข้อ 5 feature flag), 1.0 (audit before/after); docs/04 ข้อ 3 (`audit_log`)
แก้ได้เฉพาะ: `src/Platform/Lbch.LabMs.Platform.Kernel/**`, `src/Platform/Lbch.LabMs.Platform.Infrastructure/{Audit,Modules,Time}/**`, `database/platform/0002_*.sql`, `tests/Lbch.LabMs.Platform.*.Tests/**`
Contract: R0-03
เกณฑ์ผ่าน:
- [ ] Kernel: `ICurrentUser`, `IPermissionService`, `IAuditLog`, `IClock` (Asia/Bangkok), `IDomainEvent` + `IDomainEventPublisher` (in-process), `ILabMsModule` (ชื่อ, Area, permission ที่ประกาศ, เมนู, feature flag)
- [ ] `audit_log` ตามเอกสาร 04 (เพิ่ม `module`, `session_user_id`, `signed_by_user_id`, before/after JSON) + implementation ที่เขียนผ่าน connection/transaction เดียวกับ DbContext ที่เรียก
- [ ] การลงทะเบียนโมดูล + เปิด/ปิดด้วย config (`Modules:<name>:Enabled`) — โมดูลที่ปิดไม่ลง route/เมนู/permission
- [ ] test: audit rollback พร้อม transaction หลัก; โมดูลปิดแล้วไม่ถูกลงทะเบียน
ไม่อยู่ในขอบเขต: UI ดู audit (R0-07), outbox ข้าม process (R0-08)

## TASK-R0-05: CAS authentication ⚠ security review

เป้าหมาย: login/logout ผ่าน CAS ด้วย AppCode `LAB_MS` ครบ flow ตาม protocol โดยยกโค้ดที่ใช้งานแล้วจาก Lab_Stock
อ้างอิง: docs/02 ข้อ 5.1; Lab_Stock `docs/AUTH_LBCH_CAS.md`, ADR-009/024; ต้นทาง Lab_Stock `CasContracts.cs`, `CasLoginAttemptStore.cs`, `CasOptions.cs`, `LbchCasClient.cs`, `CasSessionMiddleware.cs`, `AuthController.cs` และ views `Auth/*`
แก้ได้เฉพาะ: `src/Platform/Lbch.LabMs.Platform.Infrastructure/Cas/**`, `src/Platform/Lbch.LabMs.Platform.Web/Authentication/**`, `src/Platform/Lbch.LabMs.Platform.Web/Views/Auth/**`, `src/Hosts/Lbch.LabMs.Web/Program.cs`, `src/Hosts/Lbch.LabMs.Web/appsettings*.json` (key เปล่าเท่านั้น), `tests/Lbch.LabMs.Platform.Web.Tests/**`, `docs/adr/0005-*.md`
Contract: R0-04 (`ICurrentUser`)
เกณฑ์ผ่าน:
- [ ] login → `/LAB_MS/cas/callback` → exchange-code ฝั่ง server → profile `me?appCode=LAB_MS` → local cookie (HttpOnly, Secure, SameSite=Lax, Path=`/LAB_MS`, sliding 20 นาที)
- [ ] token เก็บฝั่ง server เท่านั้น (ไม่ส่งไป browser); login attempt มีอายุ 10 นาที รองรับหลายแท็บ
- [ ] heartbeat `GET /auth/session-status` ทุก 1 นาที; request ที่เปลี่ยนข้อมูลตรวจ heartbeat ทุกครั้ง; CAS 400 = ถูกปฏิเสธ, 5xx = 503 ไม่ล้าง session
- [ ] `POST /auth/logout` (anti-forgery) → global logout CAS → ล้าง local ทุกกรณี; `/auth/switch-account`
- [ ] canonical path middleware `/lab_ms` (ตัวพิมพ์ใดก็ได้) → `/LAB_MS`
- [ ] หน้า 401 / 403 (ไม่มีสิทธิ์เข้า `LAB_MS`) / session หมดอายุ ตามข้อความมาตรฐาน CAS
- [ ] test ด้วย fake CAS server (WireMock หรือ test handler) ครอบคลุม: สำเร็จ, state ไม่ตรง, code ใช้ซ้ำ, ไม่มีสิทธิ์ AppCode, revoke ระหว่างใช้งาน, CAS ล่ม
- [ ] ผ่าน `/security-review` และ tech lead รีวิวทุกบรรทัด (06 ข้อ 6.4)
ไม่อยู่ในขอบเขต: permission ราย action (R0-06), break-glass local admin (ADR แยก ถ้าต้องการ — เอกสาร 03 ความเสี่ยง CAS ล่ม)

## TASK-R0-06: ผู้ใช้, permission catalog, ชุดสิทธิ์ ⚠ security review

เป้าหมาย: ผู้ดูแลค้นผู้ใช้จาก CAS, มอบ/ถอนชุดสิทธิ์ และ server ตรวจสิทธิ์ราย action ได้
อ้างอิง: docs/05 ข้อ 4, 4.1; Lab_Stock ADR-013/021/022, `CasProvisioning.cs`, `database/migrations/0003_permission_catalog.sql`
แก้ได้เฉพาะ: `database/platform/0003_*.sql`, `src/Platform/**/Users/**`, `src/Platform/**/Authorization/**`, `src/Platform/Lbch.LabMs.Platform.Web/Areas/Admin/**`, `tests/Lbch.LabMs.Platform.*.Tests/**`
Contract: R0-04, R0-05
เกณฑ์ผ่าน:
- [ ] ตาราง `labms_platform`: `app_user` (cas_emp_key unique, display_name, snapshot สังกัด/ตำแหน่งจาก CAS, is_active), `permission`, `permission_set`, `permission_set_permission`, `user_permission_set` (+ scope หน่วยงาน, NULL = ไม่จำกัด)
- [ ] permission catalog ประกาศโดยโมดูล (`ILabMsModule`) แล้ว sync ลงตารางผ่าน migration — ผู้ดูแลแก้ catalog/ชุดสิทธิ์ไม่ได้ (ADR-0006)
- [ ] permission ของ platform: `platform.users.manage`, `platform.audit.view`; ชุดสิทธิ์เริ่มต้น "ผู้ดูแลระบบ LAB_MS"
- [ ] `[RequirePermission("...")]` + policy ตรวจฝั่ง server ทุก action; ไม่มีสิทธิ์ → 403 + audit
- [ ] หน้าผู้ดูแล: ค้นผู้ใช้จาก CAS (`admin.user.read`), เพิ่ม/ซิงก์, เปิด/ปิดบัญชี, มอบ/ถอนชุดสิทธิ์ — ทุกการเปลี่ยนเขียน audit ใน transaction เดียว
- [ ] ป้องกัน: ปิดบัญชีตัวเอง, ถอนสิทธิ์ผู้ดูแลคนสุดท้าย
- [ ] SQL bootstrap สำหรับผู้ดูแลคนแรก (tech lead รันเอง ไม่มีชื่อบุคคลใน repo)
- [ ] integration test กับ MySQL; ผ่าน `/security-review`
ไม่อยู่ในขอบเขต: permission ของโมดูล Stat/Critical (`stat.view` ฯลฯ — ลงใน R1-00), `lis_user_code` / PIN (R1 ตามการตัดสินใจแบบลงนาม A/B)

## TASK-R0-07: Layout และ design system

เป้าหมาย: หน้าตาเดียวกันทุกโมดูล เมนูตามโมดูลและสิทธิ์
อ้างอิง: docs/05 ข้อ 3, 6 (ยก DESIGN.md + layout จาก Lab_Stock); docs/03 Phase 1; Lab_Stock `DESIGN.md`, `Views/Shared/_Layout.cshtml`
แก้ได้เฉพาะ: `DESIGN.md`, `src/Platform/Lbch.LabMs.Platform.Web/{Views/Shared,wwwroot,Navigation}/**`, `src/Platform/Lbch.LabMs.Platform.Web/Areas/Admin/Views/Audit/**`, `tests/Lbch.LabMs.Platform.Web.Tests/**`
Contract: R0-04 (`ILabMsModule` เมนู), R0-06 (permission)
เกณฑ์ผ่าน:
- [ ] `DESIGN.md` ปรับจาก Lab_Stock เป็น LAB_MS (Service / Quality / Platform)
- [ ] `_Layout`: ชื่อระบบ, เมนูจัดกลุ่ม Service / Quality / ระบบ แสดงเฉพาะโมดูลที่เปิดและผู้ใช้มีสิทธิ์, ชื่อผู้ใช้, ปุ่มออกจากระบบ, แถบสถานะการเชื่อมต่อ (heartbeat/SignalR)
- [ ] htmx + static asset ที่ vendor ไว้ใน repo (ไม่โหลด CDN ใน production)
- [ ] หน้า error/403/404 และ empty state มาตรฐาน; หน้าแรก (dashboard ว่างที่แสดงโมดูลที่เข้าได้)
- [ ] หน้าดู audit log (อ่านอย่างเดียว, `platform.audit.view`, filter ช่วงวัน/ผู้ใช้/entity)
- [ ] ใช้งานได้ที่ 1366×768 และ tablet
ไม่อยู่ในขอบเขต: หน้าของโมดูลธุรกิจ, โหมด TV (R1)

## TASK-R0-08: SignalR hub + outbox skeleton

เป้าหมาย: โครง real-time ที่ R1 (Stat dashboard, Critical) ต่อได้ทันที
อ้างอิง: docs/02 ข้อ 2.1 (outbox → push), 7; docs/03 Phase 1 (skeleton)
แก้ได้เฉพาะ: `database/platform/0004_*.sql`, `src/Platform/**/Realtime/**`, `src/Platform/**/Outbox/**`, `tests/Lbch.LabMs.Platform.*.Tests/**`
Contract: R0-04 (`IDomainEvent`), R0-05 (auth)
เกณฑ์ผ่าน:
- [ ] ตาราง `domain_event` (outbox) ตามเอกสาร 04 + `IOutboxWriter` ที่เขียนใน transaction เดียวกับข้อมูล
- [ ] `DashboardBroadcaster` (BackgroundService ใน web) อ่าน outbox ทุก 3–5 วินาทีแบบ cursor แล้ว push ผ่าน hub `/LAB_MS/hubs/platform` (ต้อง login)
- [ ] client JS reconnect อัตโนมัติ + แสดงสถานะในแถบเชื่อมต่อ (R0-07)
- [ ] test: event ที่เขียนถูก push ครั้งเดียว; rollback แล้วไม่ถูก push
ไม่อยู่ในขอบเขต: event ของโมดูลธุรกิจ, การลบ outbox เก่า (ทำใน R1 พร้อม retention policy)

## TASK-R0-09: แพ็กเกจ deploy staging

เป้าหมาย: tech lead deploy staging `/LAB_MS` ได้ด้วย script ที่ตรวจแล้ว และปิดเกณฑ์ผ่าน R0
อ้างอิง: docs/02 ข้อ 8; docs/06 ข้อ 6.5; Lab_Stock `docs/DEPLOYMENT-IIS.md`
แก้ได้เฉพาะ: `deploy/**`, `docs/DEPLOYMENT-IIS.md`, `docs/DEVELOPMENT_STATUS.md`, `src/Hosts/Lbch.LabMs.Web/web.config`
Contract: R0-01 ถึง R0-08
เกณฑ์ผ่าน:
- [ ] script publish (web + worker + migrator) ออกเป็น artifact; script deploy ที่ใช้ `app_offline.htm` และ **ไม่เขียนทับ config ทั้งไฟล์** (ตามข้อกำหนด CAS)
- [ ] `docs/DEPLOYMENT-IIS.md`: app pool, PathBase `/LAB_MS`, WebSockets, environment variables/secret ที่ต้องตั้ง (ชื่อ key เท่านั้น), ติดตั้ง Worker เป็น Windows Service, รัน migrator
- [ ] checklist ทดสอบบน staging: login, heartbeat, global logout ข้ามระบบ (CAS checklist 14, 23), มอบ/ตรวจสิทธิ์, audit, SignalR reconnect หลัง app pool recycle
- [ ] tech lead deploy และบันทึกผลใน `DEVELOPMENT_STATUS.md` → ปิด R0
ไม่อยู่ในขอบเขต: production, การ deploy โดย agent

---

## ต้องตัดสินใจก่อนเริ่ม (ข้อเสนอของ Claude)

| # | เรื่อง | ข้อเสนอ | เหตุผล |
|---|---|---|---|
| Q1 | DB สำหรับ development | MySQL **8.4 ใน Docker** บนเครื่อง dev + Testcontainers ใน test; CI ทดสอบทั้ง 8.0 และ 8.4 | เครื่องนี้มี Docker แล้ว; ไม่ต้องดูแล instance แบบ Lab_Stock (AppServ 8.0.17); ทดสอบ 8.0 ไว้เพราะ production ยังไม่อัปเกรด |
| Q2 | CI | **GitHub Actions** (ubuntu runner + MySQL service container) | ใช้ร่วมกันได้ทั้ง Claude และ Codex (06 ข้อ 2.1) |
| Q3 | การรัน migration | console `tools/Lbch.LabMs.DbMigrate` ที่คนรันตอน deploy — **ไม่ auto-migrate ตอน start** | DB ใช้ร่วมกับระบบอื่นบน server เดียว ควบคุมเวลาเปลี่ยน schema ได้ |
| Q4 | การยกโค้ดจาก Lab_Stock | คัดลอกเฉพาะไฟล์ที่ใช้ (ไม่เก็บประวัติ git) แล้วปรับ namespace/AppCode — `git subtree` ใช้ตอนย้ายทั้งโมดูลใน R2 | R0 ใช้แค่ auth/สิทธิ์/layout ไม่กี่ไฟล์ |
| Q5 | ชื่อตาราง | เอกพจน์ตามเอกสาร 04 (`app_user`, `audit_log`) ไม่ใช่พหูพจน์แบบ Lab_Stock (`audit_logs`) | เอกสาร 04/05 ใช้แบบนี้อยู่แล้ว ตอนย้าย Lab_Stock (R2) ค่อย map |
| Q6 | SignalR/outbox | ทำ skeleton ใน R0 (R0-08) | R1 lane L2/L3 ต้องใช้พร้อมกัน ถ้าไม่ทำใน R0 จะชนกันที่ Platform |

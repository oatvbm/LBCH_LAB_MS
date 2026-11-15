# ADR-0008: การทำงานแบบทีม AI, CI และการยกโค้ดจาก Lab_Stock

- สถานะ: ตัดสินแล้ว (2026-11-15)
- ผู้ตัดสิน: tech lead
- ที่มา: docs/06 ทั้งฉบับ; docs/05 ข้อ 6 (D8); task card R0 ข้อ Q2, Q4, Q6; Lab_Stock ADR-011

## บริบท

- ผู้พัฒนาคือ tech lead 1 คน + Claude Code และ Codex หลายตัวทำงานคู่ขนาน (docs/05 ข้อ 11.3)
- คอขวดคือเวลารีวิวและตัดสินใจของ tech lead ไม่ใช่จำนวน agent (docs/06 ข้อ 7)
- Lab_Stock มีโค้ด auth/สิทธิ์/layout ที่ใช้งานแล้ว และยังพัฒนาต่อใน repo เดิมจนถึง R2

## การตัดสินใจ

### บทบาท

1. **tech lead (คน)**: เขียน/อนุมัติ task card, ตัดสินใจ, รีวิวสุดท้ายและ merge, UAT, deploy
2. **Implementer (agent)**: ทำงานตาม task card ใน branch/worktree ของตัวเอง เขียน test เปิด PR
3. **Reviewer (agent อีกตัว ไขว้กัน)**: Codex รีวิว PR ของ Claude; Claude (`/code-review`) รีวิว PR ของ Codex
4. **Gatekeeper (CI)**: build, unit, integration (MySQL จริง), architecture test, secret scan

### กติกา

5. **1 task = 1 branch = 1 worktree = 1 PR**; branch `task/<task-id>-<slug>`; ห้าม commit ตรงเข้า `main` (ยกเว้น initial commit ของ repo)
6. PR เล็ก (แนะนำ ≤ ~500 บรรทัดที่เปลี่ยน) และต้องมี test
7. agent แก้ได้เฉพาะไฟล์ใน "แก้ได้เฉพาะ" ของ task card — ถ้าจำเป็นต้องแก้นอกขอบเขต ให้ระบุใน PR พร้อมเหตุผล
8. merge เมื่อ: **CI ผ่าน + reviewer ไม่มี finding ร้ายแรงค้าง + tech lead อนุมัติ** — agent ไม่ merge และไม่ deploy
9. การตัดสินใจเชิงออกแบบ → ADR; หลัง merge → อัปเดต `docs/DEVELOPMENT_STATUS.md`; แตะระบบเดิม → `docs/legacy-changes.md`
10. เปิดคู่ขนานไม่เกิน 3–4 lane ต่อผู้รีวิว 1 คน; **R0 ทำทีละ task ไม่แบ่ง lane**
11. task card อยู่ใน `docs/tasks/` ตามรูปแบบใน AGENTS.md

### CI

12. ใช้ **GitHub Actions** บน repo private `oatvbm/LBCH_LAB_MS` — Claude และ Codex ใช้ branch/PR/CI ชุดเดียวกัน
13. CI รันบน PR และ push `main`: restore → build (warning = error) → unit → integration (MySQL 8.0 และ 8.4) → architecture test → secret scan
14. CI **ไม่ deploy** — deploy staging/production ทำโดย tech lead ผ่าน script ที่ตรวจแล้ว
15. branch protection ของ `main` (ต้องผ่าน PR + CI) ตั้งโดย tech lead บน GitHub

### การยกโค้ดจาก Lab_Stock

16. R0 **คัดลอกเฉพาะไฟล์ที่ต้องใช้** (auth, ผู้ใช้/สิทธิ์, layout, DESIGN.md) แล้วปรับ namespace เป็น `Lbch.LabMs.*`, AppCode เป็น `LAB_MS` และตั้งชื่อตารางตาม ADR-0004 — **ไม่เก็บประวัติ git** และระบุไฟล์ต้นทางใน PR
17. ห้ามคัดลอก `.env`, `.local-dev/`, `RawData/` หรือไฟล์ที่มี credential
18. การย้ายทั้งโมดูล Lab_Stock / Lab_RMS ใน R2 ใช้ `git subtree` ได้ถ้าต้องการเก็บประวัติ (ตัดสินใน R2)

### ขอบเขต R0

19. R0 รวม SignalR hub + outbox skeleton (R0-08) เพื่อไม่ให้ lane L2/L3 ของ R1 ชนกันที่ Platform

## ทางเลือกที่ไม่เลือก

- **agent ตัวเดียวทำทุกอย่าง**: ช้า และไม่มีการรีวิวไขว้
- **รีวิวด้วยโมเดลเดียวกับที่เขียน**: มักพลาดจุดผิดแบบเดียวกัน
- **`git subtree` ของ Lab_Stock ตั้งแต่ R0**: ได้ประวัติทั้ง repo แต่ R0 ใช้แค่ไม่กี่ไฟล์และต้องปรับเกือบทุกไฟล์

## ผลที่ตามมา

- R0-02 สร้าง workflow ตามข้อ 13 และ PR template ที่มี Definition of Done
- เวลารีวิวของ tech lead เป็นตัวกำหนดความเร็ว — ประเมินใหม่หลังจบ R0 และ R1 (docs/06 ข้อ 7)

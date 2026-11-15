# สถานะการพัฒนา LAB_MS

> อัปเดตหลัง merge ทุกครั้ง (AGENTS.md ข้อ 3.7) — agent ต้องอ่านไฟล์นี้ก่อนเริ่ม task

อัปเดตล่าสุด: 2026-11-15

## ระยะปัจจุบัน

**R0 Platform kernel — กำลังทำ** (task card และข้อเสนอ Q1–Q6 อนุมัติ 2026-11-15) → [docs/tasks/R0-platform-kernel.md](tasks/R0-platform-kernel.md)

ADR ที่ใช้อยู่: [docs/adr/](adr/README.md) (ADR-0001 ถึง ADR-0008)

## เสร็จแล้ว

| วันที่ | รายการ | PR |
|---|---|---|
| 2026-11-15 | สร้าง repo (GitHub private `oatvbm/LBCH_LAB_MS`), README, AGENTS.md, CLAUDE.md, legacy-changes.md, เอกสาร 01–06, task card R0 | initial commit บน `main` |

## กำลังทำ

| Task | ผู้ทำ | branch / PR | หมายเหตุ |
|---|---|---|---|
| TASK-R0-00 ADR ชุดแรก | Claude | `task/r0-00-adr` | รอรีวิว PR |

## รอข้อมูล / รอตัดสินใจ

- branch protection ของ `main` บน GitHub (tech lead ตั้งหลัง R0-02)
- นโยบายอายุ session ของ CAS ใน production (ADR-0005 ข้อ 5) — ตกลงกับฝ่าย IT
- ค่า CAS ของ AppCode `LAB_MS` (legacy-changes LC-001, LC-002)
- Phase 0 ของ R1 (เอกสาร 03): HL7 ตัวอย่างที่ de-identify แล้ว, ผล query Q1–Q8, golden dataset

## Lane ที่เปิดอยู่

ยังไม่เปิด — เปิด lane L1–L5 ได้หลัง R0 เสร็จและ merge contract ของ R1 (เอกสาร 06 ข้อ 8)

# สถานะการพัฒนา LAB_MS

> อัปเดตหลัง merge ทุกครั้ง (AGENTS.md ข้อ 3.7) — agent ต้องอ่านไฟล์นี้ก่อนเริ่ม task

อัปเดตล่าสุด: 2026-11-15

## ระยะปัจจุบัน

**R0 Platform kernel — วางแผน (task card รออนุมัติ)** → [docs/tasks/R0-platform-kernel.md](tasks/R0-platform-kernel.md)

## เสร็จแล้ว

| วันที่ | รายการ | PR |
|---|---|---|
| 2026-11-15 | สร้าง repo (GitHub private `oatvbm/LBCH_LAB_MS`), README, AGENTS.md, CLAUDE.md, legacy-changes.md, เอกสาร 01–06, task card R0 | initial commit บน `main` |

## กำลังทำ

| Task | ผู้ทำ | branch / PR | หมายเหตุ |
|---|---|---|---|
| — | | | |

## รอข้อมูล / รอตัดสินใจ

- คำถามที่ต้องตัดสินใจก่อนเริ่ม R0 อยู่ในหัวข้อ "ต้องตัดสินใจก่อนเริ่ม" ของ task card R0
- ค่า CAS ของ AppCode `LAB_MS` (legacy-changes LC-001, LC-002)
- Phase 0 ของ R1 (เอกสาร 03): HL7 ตัวอย่างที่ de-identify แล้ว, ผล query Q1–Q8, golden dataset

## Lane ที่เปิดอยู่

ยังไม่เปิด — เปิด lane L1–L5 ได้หลัง R0 เสร็จและ merge contract ของ R1 (เอกสาร 06 ข้อ 8)

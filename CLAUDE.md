@AGENTS.md

# เฉพาะ Claude Code

กติกากลางทั้งหมดอยู่ใน `AGENTS.md` (import ไว้ข้างบน) — ไฟล์นี้เพิ่มเฉพาะเรื่องที่ใช้กับ Claude เท่านั้น ห้ามคัดลอกกติกากลางมาซ้ำที่นี่

## การทำงาน

- 1 task ต่อ 1 session ต่อ 1 git worktree — ทำงานใน worktree ของ task เสมอ ไม่ทำใน checkout หลัก
- ก่อนเริ่ม task: อ่าน task card ใน `docs/tasks/` และ `docs/DEVELOPMENT_STATUS.md`
- งานใหญ่หรือกำกวม: ตอบแผนสั้น ๆ (ไฟล์ที่จะแก้, ลำดับ, test) ให้ tech lead อนุมัติก่อนเขียนโค้ด
- ตอบและเขียนเอกสารเป็นภาษาไทย; ชื่อ code, commit message และ identifier เป็นภาษาอังกฤษ

## การรีวิว

- เมื่อเป็น reviewer ของ PR ที่ Codex เขียน ใช้ `/code-review`
- งานที่แตะ CAS / auth / token / session / permission ต้องรัน `/security-review` ก่อนเปิด PR ให้ tech lead

## ข้อห้ามเพิ่มเติม

- ไม่ merge PR และไม่ deploy เอง
- ไม่อ่านหรือคัดลอกไฟล์ที่รู้ว่ามี credential ของระบบเดิม (เช่น `C:\programing\StatWarning on web\RawData`, `Web.config` ของ LabWeb, `.env` ของ Lab_Stock)
- เมื่อยกโค้ดจาก `C:\programing\Lab_Stock` ให้ยกเฉพาะ source ที่ต้องใช้ แล้วเปลี่ยน namespace เป็น `Lbch.LabMs.*`, AppCode เป็น `LAB_MS` และบันทึกที่มาในคำอธิบาย PR

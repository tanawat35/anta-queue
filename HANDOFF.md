# HANDOFF — ANTA Driver Rotation Queue (ส่งต่อให้ Claude)

อัปเดต: 6 พ.ย. 2569 · ผู้ส่งต่อ: Keng (Hermes) · ผู้รับ: Claude

## เป้าหมายของงาน
ระบบจัดคิวงานให้พนักงานขับรถ ANTA อย่างเป็นธรรม — ปล่อยลูก/จับลำดับ **ครั้งเดียว** เพื่อได้ "ลำดับตั้งต้น" ของรอบ แล้วหลังจากนั้นใช้ลำดับนั้นรันงานจริงใน Google Sheet (ไม่ต้องจับใหม่ทุกวัน)

## ปัญหาที่ค้างอยู่ (เหตุผลที่ส่งต่อ)
หน้าเว็บปล่อยลูก **ไม่มีการสุ่มจริง** — ลำดับที่ได้คือการเรียงตาม "วันทำงานสะสม" จากข้อมูล 30 วันล่าสุด (deterministic) กดปล่อยกี่ครั้งก็ได้ลำดับเดิม ดูไม่เหมือนการจับสลากจริง
ต้องทำให้:
1. มีการสุ่มจริงเมื่อกดปล่อย (ผลลัพธ์ต่างกันได้)
2. แต่ยังคงความเป็นธรรม เช่น สุ่มเฉพาะในกลุ่มที่วันสะสมใกล้กัน (ช่วง ±1 วัน) หรือ weighted random ให้น้ำหนักคนที่สะสมน้อยกว่า
3. ผลลัพธ์ต้องถูกบันทึกเป็น "ลำดับของรอบ" และส่งเข้า Google Sheet เพื่อใช้รันงาน

## โครงสร้างไฟล์
| ไฟล์ | หน้าที่ |
|---|---|
| `/root/.hermes/scripts/rotation_queue.py` | คำนวณคิว: อ่าน Job Log 2 เดือน, น้ำหนักวันทำงาน, โทษ (punish), รายชื่อที่ตัดออก |
| `/root/.hermes/scripts/rotation_sheet.py` | เขียน Google Sheet แท็บ คิว/กติกา/บันทึกยกเลิก |
| `/root/.hermes/scripts/rotation_web.py` | สร้างหน้าเว็บ static (`/root/workspace/anta-queue/index.html`) |
| `/root/.hermes/scripts/rotation_draw.py` | ล็อคลำดับรอบ (new / show / assign / undo) → `anta_round.json` |
| `/root/.hermes/scripts/anta_queue_publish.sh` | รัน web + sheet แล้ว push GitHub (cron ทุก 10 นาที) |
| `/root/.workspace/anta-queue/` | working copy ของ repo (index.html, queue.json) |

State:
- `/root/.hermes/anta_round.json` — รอบที่ล็อกไว้ (date, round, order[])
- `/root/.hermes/anta_rotation.json` — โทษ/เหตุการณ์ (punish, events)
- `/root/.hermes/anta_rotation_excluded.json` — รายชื่อที่ไม่เข้าระบบคิว (ปัจจุบัน: สมพร เจะโกะ — พนักงานบริษัท ไม่เอาเข้าคิว แต่ห้ามลบประวัติใน Job Log)
- `/root/.hermes/rotation_pointer.json` — id ของ Google Sheet คิว

## กติกาที่ตกลงกับเจ้าของงานแล้ว (ห้ามเปลี่ยนเอง)
- ใช้กับงาน **ANTA เท่านั้น** (ANTA/WILSON/SALOMON ที่ลง Job Log) ไม่ยุ่งกับ Foodsmiths/พัชรา
- **เฉพาะรถ 4W** (ตัด 6W ออกตามสั่ง 6/11/69)
- น้ำหนัก "วันทำงาน": งานปกติ 1 · เข้ารับ ≥21:00 = 1.5 · ต่างจังหวัด 2
- เรียง: วันสะสมน้อย → มาก, เสมอใช้ "งานล่าสุดเก่ากว่า", แล้วชื่อ
- ยกเลิก/ไม่ไปตามนัด: นับงานเป็นวันทำงาน + จัดท้ายคิว 1 รอบ; ซ้ำใน 30 วัน = 2 รอบ
- ปฏิเสธตอนถูกเสนอ: ไม่มีโทษครั้งแรก, ครั้งที่ 2 ในเดือน = ต่อท้ายคิว
- ห้ามลบ/แก้แถวประวัติใน Job Log (ใช้คิดเงิน/วางบิล)

## ระบบที่เกี่ยวข้อง (อย่าแตะนอกขอบเขต)
- ANTA Job Log พ.ย. 69: `1wVDvo6tfgY2w2LETFpcdb5_7A9GeQ1xWqqtPT2Unw0I` (แท็บ Job Log, 17 คอลัมน์ A:Q)
- ต.ค. 69: `1CRR15kL1G6Irq00tryORmKGo0HUi9wVn7m64CIPalPg`
- ชีตคิว: `1nzRdibVVIUg9PhCO1j7HtYYPYF0lfeW6MDqDLz4vshQ` (แท็บ คิว / กติกา / บันทึกยกเลิก)
- Repo: `tanawat35/anta-queue` (public) · Pages: https://tanawat35.github.io/anta-queue/
- Token GitHub: `/root/.gh_token` · Google: `/root/.hermes/google_token.json` (helper `google_auth_helper.py`)
- **ยังไม่ควรใช้ Google Sheets wrapper เก่าเขียนแถว** — ใช้ REST `values:batchUpdate` / `PUT` ตรง และอ่านกลับตรวจทุกครั้ง

## สถานะปัจจุบัน
- เว็บ live แล้ว (build 50f2aae) แสดง 17 เลน แต่ละเลนมีหลุมเลขของตัวเอง ลูกหล่นลงหลุมตัวเอง (ไม่มีการสุ่ม)
- ชีตคิว live และตรงกับลำดับที่คำนวณ (17 คน)
- cron `anta_queue_publish.sh` ทุก 10 นาที — **ถูกปิดแล้ว** ตอนส่งต่อ (เจ้าของงานจะให้ Claude ดูแลเว็บต่อ)

## สิ่งที่เจ้าของงานต้องการเห็น
หน้าเว็บที่กดแล้ว **รู้สึกเหมือนจับสลากจริง** (มีการสุ่ม) แต่ผลลัพธ์ยังยุติธรรมและตรวจย้อนหลังได้ (ต้องเห็นว่าใครสะสมกี่วัน) และเมื่อได้ลำดับแล้วต้องส่งเข้า Sheet เพื่อใช้งานต่อทันที

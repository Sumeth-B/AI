# TOR Strategy & Artwork | AI Easy 1.0

เว็บแอปหน้าเดียว (Single-page HTML ไม่มี Build Step) สำหรับทีม PR ของ Plan B Media
ใช้วิเคราะห์ TOR และสร้าง Artwork ประกอบข้อเสนอทางเทคนิค เผยแพร่เป็น Claude Artifact

> ซอร์สโค้ดของแอปไม่ได้เก็บไว้ใน Repo นี้ เพราะ Repo เป็น Public และโค้ดมี ID ของโฟลเดอร์
> Google Drive ที่ตั้งสิทธิ์ให้ทุกคนเขียนได้ (Anyone: Writer) — ต้นฉบับที่ใช้งานจริงอยู่ใน Artifact

## ขั้นตอนการทำงาน

1. **กลยุทธ์สื่อประชาสัมพันธ์** — อัปโหลด TOR (PDF/Word/ข้อความ) → Claude วิเคราะห์และสรุปกลยุทธ์
2. **Creative Concept Idea** — ต่อยอดจาก Step 1 เป็นแนวคิดสร้างสรรค์
3. **Prompt To Artwork** — สร้าง Prompt / สั่งออกแบบ Artwork ผ่าน Canva AI และ Moda AI

## Architecture

```
เบราว์เซอร์ (หน้า Artifact ใน claude.ai viewer)
 ├─ claude.use("sample")    → Claude ของ "ผู้เปิดดู" (ใช้โควตาของผู้เปิดดู)
 ├─ claude.use("mcp")       → Connector ของ "ผู้เปิดดู" (Drive, Canva, Moda, Gmail)
 ├─ claude.use("downloads") → บันทึกไฟล์ลงเครื่อง (ต้องยืนยันทุกครั้ง)
 └─ localStorage            → สถานะงานต่อผู้ใช้ต่อเบราว์เซอร์
ไลบรารีฝั่งหน้าเว็บ: pdf.js (อ่าน/OCR PDF), mammoth (Word), jsPDF + html2canvas (ส่งออก PDF), JSZip
```

การล็อกอินด้วยอีเมล @planbmedia.co.th เป็นแค่การกรองโดเมนฝั่งเบราว์เซอร์ ไม่ใช่การยืนยันตัวตน
สิทธิ์ AI/Connector จริงมาจากบัญชี claude.ai ที่ผู้เปิดดูล็อกอินอยู่

### Capabilities ที่ประกาศไว้

```json
{
  "downloads": true,
  "sample": {},
  "mcp": {
    "servers": [
      {"server": "Google Drive", "tools": ["search_files","read_file_content","create_file","trash_file"]},
      {"server": "Canva", "tools": ["generate-design","create-design-from-candidate","get-export-formats","export-design","upload-asset-from-url"]},
      {"server": "Moda - Slides and Designs", "tools": ["canvas_create","task_delegate","task_status","export","canvas_screenshot","canvas_read","canvas_apply_markup","upload"]},
      {"server": "Gmail", "tools": ["create_draft","send_message","update_draft"]}
    ]
  }
}
```

รายการนี้ต้องตรงกับ tool ที่โค้ดเรียกใช้จริงเสมอ (เรียก tool ที่ไม่อยู่ในรายการจะได้ `not_in_manifest`)

## ฟีเจอร์หลัก

- อัปโหลด TOR จากเครื่องหรือเลือกจาก Google Drive (OCR อัตโนมัติสำหรับ PDF สแกน)
- TOR ยาวเกินขีดจำกัด 64KB ของ `sample` → แบ่งเอกสาร, สรุปทีละส่วน, แล้วรวมวิเคราะห์
- เก็บไฟล์ TOR ต้นฉบับขึ้น Google Drive อัตโนมัติ
- สร้างดีไซน์ผ่าน Canva AI / Moda AI พร้อมภาพ Preview (`canvas_screenshot`) และแทรกโลโก้อัตโนมัติ
- กล่องสรุปผล: เลือกขั้นตอน → ดาวน์โหลด Word/PDF หรือแนบเข้าฉบับร่างอีเมลเดียวกันแบบสะสม
- บีบอัดรูป/PDF แบบขั้นบันได กันไฟล์เกิน 1MB ต่อ tool call
- Template อีเมลแบบหนังสือราชการ

## ปัญหาค้าง: ผู้ใช้บางคนใช้ AI / Connector ไม่ได้

อาการ: ขึ้นข้อความ "ไม่สามารถเชื่อมต่อความสามารถของ AI ในหน้านี้ได้" และ Error ไม่มี `.code`

### สิ่งที่ลองแล้ว (ไม่หาย)

1. ตรวจแล้วว่าเชื่อมต่อ Google Drive ใน Settings → Connectors ถูกต้อง
2. แก้ `getSample()` / `getMcp()` ให้ไม่กลืน Error จริง — แต่ Error ที่ได้ยังไม่มี code
3. เปิด "Code execution and file creation" ใน Settings → Capabilities
4. Hard refresh, reconnect connector, log out/log in

### วินิจฉัยใหม่: ปัญหาอยู่ที่ "หน้าเปิดในที่ไหน" ไม่ใช่ "สิทธิ์ของบัญชี"

ข้อความนี้ขึ้นเฉพาะเมื่อ `await claude.use("sample")` คืนค่า **`null`** (ไม่ใช่ throw)
ตาม runtime contract ของ Artifact (0.2.52) ค่า `null` แปลว่า **"หน้านี้ในมุมมองนี้รัน capability ไม่ได้"**
ซึ่งเกิดเมื่อ:

- เปิดหน้าแบบ **top-level บนโดเมนของ Artifact เอง** (เช่นกด "เปิดในแท็บใหม่" / ลิงก์ตรงของไฟล์) — ทุก `use()` คืน `null`
- เปิดใน **เบราว์เซอร์ที่ไม่ใช่ claude.ai viewer** เช่น in-app browser ของ LINE / Facebook / Messenger
  หรือเบราว์เซอร์ที่ไม่ได้ล็อกอิน claude.ai — รอ ~10 วินาทีแล้วได้ `null`
- ไฟล์ที่บันทึกไว้ในเครื่อง หรือแอป Claude เวอร์ชันเก่าที่ยังไม่รองรับ capability

ส่วนปัญหาเรื่องสิทธิ์หรือการตั้งค่าบัญชี **จะไม่ทำให้ได้ `null`** แต่จะมาเป็น Error ที่มี code
ตอนเรียกใช้ครั้งแรก เช่น `not_granted`, `sampling_disabled`, `blocked_by_policy`
เพราะฉะนั้นการเปิด/ปิด Capabilities หรือ reconnect connector จึงไม่มีผลกับอาการนี้

### สิ่งที่ต้องให้ผู้ใช้ลองก่อนติดต่อ Support

1. คัดลอกลิงก์ `https://claude.ai/artifact/...` ไปวางใน **Chrome / Edge / Safari ตัวจริง** (ไม่ใช่เปิดจาก LINE)
2. ต้อง **ล็อกอิน claude.ai ในเบราว์เซอร์นั้นด้วยบัญชีตัวเอง** ก่อนเปิดลิงก์
3. ใช้หน้าแอปภายในหน้า claude.ai (มีแถบเครื่องมือของ Claude อยู่รอบๆ) — ไม่กด "เปิดในแท็บใหม่"
4. เมื่อกดปุ่ม AI ครั้งแรก ต้องกด **อนุญาต (Allow)** ในหน้าต่างขอสิทธิ์ของ Claude
5. ถ้าใช้แอป Claude บนมือถือ/เดสก์ท็อป ให้อัปเดตเป็นเวอร์ชันล่าสุด

ถ้าทำครบแล้วยังได้ข้อความเดิม ให้ส่ง Support พร้อมระบุว่า `claude.use("sample")` คืนค่า `null`
ขณะเปิดใน claude.ai viewer ที่ล็อกอินแล้ว (ระบุเบราว์เซอร์ อุปกรณ์ และเวลาที่เกิด)

### แนะนำให้แก้ในแอป

แยกกรณี `null` ออกจาก Error อื่น แล้วบอกผู้ใช้ให้ชัดว่า "กรุณาเปิดลิงก์นี้ในเบราว์เซอร์ที่ล็อกอิน
claude.ai แล้ว (ไม่ใช่เปิดจาก LINE / แท็บแยก)" แทนข้อความ "ลองใหม่ภายหลัง" ที่ใช้อยู่ตอนนี้
และข้อความบนหน้า Login ที่บอกว่าต้องเปิด "Code execution and file creation" ไม่น่าจะเป็นสาเหตุ
ของอาการนี้ ควรปรับให้ตรงกัน

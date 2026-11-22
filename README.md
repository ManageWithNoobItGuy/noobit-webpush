# noobit-webpush

Agent Skill สำหรับต่อยอดโปรเจกต์ Web App ที่มีอยู่ ให้รองรับระบบแจ้งเตือน **Web Push Notification (RFC 8291 `aes128gcm` + VAPID RFC 8292)** ที่ใช้งานได้จริงข้ามแพลตฟอร์ม ทั้ง **Chrome (FCM)**, **Safari บน macOS** และ **iOS Safari / PWA (`web.push.apple.com`)**

---

## สกิลนี้มีไว้ทำอะไร?

เวลาสร้างระบบแจ้งเตือน Web Push ใน Web App มักเจอปัญหาหลัก 2 อย่าง:
1. ส่งเข้า Chrome บนคอมพิวเตอร์ได้ แต่ **ไม่เตือนบน iPhone / Safari** เพราะ Apple Push (`web.push.apple.com`) ปฏิเสธค่า `sub` ใน VAPID JWT ที่ใช้ `.local` หรือ `localhost`
2. API ส่งหรือทดสอบแจ้งเตือนคืนค่า `200 OK` แม้ส่งออกจริง `0` เครื่อง ทำให้ไม่รู้ว่า Push Service ปฏิเสธด้วย error อะไร

**`noobit-webpush`** จะช่วยวางระบบ Web Push ตั้งแต่ต้นจนจบโดยป้องกันปัญหาเหล่านี้ให้ครบ:
- สอบถามความต้องการก่อนลงมือทำ (เหตุการณ์ที่จะแจ้งเตือน, Role ผู้ส่ง → Role ผู้รับ, ข้อความแจ้งเตือน) หรือช่วยวิเคราะห์โค้ดแล้วเสนอให้เลือกปรับแก้
- บังคับใช้ `sub` เป็นอีเมลโดเมนจริงหรือ URL ของระบบจริง (ห้ามใช้ `.local` / `localhost`)
- บังคับตรวจสอบ HTTP status และ error body จาก Push Service จริง พร้อมลบ subscription ที่หมดอายุ (`404` / `410`) อัตโนมัติ
- สร้าง Service Worker (`sw.js`), API ลงทะเบียน/ยกเลิก/ทดสอบ, และ UI สำหรับเปิดรับแจ้งเตือน
- หลังติดตั้งเสร็จ จะถามอุปกรณ์ที่ผู้ใช้ต้องการใช้งาน และแสดงคู่มือตั้งค่าเฉพาะอุปกรณ์ (iPhone Add to Home Screen, Android Chrome, Android Samsung Internet, และ Computer)

---

## วิธีติดตั้งสกิล (Installation)

คัดลอกโฟลเดอร์สกิลนี้ไปไว้ในโฟลเดอร์ skills ของ AI Agent ที่คุณใช้งาน เช่น:

### สำหรับ Antigravity (Global Skill)
```bash
mkdir -p ~/.gemini/config/skills
git clone https://github.com/<your-username>/noobit-webpush.git ~/.gemini/config/skills/noobit-webpush
```

### สำหรับใช้งานเฉพาะโปรเจกต์ (Workspace Skill)
```bash
mkdir -p .gemini/skills
git clone https://github.com/<your-username>/noobit-webpush.git .gemini/skills/noobit-webpush
```

*(หลังติดตั้งแล้ว ให้เริ่มแชทใหม่เพื่อให้ระบบโหลดสกิล `noobit-webpush` เข้ารายการ)*

---

## วิธีใช้งาน (How to Use)

เปิดโปรเจกต์ Web App ของคุณ แล้วพิมพ์สั่งงานโดยอ้างถึงสกิล `noobit-webpush` เช่น:

```text
ใช้สกิล noobit-webpush ช่วยเพิ่มระบบแจ้งเตือน Web Push ให้โปรเจกต์นี้หน่อย
```

หรือระบุความต้องการเบื้องต้นไปพร้อมกัน:

```text
ใช้ noobit-webpush เพิ่มแจ้งเตือนเมื่อพนักงานสร้างใบแจ้งซ่อม ให้ส่งแจ้งเตือนไปหา Admin
```

### ขั้นตอนการทำงานของสกิล
1. **สอบถามรายละเอียด (Interactive Discovery):** สกิลจะถามว่าต้องการแจ้งเตือนเหตุการณ์อะไร ส่งจาก Role ไหนไป Role ไหน และข้อความแบบใด (หรือให้สกิลช่วยวิเคราะห์โปรเจกต์แล้วเสนอให้เลือกก่อนก็ได้)
2. **พัฒนาและเชื่อมต่อระบบ:** สร้างตารางฐานข้อมูล, ตัวเข้ารหัส Web Push (RFC 8291 + VAPID RFC 8292), API endpoints, Service Worker และหน้า UI สำหรับเปิด/ทดสอบการแจ้งเตือน
3. **แนะนำการตั้งค่าตามอุปกรณ์:** ถามว่าต้องการรับแจ้งเตือนบนอุปกรณ์ใดบ้าง และแนะนำวิธี Add to Home Screen / เปิด Notification บน iPhone, Android (Chrome / Samsung Internet) และ Desktop

---

## โครงสร้างไฟล์

- `SKILL.md` — คำสั่งและข้อกำหนดของ Agent Skill `noobit-webpush`
- `README.md` — คู่มือแนะนำการใช้งานและการติดตั้งสกิล

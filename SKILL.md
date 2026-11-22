---
name: noobit-webpush
description: >-
  Add cross-browser Web Push notifications (RFC 8291 aes128gcm + VAPID RFC 8292)
  to an existing web app, supporting Chrome (FCM), macOS Safari, and iOS Safari/PWA
  (web.push.apple.com). Guides interactive discovery of trigger events, sender/target
  roles, notification copy, and device setup instructions (iPhone Add to Home Screen,
  Android Chrome, and Samsung Internet).
---

# Noobit Web Push Skill

Skill สำหรับต่อยอดโปรเจกต์ Web App ที่มีอยู่ ให้รองรับระบบแจ้งเตือน **Web Push Notification** ที่ทำงานได้จริงทั้งบน **Chrome (FCM)**, **Safari บน macOS** และ **iOS Safari / PWA (`web.push.apple.com`)**

---

## ขั้นตอนที่ 1: สอบถามความต้องการจากผู้ใช้ (Interactive Discovery)

ก่อนเริ่มแก้ไขโค้ด **ต้องสอบถามรายละเอียดต่อไปนี้กับผู้ใช้ก่อนเสมอ** (ใช้ `ask_question` หรือถามเป็นข้อสั้นๆ ชัดเจน):

1. **เหตุการณ์ที่จะให้แจ้งเตือน (Trigger Event)**
   - ต้องการให้แจ้งเตือนเมื่อเกิดเหตุการณ์อะไรในระบบ? (เช่น เมื่อมีการเบิกจ่ายสินค้า, แจ้งซ่อม, สร้างคำสั่งซื้อ, เปลี่ยนสถานะงาน ฯลฯ)
2. **สิทธิ์ผู้ส่งและผู้รับ (Sender Role → Target Role)**
   - แจ้งเตือนเมื่อ Role ใดเป็นผู้ทำรายการ? (เช่น `user`, `staff`, หรือทุก Role)
   - ส่งแจ้งเตือนไปยัง Role ใดบ้าง? (เช่น เฉพาะ `admin`, หรือ `manager` + `admin`)
3. **รูปแบบข้อความแจ้งเตือน (Title & Body)**
   - ต้องการให้หัวข้อและเนื้อหาแสดงข้อมูลอะไรบ้าง? (เช่น ชื่อผู้ทำรายการ, รายการสินค้า/งาน, จำนวน, สาเหตุ/หมายเหตุ)
   - **หมายเหตุ:** แจ้งผู้ใช้ด้วยว่า *สามารถให้ช่วยวิเคราะห์โค้ดของโปรเจกต์ปัจจุบัน แล้วเสนอตัวอย่าง Trigger, Role และข้อความแจ้งเตือนให้เลือก/ปรับแก้ก่อนได้*

หากผู้ใช้เลือกให้ช่วยคิด ให้สำรวจ route/schema/roles ของโปรเจกต์ก่อน แล้วเสนอตารางสรุป (Event / From Role → To Role / Title & Body) ให้ผู้ใช้ยืนยันหรือปรับแก้ก่อนลงมือเขียนโค้ด

---

## ขั้นตอนที่ 2: ข้อกำหนดทางเทคนิค (บังคับใช้ทุกโปรเจกต์)

ต้องปฏิบัติตามมาตรฐาน **RFC 8291 (`aes128gcm`)** และ **VAPID (RFC 8292)** พร้อมเงื่อนไขสำคัญเพื่อให้ผ่านทั้ง Google FCM และ Apple Push (`web.push.apple.com`):

### 1. VAPID JWT (`sub`) — ห้ามใช้ `.local` หรือ `localhost`
- ค่า `sub` ใน VAPID JWT payload **ต้องใช้อีเมลโดเมนจริงหรือ URL ของระบบจริง** เช่น `mailto:admin@<your-domain>` หรือ `https://<your-domain>`
- **ห้ามใช้ `.local` หรือ `localhost` เด็ดขาด** เพราะ Apple Push Service (`web.push.apple.com`) ตรวจสอบความถูกต้องของโดเมนอย่างเข้มงวดและจะปฏิเสธคำขอทันที (ในขณะที่ Chrome/FCM อาจยอมรับ ทำให้เกิดอาการเตือนเฉพาะคอมพิวเตอร์แต่ไม่เตือนบน iPhone)

### 2. การเข้ารหัสและ Header (RFC 8291 + RFC 8292)
- ใช้ Web Crypto API (`ECDSA P-256` สำหรับ VAPID JWT `ES256`, `ECDH P-256` + `HKDF-SHA256` + `AES-128-GCM` สำหรับ `aes128gcm`) หรือไลบรารีที่ได้มาตรฐาน
- ส่ง HTTP Headers ครบถ้วน:
  - `Authorization: vapid t=<jwt>, k=<base64url(65B public key)>`
  - `Content-Encoding: aes128gcm`
  - `Content-Type: application/octet-stream`
  - `TTL: 86400`
  - `Urgency: high`

### 3. Error Handling & Subscription Cleanup (ห้ามกลบ Error)
- ปุ่มทดสอบ (`/api/push/test`) และฟังก์ชันส่งแจ้งเตือน **ต้องตรวจสอบ HTTP status และอ่าน response body จาก Push Service เสมอ**
- **ห้ามคืนสถานะสำเร็จ (200 OK / Success) หากส่งสำเร็จ `0` เครื่อง** — ต้องคืน HTTP error (เช่น `502`) พร้อมรายละเอียด status และข้อความจาก Push Service เพื่อดีบักได้ทันที
- หาก Push Service ตอบกลับ `404` หรือ `410` (Subscription หมดอายุ/ถูกยกเลิก) **ต้องลบ subscription นั้นออกจากฐานข้อมูลอัตโนมัติ**

### 4. โครงสร้างที่ต้องสร้าง/เชื่อมต่อในโปรเจกต์
- **Database**: ตาราง `push_subscriptions` (`user_id`, `endpoint` UNIQUE, `p256dh`, `auth`, `user_agent`, `created_at`, `updated_at`)
- **Backend API** (จำกัดสิทธิ์เฉพาะ Role เป้าหมายที่รับแจ้งเตือน):
  - `GET /api/push/status` — คืน `publicKey`, จำนวนอุปกรณ์ที่ลงทะเบียน, และสถานะของ endpoint ปัจจุบัน
  - `POST /api/push/subscribe` — บันทึก/อัปเดต subscription (ตรวจสอบ `https:` endpoint)
  - `POST /api/push/unsubscribe` — ยกเลิก subscription ของอุปกรณ์
  - `POST /api/push/test` — ทดสอบส่งแจ้งเตือนไปยังอุปกรณ์ของ user ปัจจุบัน พร้อมรายงานผลจริง
- **Trigger Hook**: เรียกส่งแจ้งเตือนแบบไม่บล็อก response (เช่น `executionCtx.waitUntil(...)` บน Cloudflare Workers) เมื่อเกิดเหตุการณ์จาก Role ที่กำหนด
- **Frontend & Service Worker**:
  - `public/sw.js` รองรับ `push` และ `notificationclick`
  - ตรวจสอบ `manifest.webmanifest` / meta tags สำหรับ PWA (`display: standalone`) เพื่อให้ iOS และ Android Add to Home Screen ทำงานได้สมบูรณ์
  - UI สำหรับ Role เป้าหมายในการกด **เปิด/ปิดรับแจ้งเตือน** และ **ทดสอบแจ้งเตือน**

---

## ขั้นตอนที่ 3: สอบถามอุปกรณ์และแนะนำวิธีติดตั้งหลังทำเสร็จ

เมื่อพัฒนาและทดสอบระบบเสร็จแล้ว ให้ **ถามผู้ใช้ว่าต้องการรับการแจ้งเตือนบนอุปกรณ์ใดบ้าง** (เช่น iPhone/iOS, Android, หรือ Computer) แล้วแสดงคู่มือการตั้งค่าเฉพาะอุปกรณ์ที่ผู้ใช้ต้องการตามแนวทางด้านล่าง:

### 📱 กรณี iPhone / iPad (iOS Safari / PWA)
> บน iOS ระบบ Web Push จะทำงานผ่าน **แอปบนหน้าจอโฮม (Home Screen Web App)** เท่านั้น (ต้องเป็น iOS 16.4 ขึ้นไป)
1. เปิดเว็บแอปด้วยเบราว์เซอร์ **Safari** (ห้ามใช้เบราว์เซอร์อื่นในการ Add to Home Screen)
2. แตะปุ่ม **แชร์ (Share)** (ไอคอนสี่เหลี่ยมมีลูกศรชี้ขึ้น)
3. เลือก **"เพิ่มไปยังหน้าจอโฮม" (Add to Home Screen)**
4. แตะ **"เพิ่ม" (Add)** เพื่อสร้างไอคอนแอปบนหน้าจอหลัก
5. **ปิด Safari แล้วเปิดเว็บแอปจากไอคอนบนหน้าจอโฮมแทน**
6. เข้าสู่ระบบด้วยบัญชีที่มีสิทธิ์รับแจ้งเตือน → กด **"เปิดรับการแจ้งเตือน"** → กด **อนุญาต (Allow)** → ลองกด **"ทดสอบแจ้งเตือน"**

### 🤖 กรณี Android

#### Chrome
1. เปิด **Chrome** แล้วเข้า Web App ของเรา
2. แตะเมนู **`⋮` (จุดสามจุด)** มุมขวาบน
3. เลือก **"ติดตั้งแอป" (Install app)** หรือ **"เพิ่มลงในหน้าจอหลัก" (Add to Home screen)** *(ชื่อเมนูอาจต่างกันตามรุ่น Chrome)*
4. แตะ **"ติดตั้ง"** หรือ **"เพิ่ม"** เพื่อยืนยัน
5. จะได้ไอคอนของแอป *(ปรับชื่อและไอคอนตามแอปที่กำลังพัฒนา)* บนหน้าจอหลัก
6. *(บางครั้ง Chrome จะเด้งแถบ "เพิ่ม <ชื่อแอป> ลงในหน้าจอหลัก" ที่ด้านล่างให้เอง ก็กดจากแถบนั้นได้เลย)*
7. เปิดแอปแล้วกด **"เปิดรับการแจ้งเตือน"** → กด **อนุญาต (Allow)**

#### Samsung Internet (เบราว์เซอร์ของ Samsung)
1. เปิดเว็บใน **Samsung Internet**
2. แตะเมนู **`☰` (สามขีด)** ด้านล่างขวา
3. เลือก **"เพิ่มหน้าไปที่" (Add page to)** แล้วเลือก **"หน้าจอหลัก" (Home screen)**
4. แตะ **"เพิ่ม"**
5. เปิดแอปจากหน้าจอหลัก แล้วกด **"เปิดรับการแจ้งเตือน"** → กด **อนุญาต (Allow)**

### 💻 กรณี Computer (macOS / Windows)
- **Chrome / Edge**: เปิดเว็บ → เข้าสู่ระบบด้วย Role เป้าหมาย → กด **"เปิดรับการแจ้งเตือน"** → กด **Allow / อนุญาต**
- **Safari บน macOS**: เปิดเว็บ → กด **"เปิดรับการแจ้งเตือน"** → กด **Allow / อนุญาต** (ทำงานผ่าน Apple Push `web.push.apple.com` เช่นเดียวกับ iOS)

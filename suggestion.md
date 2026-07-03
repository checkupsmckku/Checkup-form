# Suggestion: ยืนยันผลการบันทึกลง Google Sheet ก่อนแสดง Modal สำเร็จ

## ปัญหาปัจจุบัน

ไฟล์: `survey.html` (บรรทัด ~4688-4700)

เมื่อผู้ใช้กดส่งฟอร์ม โค้ดจะยิง `fetch` ไปยัง Google Apps Script (`APPS_SCRIPT_URL`) แบบ **ไม่รอผลลัพธ์ (fire-and-forget)** แล้วเรียก `showSuccessModal(...)` ทันที โดยไม่ตรวจสอบว่าการบันทึกลง Google Sheet สำเร็จจริงหรือไม่

```js
fetch(APPS_SCRIPT_URL, {
  method: 'POST',
  body: JSON.stringify(formData),
  headers: { 'Content-Type': 'text/plain' }
})
.then(r => r.json())
.then(res => {
  if (res.status !== 'ok') console.warn('Google Sheet error:', res.message);
})
.catch(err => console.warn('Google Sheet fetch error:', err));

// ... แสดง success modal ทันที ไม่รอผลจาก fetch ด้านบน
showSuccessModal({ ... });
```

### ผลกระทบ

- ถ้า fetch ล้มเหลว (เน็ตหลุด, Apps Script error, quota เกิน, CORS ปัญหา) ผู้ใช้จะยังเห็นหน้า "ส่งสำเร็จ" เหมือนเดิม เพราะ error ถูกซ่อนไว้แค่ `console.warn`
- ข้อมูลผู้ใช้อาจหายไปโดยไม่มีใครรู้ตัว ไม่มีการแจ้งเตือนหรือทางสำรอง (retry / บันทึก local)

## แนวทางแก้ไข (เสนอ)

1. **รอผลลัพธ์จริงก่อนแสดง success modal**
   - เปลี่ยน handler ของ submit เป็น `async function` แล้ว `await fetch(...)` เหมือนที่ `doStaffSearch()` ทำอยู่แล้ว (บรรทัด ~5648-5665) เป็นตัวอย่างที่ดี
   - ถ้า response `status !== 'ok'` หรือ fetch throw error → แสดง modal/ข้อความ error แทน ไม่ใช่ success modal

2. **เพิ่ม retry หรือ fallback เมื่อส่งไม่สำเร็จ**
   - เช่น เก็บ `formData` ไว้ใน `localStorage` ชั่วคราว ถ้าส่งไม่สำเร็จ ให้ปุ่ม "ลองส่งอีกครั้ง"
   - หรือแสดงข้อความชัดเจนให้ผู้ใช้ติดต่อเจ้าหน้าที่/ถ่ายภาพหน้าจอไว้เป็นหลักฐาน

3. **แสดง loading state ระหว่างรอผลลัพธ์**
   - ปุ่ม submit ควร disable + แสดง "⏳ กำลังบันทึกข้อมูล..." ระหว่างรอ fetch (เหมือน pattern ใน `doStaffSearch`)

4. **Timeout handling**
   - ตั้ง timeout (เช่น 10-15 วินาที) สำหรับ fetch เผื่อ Apps Script ตอบช้าเกินไป จะได้ไม่ค้างหน้าจอเปล่าๆ

## หมายเหตุ

- โค้ด pattern ที่ทำถูกต้องอยู่แล้วในไฟล์เดียวกัน: ฟังก์ชัน `doStaffSearch()` (บรรทัด ~5648) ใช้ `await fetch()` และเช็ค `json.status !== 'ok'` ก่อนแสดงผล — ใช้เป็นต้นแบบได้เลย

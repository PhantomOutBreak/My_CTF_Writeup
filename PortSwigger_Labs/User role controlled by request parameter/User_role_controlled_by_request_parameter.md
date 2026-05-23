# User role controlled by request parameter

**Platform:** PortSwigger Web Security Academy

![Challenge Info](<Screenshot 2026-05-23 at 07-40-45 Server-side vulnerabilities - PortSwigger.png>)

## Description

This lab has an admin panel at `/admin`, which identifies administrators using a forgeable cookie.

Solve the lab by accessing the admin panel and using it to delete the user `carlos`.

You can log in to your own account using the following credentials: `wiener:peter`

## Solution

1. เข้าสู่ระบบด้วยบัญชีที่โจทย์กำหนดมาให้ (`wiener:peter`) ผ่านหน้า Login ของเว็บไซต์

   ![หน้า Login](<Screenshot 2026-05-23 at 07-42-13 User role controlled by request parameter.png>)
   ![เข้าสู่ระบบสำเร็จ](<Screenshot 2026-05-23 at 07-42-55 User role controlled by request parameter.png>)

2. เปิด **Burp Suite** แล้วไปที่ **Proxy → HTTP History** เพื่อตรวจสอบ Request/Response ทั้งหมดที่ Proxy ของ Burp จับได้

   ![HTTP History](<Screenshot_2026-05-23_07-45-08.png>)

   ไปดูที่ Request ฝั่ง Login จะเห็นได้ว่ามี Cookie ชื่อ `Admin` ถูกตั้งค่าเป็น `false` ซึ่งบ่งบอกว่า Server ใช้ Cookie ตัวนี้ในการระบุว่าผู้ใช้เป็น Admin หรือไม่ จึงลองไปดูที่ Request ฝั่ง My Account ด้วย

   ![Cookie Admin=false](<Screenshot_2026-05-23_07-46-03.png>)

3. คลิกขวาที่ Request ของหน้า My Account แล้วเลือก **Send to Repeater** เพื่อส่งไปยัง **Repeater** สำหรับทดสอบแก้ไข Request จากนั้นกด **Send** เพื่อยิง Request ครั้งแรกตามเดิม

   ![Send to Repeater](<Screenshot_2026-05-23_07-50-41.png>)

   ดู Response ที่ได้กลับมา จะเห็นว่าไม่มีอะไรที่เกี่ยวข้องกับ Admin Panel แสดงว่าสิทธิ์ยังเป็นผู้ใช้ทั่วไปอยู่

   ![Response ปกติ](<Screenshot_2026-05-23_07-51-01.png>)

4. แก้ไขค่าใน Cookie Header โดยเปลี่ยน `Admin=false` เป็น `Admin=true` แล้วกด **Send** อีกครั้ง จะเห็นว่า Response มีการเปลี่ยนแปลง โดยปรากฏ Link ของ **Admin Panel** ขึ้นมาใน HTML ที่ได้รับกลับมา

   ![แก้ Cookie เป็น Admin=true](<Screenshot_2026-05-23_07-51-31.png>)
   ![Response มี Admin Panel](<Screenshot_2026-05-23_07-51-54.png>)

5. กลับไปที่หน้าเว็บในบราวเซอร์ เปิด **Developer Tools** (กด `F12`) แล้วไปที่แท็บ **Application → Cookies** เพื่อดูคุกกี้ที่บราวเซอร์เก็บไว้ จะพบ Cookie `Admin=false` เช่นเดิม

   ![DevTools Cookies](<Screenshot_2026-05-23_07-52-29.png>)

   ดับเบิ้ลคลิกที่ค่าของ Cookie `Admin` แล้วเปลี่ยนจาก `false` เป็น `true` โดยตรง

   ![แก้ Cookie ใน DevTools](<Screenshot_2026-05-23_07-52-50.png>)

6. รีเฟรชหน้าเว็บ จะเห็นว่า Link **Admin Panel** ปรากฏขึ้นมาบนหน้าเว็บเนื่องจากตอนนี้บราวเซอร์ส่ง Cookie `Admin=true` ไปกับทุก Request แล้ว คลิกเข้าไปที่ Admin Panel

   ![Admin Panel ปรากฏขึ้น](<Screenshot_2026-05-23_07-53-15.png>)

   จะเห็นรายชื่อผู้ใช้ทั้งหมดในระบบ

   ![รายชื่อผู้ใช้ใน Admin Panel](<Screenshot_2026-05-23_07-53-35.png>)

7. กดปุ่ม **Delete** ที่แถวของผู้ใช้ `carlos` เพื่อลบออกจากระบบตามที่โจทย์กำหนด

   ![ลบ carlos สำเร็จ](<Screenshot_2026-05-23_07-53-50.png>)

## Summary

โจทย์ข้อนี้เป็นตัวอย่างของช่องโหว่ **Broken Access Control** ที่เกิดขึ้นเนื่องจาก Web Application เชื่อถือค่าใน Cookie ที่ส่งมาจากฝั่ง Client (`Admin=true/false`) ในการตัดสินใจว่าผู้ใช้มีสิทธิ์เข้าถึง Admin Panel หรือไม่ โดยไม่มีการตรวจสอบความถูกต้องบน Server-Side เลย ทำให้ผู้โจมตีสามารถแก้ไขค่า Cookie ได้เองอย่างง่ายดายผ่าน Burp Suite หรือ Browser DevTools

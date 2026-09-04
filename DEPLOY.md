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
   - สร้าง 3 users แยกกัน เพื่อให้เพิกถอนทีละตัวได้ถ้าหลุด:
     | Username | ใช้ที่ไหน | Permission |
     |---|---|---|
     | `boat` | ESP32 | Publish & Subscribe |
     | `nodered` | Node-RED | Publish & Subscribe |
     | `dashboard` | เว็บ | Publish & Subscribe |

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

## ส่วนที่ 2 — ขึ้นเว็บบน Cloudflare Pages

1. สมัคร https://dash.cloudflare.com (ฟรี)
2. **Workers & Pages** → Create → **Pages** → Connect to Git → เลือก repo `boat-dashboard`
3. ตั้งค่า build:
   - Framework preset: **None**
   - Build command: *(เว้นว่าง)*
   - Build output directory: `/`
4. Save and Deploy → ได้ URL `boat-dashboard.pages.dev` ทันที
5. ต่อจากนี้ `git push` ทุกครั้ง Cloudflare จะ deploy ให้เองอัตโนมัติ

### 2.1 ใส่โดเมนของตัวเอง
1. ซื้อโดเมนที่ Cloudflare Registrar (ขายราคาทุน ไม่บวกกำไร) หรือที่อื่นก็ได้
2. ใน Pages project → **Custom domains** → Set up a custom domain → พิมพ์โดเมน
3. ถ้าโดเมนอยู่ใน Cloudflare อยู่แล้ว มันจะตั้ง DNS + ออก HTTPS ให้อัตโนมัติภายในไม่กี่นาที

### 2.2 (ถ้าต้องการ) ใส่ระบบ login กันคนนอก
Cloudflare **Zero Trust** → Access → Applications → Add self-hosted application
- ใส่โดเมนของเว็บ
- Policy: Allow → Emails → ใส่อีเมลอาจารย์กับของเรา
- ฟรีถึง 50 users — คนอื่นเปิดลิงก์จะเจอหน้า login ก่อนเสมอ

---

## ส่วนที่ 3 — สิ่งที่ยังค้างอยู่

- [ ] เว็บยังไม่มี `@media` query เลย — เปิดบนมือถือ layout จะแน่น
- [ ] ข้อมูลไม่ได้ถูกเก็บลงฐานข้อมูล ปิดเว็บแล้วกราฟย้อนหลังหายหมด
- [ ] แท็บ "1 ชั่วโมง / 6 ชั่วโมง / 24 ชั่วโมง / 7 วัน" ยังกดไม่ทำงานจริง
- [ ] `boat/gps` ยังไม่มีหน้าแผนที่บนเว็บ (รอฮาร์ดแวร์ GPS)

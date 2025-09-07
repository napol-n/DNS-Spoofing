# 🔐 DNS Spoofing Lab - คู่มือสมบูรณ์

> **⚠️ เพื่อการศึกษาเท่านั้น**: Lab นี้ถูกออกแบบมาเพื่อการศึกษาด้านความปลอดภัยไซเบอร์และการทดสอบเจาะระบบที่ได้รับอนุญาต อย่านำเทคนิคเหล่านี้ไปใช้กับเครือข่ายที่คุณไม่ได้เป็นเจ้าของหรือไม่ได้รับอนุญาตอย่างชัดเจน

## 📋 สารบัญ
- [สภาพแวดล้อม Lab](#-สภาพแวดล้อม-lab)
- [การตั้งค่าเครือข่าย](#-การตั้งค่าเครือข่าย)
- [การตั้งค่า DNS Server](#-การตั้งค่า-dns-server)
- [การตั้งค่าเครื่องเหยื่อ](#-การตั้งค่าเครื่องเหยื่อ)
- [การวิเคราะห์ Traffic](#-การวิเคราะห์-traffic)
- [ผลลัพธ์ Lab](#-ผลลัพธ์-lab)
- [สรุปผล](#-สรุปผล)

---

## 🏗️ สภาพแวดล้อม Lab

### โครงสร้างเครือข่าย
```
┌─────────────────┐    DNS Query     ┌─────────────────┐
│   เครื่องเหยื่อ    │ ───────────────► │   เครื่องผู้โจมตี  │
│ 192.168.147.20  │                  │ 192.168.147.10  │
│   (max)         │ ◄─────────────── │   (vouce)       │
└─────────────────┘   Spoofed IP     └─────────────────┘
```

| ส่วนประกอบ | รายละเอียด |
|-----------|-----------|
| **IP ผู้โจมตี** | `192.168.147.10` (Kali Linux/Debian) |
| **IP เหยื่อ** | `192.168.147.20` (Linux Client) |
| **Gateway** | `192.168.147.1` (ตัวเลือก) |
| **โดเมนเป้าหมาย** | `example.com` |
| **เครือข่าย** | `192.168.147.0/24` |

---

## 🌐 การตั้งค่าเครือข่าย

<details>
<summary><strong>🔧 การตั้งค่าเครื่องผู้โจมตี (vouce)</strong></summary>

### ตรวจสอบ Network Interface
```bash
ip a show eth0
```

**ผลลัพธ์ที่คาดหวัง:**
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.147.10/24 scope global eth0
```

### ตั้งค่า Static IP (หากจำเป็น)
```bash
# กำหนด static IP หากยังไม่ได้กำหนด
sudo ip addr add 192.168.147.10/24 dev eth0
sudo ip link set eth0 up
```

### ตรวจสอบ Routing Table
```bash
ip route
```

**ผลลัพธ์ที่คาดหวัง:**
```
default via 192.168.147.1 dev eth0
192.168.147.0/24 dev eth0 proto kernel scope link src 192.168.147.10
```

</details>

<details>
<summary><strong>🎯 การตั้งค่าเครื่องเหยื่อ (max)</strong></summary>

### ตรวจสอบ Network Interface
```bash
ip a show eth0
```

**ผลลัพธ์ที่คาดหวัง:**
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.147.20/24 scope global eth0
```

### ตั้งค่าเครือข่าย (หากจำเป็น)
```bash
# กำหนด IP หากจำเป็น
sudo ip addr add 192.168.147.20/24 dev eth0
sudo ip link set eth0 up

# ตั้งค่า default gateway
sudo ip route add default via 192.168.147.1
```

### ตรวจสอบการตั้งค่า
```bash
ip a
ip route
```

**ผลลัพธ์ที่คาดหวัง:**
```
default via 192.168.147.1 dev eth0
192.168.147.0/24 dev eth0 proto kernel scope link src 192.168.147.20
```

</details>

---

## 🔧 การตั้งค่า DNS Server

### ติดตั้ง dnsmasq
```bash
sudo apt update
sudo apt install dnsmasq -y
```

### ตรวจสอบเวอร์ชัน
```bash
dnsmasq --version
```

**ผลลัพธ์:**
```
Dnsmasq version 2.91
```

<details>
<summary><strong>📝 ตั้งค่า dnsmasq</strong></summary>

### แก้ไขไฟล์ Configuration
```bash
sudo nano /etc/dnsmasq.conf
```

### เพิ่มการตั้งค่า
```conf
# ฟังเฉพาะ IP ของผู้โจมตี
listen-address=192.168.147.10

# bind เฉพาะ interface ที่ระบุ
bind-interfaces

# ปลอม example.com ให้ชี้ไป IP ผู้โจมตี
address=/example.com/192.168.147.10

# ไม่ query upstream DNS servers
no-resolv
```

**คำอธิบายการตั้งค่า:**
- `listen-address` - IP address ที่ dnsmasq จะฟัง
- `bind-interfaces` - bind เฉพาะ interface ที่ระบุ
- `address` - การ mapping โดเมนไป IP (กฎการปลอม)
- `no-resolv` - ไม่ query upstream DNS servers

</details>

### เริ่มต้น dnsmasq
```bash
# หยุด dnsmasq processes ที่มีอยู่
sudo pkill dnsmasq

# เริ่มต้น dnsmasq ด้วยการตั้งค่าการปลอม
sudo dnsmasq --no-daemon \
  --interface=eth0 \
  --bind-interfaces \
  --no-resolv \
  --address=/example.com/192.168.147.10
```

**ผลลัพธ์ที่คาดหวัง:**
```
dnsmasq: started, version 2.91 cachesize 150
dnsmasq: read /etc/hosts - 8 names
dnsmasq: warning: no upstream servers configured
```

### ตรวจสอบว่า dnsmasq ทำงาน
```bash
sudo ss -ulpn | grep :53
```

**ผลลัพธ์ที่คาดหวัง:**
```
UNCONN 0 0 127.0.0.1:53 0.0.0.0:* users:(("dnsmasq",pid=314782,fd=6))
UNCONN 0 0 192.168.147.10:53 0.0.0.0:* users:(("dnsmasq",pid=314782,fd=4))
UNCONN 0 0 [::1]:53 0.0.0.0:* users:(("dnsmasq",pid=314782,fd=8))
```

---

## 🎯 การตั้งค่าเครื่องเหยื่อ

### ตั้งค่า DNS Server
```bash
sudo nano /etc/resolv.conf
```

### เพิ่มเครื่องผู้โจมตีเป็น DNS Server
```conf
nameserver 192.168.147.10
```

### ทดสอบการแปลง DNS
```bash
dig example.com
```

**ผลลัพธ์ที่คาดหวัง:**
```
; <<>> DiG 9.20.9-1-Debian <<>> example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25879
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1
;; ANSWER SECTION:
example.com. 0 IN A 192.168.147.10
```

### ทดสอบการเชื่อมต่อ
```bash
ping example.com
```

**ผลลัพธ์ที่คาดหวัง:**
```
PING example.com (192.168.147.10) 56(84) bytes of data.
64 bytes from 192.168.147.10: icmp_seq=1 ttl=64 time=0.165 ms
```

---

## 📊 การวิเคราะห์ Traffic

### จับ DNS Traffic (บนเครื่องผู้โจมตี)
```bash
sudo tcpdump -i eth0 port 53 -vv
```

### ตัวอย่างผลลัพธ์ที่จับได้
```
09:00:01.123456 IP 192.168.147.20.53321 > 192.168.147.10.53: 12345+ A? example.com
09:00:01.123789 IP 192.168.147.10.53 > 192.168.147.20.53321: 12345 1/0/0 A 192.168.147.10
```

### การวิเคราะห์ Traffic Flow
1. **เครื่องเหยื่อ** ส่ง DNS query สำหรับ `example.com`
2. **เครื่องผู้โจมตี** ตอบกลับทันทีด้วย spoofed IP `192.168.147.10`
3. **เครื่องเหยื่อ** ได้รับ response ที่ถูกปลอมและเปลี่ยนเส้นทาง traffic ไปยังผู้โจมตี
4. **การโจมตีสำเร็จ** - traffic ทั้งหมดไปยัง `example.com` ถูกส่งไปยังผู้โจมตี

---

## 📈 ผลลัพธ์ Lab

<details>
<summary><strong>📋 ตารางสรุป</strong></summary>

| ขั้นตอน | เครื่องผู้โจมตี (vouce) | เครื่องเหยื่อ (max) |
|---------|----------------------|-------------------|
| **การตั้งค่า IP** | `192.168.147.10/24` | `192.168.147.20/24` |
| **Default Gateway** | `192.168.147.1` | `192.168.147.1` |
| **DNS Server** | `dnsmasq บน 192.168.147.10` | `nameserver 192.168.147.10` |
| **DNS Query สำหรับ example.com** | ตอบกลับ: `192.168.147.10` (ปลอม) | ได้รับ `192.168.147.10` |
| **การจับ Packet** | ✅ ใช่ (tcpdump บน port 53) | ❌ ไม่ |

</details>

### แผนภาพขั้นตอนการโจมตี
```
┌─────────────────────┐
│   เครื่องเหยื่อ        │
│ (192.168.147.20)    │
└─────────┬───────────┘
          │
          │ 1. DNS query: example.com?
          ▼
┌─────────────────────┐
│   เครื่องผู้โจมตี      │
│ (192.168.147.10)    │
│   dnsmasq ทำงาน     │
└─────────┬───────────┘
          │
          │ 2. ตอบกลับ: example.com → 192.168.147.10
          ▼
┌─────────────────────┐
│   เครื่องเหยื่อได้รับ   │
│   IP ที่ถูกปลอม      │
└─────────┬───────────┘
          │
          │ 3. ping/เชื่อมต่อไป 192.168.147.10
          ▼
┌─────────────────────┐
│   เครื่องผู้โจมตี      │
│   ได้รับ traffic      │
│   ที่ถูกเปลี่ยนเส้นทาง │
└─────────────────────┘
```

---

## 📝 สรุปผล

### 🎯 สิ่งที่เราสำเร็จ
- ✅ ตั้งค่า **เครื่องผู้โจมตี** ด้วย static IP และ `dnsmasq`
- ✅ ตั้งค่า **เครื่องเหยื่อ** ให้ใช้เครื่องผู้โจมตีเป็น DNS server
- ✅ **ปลอม DNS queries** สำหรับ `example.com` สำเร็จ
- ✅ **เปลี่ยนเส้นทาง traffic ของเหยื่อ** ไปยังเครื่องผู้โจมตี
- ✅ **จับและวิเคราะห์** DNS traffic

### 🔍 จุดเรียนรู้หลัก
1. **DNS spoofing** สามารถเปลี่ยนเส้นทาง traffic ที่ถูกต้องไปยัง server ที่เป็นอันตราย
2. **Evil Twin attacks** ใช้ประโยชน์จากการแปลง DNS เพื่อดักจับการสื่อสาร
3. **เครื่องมือตรวจสอบเครือข่าย** เช่น `tcpdump` ช่วยวิเคราะห์ traffic การโจมตี
4. **การตั้งค่า DNS ที่ถูกต้อง** มีความสำคัญต่อความปลอดภัยของเครือข่าย

### 🛡️ กลยุทธ์ป้องกัน
- ใช้ **DNS ที่เข้ารหัส** (DNS over HTTPS/TLS)
- ใช้งาน **DNS filtering** และการตรวจสอบ
- ใช้ **VPN connections** สำหรับการสื่อสารที่สำคัญ
- **ตรวจสอบความปลอดภัยเครือข่าย** อย่างสม่ำเสมอ
- **DNSSEC validation** เมื่อเป็นไปได้

### 🔬 ข้อมูลเทคนิค

#### คำสั่งสำคัญที่ใช้:
- `dnsmasq` - DNS server สำหรับปลอม domain
- `dig` - เครื่องมือทดสอบ DNS resolution
- `tcpdump` - เครื่องมือจับ network traffic
- `ip` - เครื่องมือจัดการ network configuration

#### ไฟล์สำคัญ:
- `/etc/dnsmasq.conf` - ไฟล์ config หลักของ dnsmasq
- `/etc/resolv.conf` - ไฟล์ config DNS ของระบบ

#### Ports ที่สำคัญ:
- `53/UDP` - DNS query port
- `53/TCP` - DNS zone transfer port

---

> **🔬 Lab เสร็จสมบูรณ์**: การโจมตี DNS Spoofing ได้รับการสาธิตเรียบร้อยแล้วโดยใช้เทคนิค dnsmasq Evil Twin ให้ใช้ความรู้นี้อย่างมีความรับผิดชอบและเฉพาะบนเครือข่ายที่คุณเป็นเจ้าของหรือได้รับอนุญาตอย่างชัดเจนในการทดสอบเท่านั้น

### 📚 แหล่งข้อมูลเพิ่มเติม
- [OWASP DNS Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/DNS_Security_Cheat_Sheet.html)
- [RFC 1035 - Domain Names Implementation](https://tools.ietf.org/html/rfc1035)
- [dnsmasq Official Documentation](http://www.thekelleys.org.uk/dnsmasq/doc.html)

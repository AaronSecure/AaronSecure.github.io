---
title: PWNLAB_INIT
date: 2026-03-17 10:00:00 +0300
categories: [Cybersecurity(Binary Exploitation)]
tags: [Integer Overflow to Buffer Overflow, Memory Corruption, Software Flaw]
---

## 1. Executive Summary

This report documents a complete end-to-end penetration test performed against the PwnLab: init 
vulnerable virtual machine a deliberately insecure Capture-The-Flag (CTF) target designed to 
simulate a real-world internal intranet image-sharing web application. The assessment was 
conducted from an attacking Kali Linux machine residing on the same host-only network segment. 
The engagement resulted in a full system compromise, progressing from unauthenticated web 
enumeration through Local File Inclusion (LFI) exploitation, PHP filter wrapper abuse, credential 
extraction from a MySQL database, web shell upload via MIME-type bypass, reverse shell 
establishment, and multi-stage lateral movement through three user accounts (www-data → kane 
→ mike) before achieving root-level access via a SUID binary exploitation technique. 
### Security Findings Summary

Total Findings: **7**

| Severity | Count |
|----------|-------|
| 🔴 CRITICAL | 3 |
| 🟠 HIGH | 3 |
| 🟡 MEDIUM | 1 |

#### Detailed Findings

| Category | Finding | Severity |
|----------|---------|----------|
| Web Application | Local File Inclusion (LFI) | 🔴 CRITICAL |
| Authentication | Hardcoded DB Credentials in PHP | 🟠 HIGH |
| File Upload | MIME-Type Bypass (PHP Shell) | 🔴 CRITICAL |
| Database | Weak MD5 Hashed Passwords | 🟠 HIGH |
| Privilege Escalation | SUID Binary (msg2root) | 🔴 CRITICAL |
| Privilege Escalation | Lateral Movement via su kane | 🟠 HIGH |
| OS Hardening | Outdated Apache 2.4.10 | 🟡 MEDIUM |
#### Lab Environment

- Attacker Machine: Kali Linux(192.168.56.104 )
- Target Machine: Debian GNU/Linux (192.168.56.107)
- Network: Oracle VirtualBox — Host-Only / NAT Network 
- Tools Used: Nmap, net discover, Nikto, Burp Suite, CyberChef, MySQL CLI, php-reverse-shell, Netcat (nc), Python pty, vim  
## 2. Scope and Environment
### 2.1 Network Topology

Both machines were hosted inside Oracle VirtualBox. The target machine was configured with a 
Host-Only Adapter (isolating it from the internet), while the attacking machine had two adapters 
one Host-Only for direct communication and one NAT for internet access. 
### Lab Network Configuration

| Machine | Role | IP Address | Network | Adapter Type |
|---------|------|------------|---------|--------------|
| Ethical-Hacker-Kali | 🟢 Attacker | 192.168.56.104 | 192.168.56.0/24 | Host-Only (eth1) + NAT (eth0) |
| PwnLab VM (vm) | 🔴 Target | 192.168.56.107 | 192.168.56.0/24 | Host-Only Adapter Only |

### 2.2 Tools Used 
| Tool | Purpose |
|------|---------|
| 🕵️ Nmap | Network discovery and port/service enumeration |
| 🔍 net discover | ARP-based host discovery to find target IP |
| 🛡️ Nikto | Web server vulnerability scanning |
| 🔐 Burp Suite | HTTP proxy — request interception and manipulation |
| 🧩 CyberChef | Base64 decoding of PHP source code and password hashes |
| 🗄️ MySQL CLI | Remote database access and credential harvesting |
| 💻 php-reverse-shell | PHP web shell to establish a reverse TCP connection |
| 📡 Netcat (nc) | TCP listener for catching the incoming reverse shell |
| 🐍 Python pty | TTY shell upgrade from dumb shell to interactive bash |
| ✏️ vim | Text editor to configure shell payload before upload |
# Procedure
## 3. Reconnaissance 
### 3.1 Host Discovery with net discover 
#### Step 1: Identify the Network (Reconnaissance) 
The attacker first confirmed their own IP address by running ifconfig on the Kali machine. The 
eth1 interface was assigned 192.168.56.104/24 on the Host-Only network. With the network range 
known, net discover was used to perform passive ARP scanning:
```bash
net discover
```
### Scanning for IP of pwnlab using net discover
![Scanning for IP of pwnlab using netdicover](/assets/img/network-scan/net_discover.png)
The scan returned three live hosts. The target was identified by its MAC vendor (PCS Systemtechnik GmbH the VirtualBox vendor signature) at IP 192.168.56.107: 
Note: The method I used to find the IP is by opening my attacking machine scan with netdiscover then after it finished scanning the existing IP I now turned on the pwnlab machine and the IP pop up.

### 3.2 Connectivity Check (Ping)
Before launching a full port scan, a standard ICMP ping test was performed to confirm the target 
was online and responsive: 
![nmap scan](/assets/img/network-scan/Ping_target_ip.png)
All four packets were returned with 0% packet loss, confirming the machine is alive and responsive.

## 4. Enumeration
### 4.1 Nmap Full Port Scan
A comprehensive Nmap scan was launched using aggressive service detection, version scanning, OS detection, and script scanning. Results were saved to a file for reference:
#### Command
```bash
nmap -sS -A -T4 -oN nmap-scan.txt 192.168.56.107
```
The scan revealed the following open ports and services:
### Nmap Scan Results - Open Ports

### Service Enumeration

| Port | Protocol | Service | Version | Details |
|------|----------|---------|---------|---------|
| **80/tcp** | TCP | HTTP | Apache 2.4.10 | - Web server running on Debian<br>- PwnLab Intranet Image Hosting<br>- Potential LFI, RFI, file upload vulnerabilities |
| **111/tcp** | TCP | rpcbind | RPC #100000 | - RPC service versions 2,3,4<br>- May expose NFS or other RPC services |
| **3306/tcp** | TCP | MySQL | MySQL 5.5.47 | - Remote database server<br>- Protocol version 10<br>- Thread ID: 52<br>- Potential weak credentials or SQL injection |
| **100024/udp** | UDP | status | RPC #100024 | - RPC status service<br>- Information disclosure |

### Attack Surface Analysis

| Service | Attack Vectors | Exploitation Potential |
|---------|---------------|----------------------|
| **HTTP (80/tcp)** | - LFI/RFI<br>- File upload bypass<br>- Directory traversal<br>- Outdated Apache exploits | 🔴 CRITICAL |
| **rpcbind (111/tcp)** | - RPC enumeration<br>- NFS exposure<br>- Information disclosure | 🟡 MEDIUM |
| **MySQL (3306/tcp)** | - Weak credentials<br>- SQL injection<br>- Remote database access<br>- Privilege escalation | 🔴 HIGH |
| **status (100024/udp)** | - Service enumeration<br>- Information leakage | 🟢 LOW |

![nmap scan](/assets/img/network-scan/nmap_scan.png)

### 4.2 Web Application Discovery (Browser Enumeration)
Port 80 was open, so the application was opened directly in the Firefox browser on the Kali machine. The site presented a simple intranet image hosting portal branded as "PWNLAB" with three navigation links: Home, Login, and Upload.
| Page / URL | Observation | Security Issue | Priority |
|------------|-------------|----------------|----------|
| http://192.168.56.107/ | Home page — message: 'Use this server to upload and share image files inside the intranet' | Information disclosure | 🟢 LOW |
| http://192.168.56.107/?page=login | Login form — Username and Password fields with Login button | Potential brute force, SQL injection | 🔴 HIGH |
| http://192.168.56.107/?page=upload | Upload form — 'You must be logged in' (authentication required) | Authentication bypass possible | 🔴 CRITICAL |
| http://192.168.56.107/upload/ | Upload directory listing — Apache directory indexing is ENABLED | Directory listing exposure | 🟠 HIGH |

![nmap scan](/assets/img/network-scan/pwn_homepage.png "Pwnlab Home page")

*Figure 9 Pwn login page*
![nmap scan](/assets/img/network-scan/pwn_loginpage.png "Pwn login page")

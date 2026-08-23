---
title: PWNLAB_INIT
date: 2026-03-17 10:00:00 +0300
categories: [Cybersecurity(Binary Exploitation)]
tags: [Integer Overflow to Buffer Overflow, Memory Corruption, Software Flaw]
---

## Executive Summary

This report documents a complete end-to-end penetration test performed against the PwnLab: init 
vulnerable virtual machine a deliberately insecure Capture-The-Flag (CTF) target designed to 
simulate a real-world internal intranet image-sharing web application. The assessment was 
conducted from an attacking Kali Linux machine residing on the same host-only network segment. 
The engagement resulted in a full system compromise, progressing from unauthenticated web 
enumeration through Local File Inclusion (LFI) exploitation, PHP filter wrapper abuse, credential 
extraction from a MySQL database, web shell upload via MIME-type bypass, reverse shell 
establishment, and multi-stage lateral movement through three user accounts (www-data → kane 
→ mike) before achieving root-level access via a SUID binary exploitation technique. 

## Lab Environment

- Attacker Machine: Kali Linux(192.168.56.104 )
- Target Machine: Debian GNU/Linux (192.168.56.107)
- Network: Oracle VirtualBox — Host-Only / NAT Network 
- Tools Used: Nmap, net discover, Nikto, Burp Suite, CyberChef, MySQL CLI, php-reverse-shell, Netcat (nc), Python pty, vim  

## Procedure
### Step 1: Identify the Network (Reconnaissance)
## First, Host Discovery with net discover  

The attacker first confirmed their own IP address by running ifconfig on the Kali machine. The 
eth1 interface was assigned 192.168.56.104/24 on the Host-Only network. With the network range 
known, net discover was used to perform passive ARP scanning:
```bash
net discover
```
## Scanning for IP of pwnlab using netdicover
![Scanning for IP of pwnlab using netdicover](/assets/img/network-scan/net_discover.png)

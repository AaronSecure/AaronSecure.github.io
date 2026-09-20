---
title: Kioptrix 4 — LigGoat Web Compromise
date: 2026-03-24 10:00:00 +0300
categories: [Cybersecurity, Web Exploitation]
tags: [kioptrix, sqli, mysql-udf, privilege-escalation, pentest]
image:
  path: /assets/img/kioptrix-4/KIOP4-LIGGOAT.jpg
  alt: KIOPTRIX 4 LIGGOAT
---

## 1. Executive Summary

This report documents a complete end-to-end penetration test performed against the Kioptrix Level 4 vulnerable virtual machine — a deliberately insecure Capture-The-Flag (CTF) target running Ubuntu with the **LigGoat** member login portal, Samba, and SSH. The assessment was conducted from an attacking Kali Linux machine on the same host-only network segment. The engagement resulted in a full system compromise, progressing from host discovery and service enumeration through SQL injection and insecure direct object reference (IDOR) on the web application, SSH access with a restricted **LigGoat Shell**, breakout to a normal Bash session, discovery of empty MySQL root credentials in PHP source, and privilege escalation to root using the pre-installed **MySQL UDF** `sys_exec` to add the low-privilege user to the `admin` group.

The attack surface began with reconnaissance that identified Apache 2.2.8, PHP 5.2.4, OpenSSH 4.7, and Samba 3.0.28a on host **KIOPTRIX4**. Web testing against the LigGoat login application yielded plaintext credentials for two users. SSH as `john` landed in a restricted shell that was bypassed with a Python `echo os.system('/bin/bash')` trick. Database access as MySQL root without a password enabled command execution through `lib_mysqludf_sys.so`, which was already loaded on the box.

### Security Findings Summary

Total Findings: **6**

| Severity | Count |
|----------|-------|
| 🔴 CRITICAL | 3 |
| 🟠 HIGH | 2 |
| 🟡 MEDIUM | 1 |

#### Detailed Findings

| Category | Finding | Severity |
|----------|---------|----------|
| Web Application | SQL injection in LigGoat login (`' or 1=1#`) | 🔴 CRITICAL |
| Web Application | IDOR on `member.php?username=` exposing passwords | 🔴 CRITICAL |
| Authentication | Hardcoded / weak credential handling in PHP | 🟠 HIGH |
| Database | MySQL root with empty password | 🔴 CRITICAL |
| Privilege Escalation | MySQL UDF `sys_exec` command execution | 🔴 CRITICAL |
| Access Control | Restricted LigGoat Shell bypass via `echo` | 🟠 HIGH |

#### Lab Environment

- Attacker machine: Kali Linux (`192.168.56.103`)
- Target machine: Kioptrix 4 / Ubuntu (`192.168.56.113`)
- Network: Oracle VirtualBox — Host-Only / NAT
- Date of assessment: 24 March 2026
- Tools used: Netdiscover, ping, Nmap, Firefox, Gobuster, SSH, MySQL CLI

## 2. Scope and Environment

### 2.1 Network Topology

Both machines were hosted inside Oracle VirtualBox. The target used a Host-Only adapter; the attacking machine used Host-Only for lab traffic and NAT for internet access.

### Lab Network Configuration

| Machine | Role | IP Address | Network | Adapter Type |
|---------|------|------------|---------|--------------|
| Ethical-Hacker-Kali | 🟢 Attacker | 192.168.56.103 | 192.168.56.0/24 | Host-Only (eth1) + NAT (eth0) |
| Kioptrix 4 VM | 🔴 Target | 192.168.56.113 | 192.168.56.0/24 | Host-Only Adapter Only |

### 2.2 Tools Used

| Tool | Purpose |
|------|---------|
| 🕵️ Netdiscover | ARP-based host discovery on the lab subnet |
| 📡 ping | Confirm target reachability before scanning |
| 🔍 Nmap | Port, service, and SMB script enumeration |
| 🌐 Firefox | Manual web application testing |
| 📂 Gobuster | Directory and path brute force on HTTP |
| 🔐 SSH | Remote access as compromised web users |
| 🗄️ MySQL CLI | Database access and UDF abuse for privilege escalation |

# Procedure

## 3. Reconnaissance

### 3.1 Host Discovery with Netdiscover

The attacker confirmed the host-only range, then used Netdiscover to find live hosts. The Kioptrix 4 target was identified by its VirtualBox MAC vendor (PCS Systemtechnik GmbH) at **192.168.56.113**.

![Host discovery with Netdiscover](/assets/img/kioptrix-4/netdiscover.png)
*Figure 1 — Netdiscover identifying 192.168.56.113 on the host-only network*

### 3.2 Connectivity Check (Ping)

A short ICMP ping confirmed the target was online before deeper scanning:

![Ping the target IP](/assets/img/kioptrix-4/ping-target.png)
*Figure 2 — Verifying connectivity to 192.168.56.113*

### 3.3 Attacker Network Configuration

`ip a s` on Kali showed **192.168.56.103/24** on `eth1` (host-only) and **10.0.2.15/24** on `eth0` (NAT).

![Attacker IP configuration](/assets/img/kioptrix-4/ip-address-show.png)
*Figure 3 — Kali host-only address used for reverse connections and SSH*

## 4. Enumeration

### 4.1 Nmap Service Scan

```bash
nmap -sV -sC -Pn 192.168.56.113 -oN kiop4.txt
```

### Service Enumeration

| Port | Protocol | Service | Version | Details |
|------|----------|---------|---------|---------|
| **22/tcp** | TCP | SSH | OpenSSH 4.7p1 Debian | Legacy key algorithms required from modern clients |
| **80/tcp** | TCP | HTTP | Apache 2.2.8 (PHP 5.2.4) | LigGoat member login web application |
| **139/tcp** | TCP | NetBIOS-SSN | Samba smbd 3.X–4.X | Hostname **KIOPTRIX4** |
| **445/tcp** | TCP | Microsoft-DS | Samba smbd 3.0.28a | Guest access; message signing disabled |

![Nmap scan of Kioptrix 4](/assets/img/kioptrix-4/nmap-scan.png)
*Figure 4 — Nmap results and SMB host scripts for KIOPTRIX4*

### Attack Surface Analysis

| Service | Attack Vectors | Exploitation Potential |
|---------|----------------|------------------------|
| **HTTP (80/tcp)** | SQLi, IDOR, directory brute force | 🔴 CRITICAL |
| **SSH (22/tcp)** | Credential reuse from web app | 🟠 HIGH |
| **Samba (139/445)** | Guest enumeration, legacy SMB issues | 🟡 MEDIUM |

### 4.2 Web Application Discovery

Port 80 served the **LigGoat** “Member Login” page (`index.php`).

![LigGoat login page](/assets/img/kioptrix-4/liggoat-login.png)
*Figure 5 — LigGoat secure login portal on port 80*

### 4.3 Directory Brute Force (Gobuster)

```bash
gobuster dir -u http://192.168.56.113 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Notable paths included `/member`, `/john`, `/robert`, and `/images`.

![Gobuster directory scan](/assets/img/kioptrix-4/gobuster-dir.png)
*Figure 6 — User-specific directories discovered under the web root*

## 5. Exploitation Phase 1 — Web Application

### 5.1 SQL Injection Login Bypass

The login form did not safely handle special characters. A classic tautology payload bypassed authentication:

```sql
' or 1=1#
```

Testing with username **john**:

![SQL injection payload for john](/assets/img/kioptrix-4/sqli-payload-john.png)
*Figure 7 — Preparing the SQL injection bypass for user john*

After a successful login, the application redirected to a member control panel that disclosed credentials in plain text:

![John member panel with plaintext password](/assets/img/kioptrix-4/member-panel-john.png)
*Figure 8 — IDOR / information disclosure reveals password MyNameIsJohn*

The same technique worked for **robert**:

![SQL injection payload for robert](/assets/img/kioptrix-4/sqli-payload-robert.png)
*Figure 9 — SQL injection bypass for user robert*

![Robert member panel with encoded password](/assets/img/kioptrix-4/member-panel-robert.png)
*Figure 10 — Robert’s panel exposes an encoded password string for offline decoding*

Direct access to `member.php?username=john` (and `robert`) without proper authorization checks confirmed **IDOR** on the member pages.

## 6. Exploitation Phase 2 — SSH and Shell Breakout

### 6.1 SSH as john

Using the recovered password and legacy SSH options for the old server key types:

```bash
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa john@192.168.56.113
```

The session dropped into **LigGoat Shell**, a restricted environment exposing only a small command set (`cd`, `echo`, `ls`, etc.).

![SSH login and LigGoat Shell escape](/assets/img/kioptrix-4/ssh-lshell-escape.png)
*Figure 11 — Breaking out of LigGoat Shell with echo os.system('/bin/bash')*

### 6.2 Web Root Enumeration

From an unrestricted Bash shell, `/var/www` contained the PHP login logic and a `database.sql` artefact. `checklogin.php` revealed MySQL credentials:

- **User:** `root`
- **Password:** *(empty)*
- **Database:** `members`

![checklogin.php database credentials](/assets/img/kioptrix-4/checklogin-db-creds.png)
*Figure 12 — Hardcoded database connection strings in checklogin.php*

## 7. Exploitation Phase 3 — MySQL and Privilege Escalation

### 7.1 MySQL Root Access

```bash
mysql -h localhost -u root
```

The server reported **MySQL 5.0.51a** on Ubuntu — an end-of-life version often paired with dangerous UDF plugins in training labs.

![MySQL root session](/assets/img/kioptrix-4/mysql-root-access.png)
*Figure 13 — Unauthenticated local MySQL root access from the john account*

### 7.2 MySQL UDF Command Execution

Querying `mysql.func` showed **`sys_exec`** and **`lib_mysqludf_sys_info`** from `lib_mysqludf_sys.so` — a User Defined Function that runs shell commands as the MySQL service user (typically root).

```sql
SELECT sys_exec('usermod -a -G admin john');
```

![MySQL UDF sys_exec privilege escalation](/assets/img/kioptrix-4/mysql-udf-sys-exec.png)
*Figure 14 — Adding john to the admin group through sys_exec*

### 7.3 Root Shell

Members of the **`admin`** group on this Ubuntu build could use `sudo`. After the group change:

```bash
sudo su
whoami
```

![sudo su to root](/assets/img/kioptrix-4/sudo-su-root.png)
*Figure 15 — Elevating from john to root via sudo*

## 8. Capturing the Flag

With root access, `/root/congrats.txt` confirmed full compromise of Kioptrix 4. The author notes that multiple paths to root exist on this VM — this write-up documents the **web → SSH → MySQL UDF** chain.

![Root flag in /root/congrats.txt](/assets/img/kioptrix-4/root-congrats.png)
*Figure 16 — Root flag and author notes from loneferret*

## 9. Full Attack Chain Summary

| Step | Phase | Action | Outcome |
|------|-------|--------|---------|
| 1 | Reconnaissance | Netdiscover on 192.168.56.0/24 | Target IP 192.168.56.113 identified |
| 2 | Enumeration | Nmap `-sV -sC` | HTTP, SSH, Samba enumerated |
| 3 | Enumeration | Gobuster on port 80 | `/john`, `/robert`, `/member` discovered |
| 4 | Exploitation | SQL injection on LigGoat login | Access to member panels |
| 5 | Exploitation | IDOR on `member.php` | Passwords for john and robert recovered |
| 6 | Exploitation | SSH as john + LigGoat Shell bypass | Interactive Bash on target |
| 7 | Post-Exploitation | Read `checklogin.php` | Empty MySQL root password found |
| 8 | Privilege Escalation | `sys_exec` via MySQL UDF | john added to admin group |
| 9 | Post-Exploitation | `sudo su` + read `/root/congrats.txt` | Root flag captured |

## 10. Vulnerabilities and Recommendations

### VULN-01: SQL Injection in Authentication

**Severity:** 🔴 CRITICAL  
**Location:** LigGoat login (`index.php` / `checklogin.php`)

Unsanitized input allowed authentication bypass with a tautology payload.

**Remediation:** Use parameterized queries (prepared statements). Never concatenate user input into SQL. Add server-side validation and a Web Application Firewall for legacy apps during migration.

### VULN-02: Insecure Direct Object Reference on Member Pages

**Severity:** 🔴 CRITICAL  
**Location:** `member.php?username=`

Any authenticated or bypassed session could view other users’ credentials by changing the username parameter.

**Remediation:** Enforce server-side session binding so users can only read their own account data. Remove password display from the UI entirely.

### VULN-03: MySQL Root Without Password

**Severity:** 🔴 CRITICAL  
**Location:** Local MySQL service

Any local user who could reach the socket could authenticate as database root.

**Remediation:** Set strong passwords for all database accounts, restrict MySQL to required hosts, and remove unused UDF libraries.

### VULN-04: MySQL UDF `sys_exec`

**Severity:** 🔴 CRITICAL  
**Location:** `lib_mysqludf_sys.so` in `mysql.func`

Command execution from SQL enabled full system privilege changes.

**Remediation:** Do not install sys_exec UDFs on production systems. Run MySQL with minimal privileges and monitor `mysql.func` for unauthorized functions.

### VULN-05: Restricted Shell Bypass

**Severity:** 🟠 HIGH  
**Location:** LigGoat Shell over SSH

The restricted shell could invoke `/bin/bash` through `echo os.system(...)`.

**Remediation:** Use a properly configured rbash or ForceCommand with no escape vectors. Prefer key-based SSH with command restrictions only when the wrapper is audited.

### VULN-06: Legacy Software Stack

**Severity:** 🟡 MEDIUM  
**Location:** Apache 2.2.8, PHP 5.2.4, OpenSSH 4.7, Samba 3.0.28a

End-of-life components expand the attack surface.

**Remediation:** Upgrade to supported releases, patch regularly, and segment lab networks from production.

## 11. Conclusion

Kioptrix 4 was fully compromised through a chain that started on the LigGoat web application and ended in root via MySQL UDF abuse — without relying on a kernel exploit. The lab highlights why input validation, secure session handling, database hardening, and safe SSH environments must work together. Fixing any one layer would have broken this particular path.

## References

- OWASP Foundation. (2021). *OWASP Top 10*. <https://owasp.org/www-project-top-ten/>
- Kioptrix series — <https://www.kioptrix.com/>
- National Institute of Standards and Technology. *NIST SP 800-115: Technical Guide to Information Security Testing and Assessment*

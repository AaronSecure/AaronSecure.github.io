---
title: Kioptrix 5 — FreeBSD Compromise
date: 2026-03-22 10:00:00 +0300
categories: [Cybersecurity, Web Exploitation]
tags: [kioptrix, freebsd, lfi, privilege-escalation, pentest]
image:
  path: /assets/img/kioptrix-5/KIOP5FREEBSD.jpg
  alt: KIPTRIX 5 FREEBSD
---

## 1. Executive Summary

This report documents a complete end-to-end penetration test performed against the Kioptrix Level 5 vulnerable virtual machine — a deliberately insecure Capture-The-Flag (CTF) target running FreeBSD. The assessment was conducted from an attacking Kali Linux machine on the same host-only network segment. The engagement resulted in a full system compromise, progressing from unauthenticated web enumeration through Local File Inclusion (LFI), log and application abuse, remote code execution, and privilege escalation to root using a known FreeBSD 9.0 kernel issue.

The attack surface started with reconnaissance that revealed an outdated **pChart 2.1.3** application hidden behind a default Apache page. A directory traversal issue in pChart allowed reading of system files, including Apache configuration. A second HTTP service on port 8080 exposed **phptax**. Combining those findings led to a low-privilege `www` foothold. Host enumeration identified **FreeBSD 9.0-RELEASE**, which is end-of-life. A public local privilege-escalation issue (mmap/ptrace family) was then used to obtain root and read the final flag.

### Security Findings Summary

Total Findings: **5**

| Severity | Count |
|----------|-------|
| 🔴 CRITICAL | 3 |
| 🟠 HIGH | 1 |
| 🟡 MEDIUM | 1 |

#### Detailed Findings

| Category | Finding | Severity |
|----------|---------|----------|
| Web Application | Local File Inclusion / directory traversal in pChart 2.1.3 | 🔴 CRITICAL |
| Web Application | Log / application chaining leading to remote code execution | 🔴 CRITICAL |
| Privilege Escalation | Unpatched FreeBSD 9.0 kernel (mmap/ptrace) | 🔴 CRITICAL |
| Web Application | Outdated pChart 2.1.3 and phptax | 🟠 HIGH |
| Access Control | User-Agent based restriction on port 8080 | 🟡 MEDIUM |

#### Lab Environment

- Attacker machine: Kali Linux (`192.168.56.104`)
- Target machine: Kioptrix 5 / FreeBSD (`192.168.56.112`)
- Network: Oracle VirtualBox — Host-Only / NAT
- Date of assessment: 22 March 2026
- Tools used: Netdiscover, Nmap, Firefox, Burp Suite, curl, Metasploit, Netcat, searchsploit

> Some screenshots in this write-up were captured during a repeat of the same lab on neighbouring host-only addresses (`.105` / `.109`). The primary target for this report is **192.168.56.112**.
{: .prompt-info }

## 2. Scope and Environment

### 2.1 Network Topology

Both machines were hosted inside Oracle VirtualBox. The target used a Host-Only adapter (isolated from the internet). The attacking machine had two adapters: Host-Only for lab traffic and NAT for internet access.

### Lab Network Configuration

| Machine | Role | IP Address | Adapter Type |
|---------|------|------------|--------------|
| Ethical-Hacker-Kali | 🟢 Attacker | 192.168.56.104 | Host-Only (eth1) + NAT (eth0) |
| Kioptrix 5 VM | 🔴 Target | 192.168.56.112 | Host-Only adapter only |

### 2.2 Tools Used

| Tool | Purpose |
|------|---------|
| 🕵️ Netdiscover | ARP-based host discovery |
| 🔍 Nmap | Port and service enumeration |
| 🌐 Web browser | Manual application review |
| 🔐 Burp Suite | HTTP interception and header testing |
| 📡 curl | Crafted HTTP requests |
| 🛡️ Metasploit | Application testing against phptax |
| 📡 Netcat | Listener for reverse connections |
| 📚 searchsploit | Local privilege-escalation research |

# Procedure

## 3. Reconnaissance

### 3.1 Host Discovery with Netdiscover

The attacker confirmed the host-only range, then used Netdiscover to find live hosts. The Kioptrix target was identified by its VirtualBox MAC vendor (PCS Systemtechnik GmbH) at **192.168.56.112**.

![Host discovery with Netdiscover](/assets/img/kioptrix-5/netdiscover.png)
*Figure 1 — Netdiscover identifying 192.168.56.112 on the host-only network*

### 3.2 Nmap Service Scan

A service-detection scan was run against the target:

```bash
nmap -sS -A -T4 -oN nmap-kiop.txt 192.168.56.112
```

### Service Enumeration

| Port | Protocol | Service | Version | Details |
|------|----------|---------|---------|---------|
| **22/tcp** | TCP | SSH | — | Closed |
| **80/tcp** | TCP | HTTP | Apache httpd 2.2.21 (FreeBSD) | PHP/5.3.8, OpenSSL, DAV |
| **8080/tcp** | TCP | HTTP | Apache httpd 2.2.21 (FreeBSD) | Second web root on a non-standard port |

OS fingerprinting suggested FreeBSD 9.x. Network distance was one hop.

![Nmap scan of Kioptrix 5](/assets/img/kioptrix-5/nmap-scan.png)
*Figure 2 — Nmap results for 192.168.56.112*

### Attack Surface Analysis

| Service | Attack Vectors | Exploitation Potential |
|---------|----------------|------------------------|
| **HTTP (80/tcp)** | Hidden application paths, outdated charting software, LFI | 🔴 CRITICAL |
| **HTTP (8080/tcp)** | Secondary app (phptax), weak access control | 🟠 HIGH |

## 4. Enumeration

### 4.1 Web Application Discovery (Port 80)

Opening `http://192.168.56.112/` showed only the default Apache **"It works!"** page.

![Default Apache page](/assets/img/kioptrix-5/homepage-it-works.png)
*Figure 3 — Default page on port 80*

Viewing the page source revealed a hidden HTML comment with a meta refresh to **pChart 2.1.3**:

![Page source showing pChart path](/assets/img/kioptrix-5/view-source-pchart.png)
*Figure 4 — Hidden refresh to /pChart2.1.3/*

Navigating to the examples folder loaded the pChart 2.1.3 demo UI (Release 2.1.3).

![pChart 2.1.3 examples](/assets/img/kioptrix-5/pchart-examples.png)
*Figure 5 — pChart 2.1.3 examples interface*

### 4.2 Web Application Discovery (Port 8080)

The same host on port 8080 initially returned **403 Forbidden**.

![Port 8080 forbidden](/assets/img/kioptrix-5/port-8080-forbidden.png)
*Figure 6 — Direct browser access to port 8080 denied*

## 5. Exploitation Phase 1 — Local File Inclusion

### 5.1 Identifying the LFI Vulnerability

Public documentation for pChart 2.1.3 describes a directory-traversal issue in the examples viewer (`Action=View` / `Script=`). That class of bug lets the web server read files outside the application directory.

![pChart directory traversal advisory](/assets/img/kioptrix-5/pchart-advisory.png)
*Figure 7 — Documented pChart examples traversal issue*

Using that examples endpoint, `/etc/passwd` was readable and confirmed a FreeBSD user database (including `www` and `root`).

![LFI reading /etc/passwd](/assets/img/kioptrix-5/lfi-passwd.png)
*Figure 8 — /etc/passwd disclosed through pChart*

### 5.2 Reading Apache Configuration

FreeBSD does not use the same Apache paths as Linux. Handbook documentation pointed to `/usr/local/etc/apache22/httpd.conf`.

![FreeBSD Apache handbook](/assets/img/kioptrix-5/freebsd-apache-docs.png)
*Figure 9 — FreeBSD Apache configuration path*

The same LFI read `httpd.conf` and confirmed listeners on **80** and **8080**, plus FreeBSD-specific log and document-root locations.

![LFI reading httpd.conf](/assets/img/kioptrix-5/lfi-httpd-conf.png)
*Figure 10 — Apache configuration disclosed through LFI*

## 6. Exploitation Phase 2 — phptax and Remote Code Execution

### 6.1 Bypassing the Port 8080 Restriction

Burp Suite showed that port 8080 rejected the default Firefox User-Agent. Changing the User-Agent to an older Mozilla 4.x string allowed the request through. That is not real authentication — it is a client header that anyone can spoof.

![Burp intercept of port 8080](/assets/img/kioptrix-5/burp-intercept.png)
*Figure 11 — Intercepted request still using a modern User-Agent*

![User-Agent changed in Burp](/assets/img/kioptrix-5/burp-user-agent.png)
*Figure 12 — User-Agent adjusted to gain access*

After the header change, directory listing exposed **phptax/**.

![phptax directory listing](/assets/img/kioptrix-5/phptax-directory.png)
*Figure 13 — Index of / showing phptax*

![phptax application](/assets/img/kioptrix-5/phptax-application.png)
*Figure 14 — phptax web application on port 8080*

### 6.2 Application Testing

phptax is a known outdated tax demo with a public Metasploit module (`exploit/multi/http/phptax_exec`). The module was loaded and pointed at the lab host on port 8080.

![Rapid7 module documentation](/assets/img/kioptrix-5/rapid7-module.png)
*Figure 15 — Public phptax module documentation*

![Metasploit module options](/assets/img/kioptrix-5/msfconsole-options.png)
*Figure 16 — Module options in msfconsole*

Initial runs failed until a payload and `LHOST` were set. Compatible payloads included several Unix reverse-shell options.

![Payload selection](/assets/img/kioptrix-5/msf-payloads.png)
*Figure 17 — Compatible payloads listed after a failed run*

![Setting listener options](/assets/img/kioptrix-5/msf-lhost.png)
*Figure 18 — Listener host still required before a session*

Command execution on the target confirmed identity **`www`** (`uid=80`).

![Command execution as www](/assets/img/kioptrix-5/command-execution-www.png)
*Figure 19 — Foothold confirmed as the www user*

A Netcat listener then caught a reverse shell. `uname -a` reported:

**FreeBSD kioptrix2014 9.0-RELEASE FreeBSD 9.0-RELEASE #0**

![Reverse shell and OS identification](/assets/img/kioptrix-5/reverse-shell-freebsd.png)
*Figure 20 — www shell on FreeBSD 9.0-RELEASE*

## 7. Exploitation Phase 3 — Privilege Escalation to Root

### 7.1 Kernel Research

`searchsploit` against FreeBSD 9.0 returned local privilege-escalation research, including the well-known mmap/ptrace class of issues on this end-of-life release.

![searchsploit results for FreeBSD 9.0](/assets/img/kioptrix-5/searchsploit-freebsd.png)
*Figure 21 — Local privilege-escalation research for FreeBSD 9.0*

### 7.2 Local Privilege Escalation

Exploit source matching those public issues was transferred to the target (visible under `/tmp` as `26368` / `28718` artefacts). Running the compiled local exploit from the `www` shell produced a new process with **root** privileges.

![Files in /tmp during privilege escalation](/assets/img/kioptrix-5/tmp-privesc-files.png)
*Figure 22 — Local exploit artefacts in /tmp*

## 8. Capturing the Flag

With root access, `/root/congrats.txt` confirmed full compromise of Kioptrix 5.

![Root flag in /root/congrats.txt](/assets/img/kioptrix-5/root-congrats.png)
*Figure 23 — Root flag and author notes in /root/congrats.txt*

The flag file also documents FreeBSD-specific lessons used during the assessment: Apache logs live under `/var/log/httpd-access.log`, and the default document root is `/usr/local/www/` rather than `/var/www/`.

## 9. Full Attack Chain Summary

| Step | Phase | Action | Outcome |
|------|-------|--------|---------|
| 1 | Reconnaissance | Netdiscover on 192.168.56.0/24 | Target IP 192.168.56.112 identified |
| 2 | Enumeration | Nmap service scan | Ports 80 and 8080 open; Apache on FreeBSD |
| 3 | Enumeration | View page source | Hidden pChart 2.1.3 path discovered |
| 4 | Exploitation | pChart examples directory traversal | `/etc/passwd` and `httpd.conf` disclosed |
| 5 | Exploitation | Burp User-Agent change on :8080 | phptax application reachable |
| 6 | Exploitation | Outdated phptax + reverse connection | Shell as `www` |
| 7 | Privilege Escalation | `uname -a` + searchsploit | FreeBSD 9.0-RELEASE identified |
| 8 | Privilege Escalation | Public mmap/ptrace-class local issue | Root shell |
| 9 | Post-Exploitation | Read `/root/congrats.txt` | Root flag captured |

## 10. Vulnerabilities and Recommendations

### VULN-01: Local File Inclusion / Directory Traversal in pChart

**Severity:** 🔴 CRITICAL  
**Location:** pChart 2.1.3 examples on port 80

The examples viewer used unsanitized input to include files. That allowed arbitrary file read and helped map FreeBSD Apache paths.

**Remediation:** Do not pass user input into file-read or include APIs. Allowlist paths. Remove example code from any exposed host. Replace or remove pChart 2.1.3.

### VULN-02: Remote Code Execution via Outdated Application Chain

**Severity:** 🔴 CRITICAL  
**Location:** phptax on port 8080, chained with file disclosure

An obsolete application plus weak access control produced command execution as `www`.

**Remediation:** Remove unused demo applications. Disable PHP execution where it is not required. Patch or replace third-party web apps. Do not rely on logs or demo scripts as an execution surface.

### VULN-03: Unpatched FreeBSD 9.0 Kernel

**Severity:** 🔴 CRITICAL  
**Location:** FreeBSD 9.0-RELEASE

The OS is end-of-life and has public local privilege-escalation research (mmap/ptrace family). A low-privilege web user could become root.

**Remediation:** Upgrade to a supported FreeBSD release and apply a regular patch cycle. Limit what the web user can write and execute.

### VULN-04: Outdated Software Inventory (pChart, phptax)

**Severity:** 🟠 HIGH

Unsupported charting and tax-demo software expanded the attack surface with well-documented issues.

**Remediation:** Maintain a software inventory. Remove packages that are not required. Patch remaining components.

### VULN-05: User-Agent Based Access Control

**Severity:** 🟡 MEDIUM  
**Location:** HTTP service on port 8080

Access depended on a spoofable `User-Agent` header.

**Remediation:** Use real authentication and authorization. Never treat client headers as a security boundary.

## 11. Conclusion

Kioptrix 5 was fully compromised along a linear path: hidden outdated software on port 80, file disclosure through pChart, a second app on port 8080, a `www` foothold, then root via an unpatched FreeBSD 9.0 kernel. The lab shows why defense in depth matters. A modern posture would require patching, removing demo apps, and not treating client headers as access control.

## References

- OWASP Foundation. (2021). *OWASP Top 10*. <https://owasp.org/www-project-top-ten/>
- MITRE / Exploit-DB public research for FreeBSD 9.0 local privilege escalation (mmap/ptrace class)
- National Institute of Standards and Technology. *NIST SP 800-115: Technical Guide to Information Security Testing and Assessment*

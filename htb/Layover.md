
# Hack The Box — Layover

<p align="center">
  <img src="images/layover.png" width="850">
</p>

<p align="center">
  <strong>Hack The Box — Layover</strong><br>
  RDP → Internal Network → Wi-Fi Credential Capture → Craft CMS → RCE → SSH → CUPS LPE → Root
</p>

<p align="center">
  <a href="https://github.com/jaisurya93945">GitHub</a> •
  <a href="https://www.linkedin.com/in/badathala-jaisurya/">LinkedIn</a> •
  <a href="https://labs.hackthebox.com/achievement/machine/1830127/984">HTB Achievement</a>
</p>

## Overview

Layover is a Linux machine from Hack The Box that combines several attack surfaces into a single attack chain.

The initial foothold comes through RDP access to a workstation. From there, investigating the internal wireless environment leads to credentials for another user.

Those credentials provide access to an internal portal running Craft CMS. The CMS version is vulnerable to authenticated remote code execution, giving a foothold as `www-data`.

From the compromised application, configuration and database information leads to credentials for the `aporter` system account. SSH access then provides the user flag.

The final privilege escalation comes from a vulnerable CUPS installation running version `2.4.16`, leading to root access.

### Attack Chain

```text
RDP
 │
 ├── contractor
 │
 ▼
Internal Wi-Fi
 │
 ├── Wireless Traffic Analysis
 │
 └── jenny Credentials
      │
      ▼
portal.international.htb
      │
      ├── Craft CMS 5.9.8
      │
      └── Authenticated RCE
              │
              ▼
           www-data
              │
              ├── .env
              └── Database Backup
                      │
                      ▼
                   aporter
                      │
                      ├── SSH
                      └── user.txt
                              │
                              ▼
                       CUPS 2.4.16
                              │
                              ▼
                       CVE-2026-34990
                              │
                              ▼
                             root
                              │
                              └── root.txt
````

---

# 1. Initial Enumeration

I started with a standard Nmap scan:

```bash
nmap -sC -sV <TARGET_IP>
```

The main exposed services were:

```text
22/tcp    SSH
3389/tcp  RDP
```

The provided credentials were:

```text
Username: contractor
Password: Contractor2026!
```

These credentials allowed me to access the machine through RDP.

---

# 2. RDP Access

After connecting through RDP, the workstation hostname was:

```text
airside-ws01
```

I started enumerating the local network configuration and noticed that the workstation had access to the internal wireless network:

```text
HTB International WiFi
```

There was also another wireless interface that initially wasn't associated.

This became important during the next stage.

---

# 3. Wireless Traffic Analysis

I investigated the available wireless interfaces and traffic.

The traffic contained credentials that could be recovered and reused against the internal portal.

The credentials discovered were:

```text
Username: jenny
Password: Fl1ghtDeck2026!
```

This provided the next foothold into the internal web environment.

---

# 4. Internal Portal

The internal portal was:

```text
http://portal.international.htb
```

The Miles application was available at:

```text
http://portal.international.htb/miles/login.php
```

After logging in as Jenny, I continued enumerating the main portal.

The `/admin` endpoint redirected to the Craft CMS administration interface:

```text
http://portal.international.htb/admin
```

The administration interface identified the application as Craft CMS.

The installed version was:

```text
Craft CMS 5.9.8
```

This version became the main target for the web exploitation stage.

---

# 5. Craft CMS RCE

The vulnerable Craft functionality was exposed through the element search endpoint:

```text
/admin/actions/element-search/search
```

Craft CMS 5.9.8 was vulnerable to authenticated remote code execution through the conditions/element-search functionality.

After constructing the appropriate request for the vulnerable version, I obtained command execution as:

```text
www-data
```

At this point, I moved from web application exploitation to local enumeration.

---

# 6. Application Enumeration

With access as `www-data`, I started looking through the application files and configuration.

One of the important files was:

```text
/var/www/portal/.env
```

The environment configuration contained application and database information.

I also discovered a database backup through the Craft CMS administration interface.

The database information eventually led to credentials for the system account:

```text
aporter
```

This was the transition from the web application foothold to a normal SSH account.

---

# 7. SSH Access

The internal server was reachable at:

```text
10.13.37.10
```

Using the recovered credentials, I connected over SSH:

```bash
ssh aporter@10.13.37.10
```

This provided a shell as:

```text
aporter
```

---

# 8. User Flag

The user flag was located in the `aporter` home directory:

```bash
cat ~/user.txt
```

The user flag was successfully captured.

At this point, the initial access chain was complete and I moved on to privilege escalation.

---

# 9. CUPS Enumeration

During local enumeration, I discovered that CUPS was running as a system service.

I checked its status with:

```bash
systemctl status cups --no-pager
```

The service was active:

```text
cups.service - CUPS Scheduler
```

The CUPS daemon was running as:

```text
/usr/sbin/cupsd -f
```

The service was also listening locally on port `631`.

The relevant listener was:

```text
127.0.0.1:631
```

---

# 10. Identifying the CUPS Version

The normal package queries did not immediately reveal the version.

I eventually found the CUPS configuration utility:

```bash
cups-config --version
```

The result was:

```text
2.4.16
```

This was significant because the machine was running a version affected by:

```text
CVE-2026-34990
```

---

# 11. CUPS Privilege Escalation

The target did not have external Internet access, so I obtained the PoC on my Kali machine and transferred it to the target.

The target already had the tools required to execute the exploit, including:

```text
python3
ipptool
socat
nc
```

The exploit was transferred to:

```text
/tmp/exploit.py
```

When I initially executed the PoC, Python returned:

```text
NameError: name 'os' is not defined
```

The script referenced `os.environ` without importing the `os` module.

I fixed the missing import:

```bash
sed -i '2i import os' /tmp/exploit.py
```

Then executed the exploit:

```bash
python3 /tmp/exploit.py
```

The CUPS privilege escalation succeeded.

---

# 12. Root

After the privilege escalation, I verified the current user:

```bash
id
```

The shell was running with root privileges.

The final flag was retrieved with:

```bash
cat /root/root.txt
```

Root access was successfully obtained.

---

# Complete Attack Chain

```text
RDP
 │
 │ contractor / Contractor2026!
 ▼
airside-ws01
 │
 │ Internal Wi-Fi
 ▼
Wireless Traffic Analysis
 │
 │ jenny / Fl1ghtDeck2026!
 ▼
portal.international.htb
 │
 │ Craft CMS 5.9.8
 ▼
Authenticated RCE
 │
 ▼
www-data
 │
 │ .env / Database Backup
 ▼
aporter Credentials
 │
 ▼
SSH
 │
 ▼
aporter
 │
 ├── user.txt
 │
 ▼
CUPS 2.4.16
 │
 │ CVE-2026-34990
 ▼
root
 │
 └── root.txt
```

---

# Credentials Discovered

| Service         | Username     | Password                                  |
| --------------- | ------------ | ----------------------------------------- |
| RDP             | `contractor` | `Contractor2026!`                         |
| Internal Portal | `jenny`      | `Fl1ghtDeck2026!`                         |
| SSH             | `aporter`    | Recovered from application/database stage |

> These credentials belong to the Hack The Box lab environment.

---

# Vulnerabilities

| Component | Version | Impact                                      |
| --------- | ------: | ------------------------------------------- |
| Craft CMS |   5.9.8 | Authenticated Remote Code Execution         |
| CUPS      |  2.4.16 | Local Privilege Escalation — CVE-2026-34990 |

---

# Key Takeaways

### Follow the attack surface

The initial RDP access was only the starting point. The workstation provided access to an internal environment that exposed additional services and credentials.

### Investigate unusual network interfaces

The wireless interfaces initially looked like a minor detail, but investigating the traffic led to credentials for the internal portal.

### Always identify exact versions

Finding Craft CMS and CUPS was only the beginning. Identifying the exact versions made it possible to match the services against known vulnerabilities.

### Check application backups

The database backup contained information that wasn't directly exposed through the normal application interface and eventually led to SSH access.

### Don't ignore local services

CUPS was only exposed locally, but it was running as a privileged service. Local-only services can still provide an effective privilege-escalation path.

---

# Results

| Stage                              | Status    |
| ---------------------------------- | --------- |
| Initial RDP Access                 | Completed |
| Internal Network Enumeration       | Completed |
| Wireless Credential Discovery      | Completed |
| Craft CMS Enumeration              | Completed |
| Craft CMS RCE                      | Completed |
| `www-data` Access                  | Completed |
| Application / Database Enumeration | Completed |
| SSH Access                         | Completed |
| User Flag                          | Captured  |
| CUPS Enumeration                   | Completed |
| CVE-2026-34990                     | Exploited |
| Root Access                        | Obtained  |
| Root Flag                          | Captured  |

---

# Proof of Completion

The Hack The Box achievement/proof of completion is included below.

<p align="center">
  <img src="images/layover.png" width="850">
</p>

You can also view the achievement here:

[Hack The Box — Layover Achievement](https://labs.hackthebox.com/achievement/machine/1830127/984)

---

# Author

## Badathala Jaisurya

Cybersecurity | DevOps | AI Security

**GitHub:**
[https://github.com/jaisurya93945](https://github.com/jaisurya93945)

**LinkedIn:**
[https://www.linkedin.com/in/badathala-jaisurya/](https://www.linkedin.com/in/badathala-jaisurya/)

**Hack The Box Achievement:**
[https://labs.hackthebox.com/achievement/machine/1830127/984](https://labs.hackthebox.com/achievement/machine/1830127/984)

---

## Disclaimer

This write-up documents exploitation performed against an authorized Hack The Box laboratory environment.

The techniques described here are intended for authorized security testing, education, and CTF practice.

```

```

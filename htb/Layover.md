````markdown
# Hack The Box — Layover

> A complete walkthrough of the Layover machine, covering initial access, internal network enumeration, Craft CMS exploitation, credential discovery, SSH access, and CUPS privilege escalation.

**Author:** Badathala Jaisurya  
**Platform:** Hack The Box  
**Machine:** Layover  
**OS:** Linux

[GitHub](https://github.com/jaisurya93945) · [LinkedIn](https://www.linkedin.com/in/badathala-jaisurya/) · [HTB Achievement](https://labs.hackthebox.com/achievement/machine/1830127/984)

---

## Overview

Layover is a multi-stage Linux machine where the initial RDP access leads into an internal network. From there, wireless traffic analysis reveals credentials for an internal user, which provides access to a Craft CMS installation.

The Craft CMS instance is running version `5.9.8` and can be exploited for authenticated remote code execution. After gaining access as `www-data`, application and database information leads to SSH credentials for `aporter`.

The final privilege escalation comes from an outdated CUPS installation running version `2.4.16`, which is vulnerable to CVE-2026-34990.

### Attack Chain

```text
RDP
 │
 ├── contractor
 │
 ▼
Internal Wi-Fi
 │
 ├── Wireless traffic analysis
 │
 └── jenny credentials
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
      ├── .env / database enumeration
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
           └── CVE-2026-34990
                    │
                    ▼
                   root
                    │
                    └── root.txt
````

---

# 1. Enumeration

I started with an Nmap scan:

```bash
nmap -sC -sV <TARGET_IP>
```

The main exposed services were:

```text
22/tcp    SSH
3389/tcp  RDP
```

The provided RDP credentials were:

```text
Username: contractor
Password: Contractor2026!
```

I used these credentials to access the machine through RDP.

---

# 2. RDP and Internal Network

After connecting through RDP, I enumerated the available network interfaces.

The workstation was connected to:

```text
HTB International WiFi
```

There was also another wireless interface that initially wasn't associated.

Investigating the wireless traffic eventually exposed another set of credentials:

```text
Username: jenny
Password: Fl1ghtDeck2026!
```

These credentials provided access to the internal web application.

---

# 3. Internal Portal

The internal portal was:

```text
http://portal.international.htb
```

The Miles application was available at:

```text
http://portal.international.htb/miles/login.php
```

After authenticating as Jenny, I continued enumerating the portal and discovered a Craft CMS administration interface:

```text
http://portal.international.htb/admin
```

The Craft CMS version shown in the administration interface was:

```text
5.9.8
```

This became the main web exploitation path.

---

# 4. Craft CMS RCE

The Craft CMS installation was vulnerable to authenticated remote code execution through its element search/condition functionality.

The relevant endpoint was:

```text
/admin/actions/element-search/search
```

After constructing the appropriate request for the vulnerable version, I obtained command execution as:

```text
www-data
```

At this point, I started enumerating the application files and configuration.

---

# 5. Application Enumeration

One of the important application files was:

```text
/var/www/portal/.env
```

The environment configuration contained useful application and database information.

I also discovered a database backup through the Craft administration interface.

The database information ultimately provided credentials that could be reused for a system account.

The account was:

```text
aporter
```

---

# 6. SSH Access

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

The user flag was then retrieved:

```bash
cat ~/user.txt
```

---

# 7. CUPS Enumeration

With access as `aporter`, I started enumerating local services.

CUPS was running:

```bash
systemctl status cups --no-pager
```

The service was running `cupsd` and was bound locally to port `631`.

```text
127.0.0.1:631
[::1]:631
```

The CUPS version was determined using:

```bash
cups-config --version
```

The result:

```text
2.4.16
```

This version is vulnerable to:

```text
CVE-2026-34990
```

---

# 8. CVE-2026-34990

The target did not have external Internet access, so I obtained the PoC from my Kali machine and transferred it to the target.

The target already had the tools required to execute the PoC:

```text
python3
ipptool
socat
nc
```

After transferring the exploit to `/tmp`, I initially encountered a Python error because the script referenced `os.environ` without importing the `os` module.

I fixed the missing import:

```bash
sed -i '2i import os' /tmp/exploit.py
```

Then executed the exploit:

```bash
python3 /tmp/exploit.py
```

The CUPS privilege escalation succeeded and resulted in root access.

---

# 9. Root

After obtaining root:

```bash
id
```

The final flag was retrieved with:

```bash
cat /root/root.txt
```

Root access achieved.

---

# Credentials Discovered

| Service         | Username     | Password                                  |
| --------------- | ------------ | ----------------------------------------- |
| RDP             | `contractor` | `Contractor2026!`                         |
| Internal Portal | `jenny`      | `Fl1ghtDeck2026!`                         |
| SSH             | `aporter`    | Recovered from application/database stage |

---

# Vulnerabilities

| Component |  Version | Issue             |
| --------- | -------: | ----------------- |
| Craft CMS |  `5.9.8` | Authenticated RCE |
| CUPS      | `2.4.16` | CVE-2026-34990    |

---

# Key Takeaways

* Initial access does not always lead directly to the target system; the RDP workstation provided access to an internal network.
* Wireless interfaces and traffic can expose credentials that aren't visible through normal web enumeration.
* Identifying exact software versions is important when moving from enumeration to exploitation.
* Application backups can contain credentials that can be reused against system services.
* Local services should always be reviewed during privilege-escalation enumeration.
* CUPS was only exposed locally, but it still provided the final privilege-escalation path.

---

# Flags

```text
User Flag:  ████████████████████████████████████
Root Flag:  ████████████████████████████████████
```

Both flags successfully captured.

---

## Author

**Badathala Jaisurya**

Cybersecurity | DevOps | AI Security

* GitHub: [https://github.com/jaisurya93945](https://github.com/jaisurya93945)
* LinkedIn: [https://www.linkedin.com/in/badathala-jaisurya/](https://www.linkedin.com/in/badathala-jaisurya/)
* HTB Achievement: [https://labs.hackthebox.com/achievement/machine/1830127/984](https://labs.hackthebox.com/achievement/machine/1830127/984)

---

## Disclaimer

This write-up is for educational purposes and documents exploitation performed against an authorized Hack The Box lab environment.

```
```

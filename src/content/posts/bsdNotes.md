---
title: Setting up an OpenBSD Web Server
published: 2023-09-20
description: ''
image: ''
tags: [openBSD]
category: ''
project: 'OpenBSD Web Server'
draft: false 
---

The following details the basic setup of a webserver on openBSD using httpd. First, we need to do some basic setup and securing of our server.

# Securing our Server

## Generating SSH keys

We will create some `ssh` keys to log into our webserver with instead of passwords.

**On your local machine, create a keypair with:**  
  
`ssh-keygen -t ed25519 -C "enter a string e.g. name, email"`

You will be prompted to save the key to a file of your choosing.

**Now we need to copy the public key to our remote server with:**  
  
`ssh-copy-id -i ~/.ssh/filename.pub user@server`

## Configure A New User

If you are using a VPS or similiar you may be interacting with the server as the root user. For security we are going to disable the root login and create a new user with the ability to act as root when needed with `doas`.

If you are using a VPS, it is a good idea to change the default password set by your provider.  
  
`passwd root`

**Create a new user**  
*use a username you do not use in public*  
  
`adduser -batch hemingway wheel`

Here hemingway is the username and wheel is our user group. The `wheel` group allows our user to act as root when needed. ([man adduser](https://man.openbsd.org/adduser))

The `-batch` flag is used for creating multiple users at once, but allows us to skip a lot of the intial set up of an account e.g. name, default shell. However it also leaves our user without a password, so go ahead and set one up right away:

`passwd hemingway`

SSH keys are local to users so we will need to copy our ssh keys we generated earlier to our new user.

**Copy your public ssh key again, this time to the new user**  
  
`ssh-copy-id -i ~/.ssh/filename.pub hemingway@server`

## Configure SSH on Remote
*if you have not copied your public key to the new user yet, you are going to have a bad time*

`vi /etc/ssh/sshd_config`  
`Permitrootlogin no`  *disables logging in as root through ssh*  
`PubkeyAuthentication yes` *make sure this line is uncommented*  
`PasswordAuthentication no` *force users to ssh with keys only*   
`/etc/rc.d/sshd restart`  *restart ssh to make changes*

## Setup `doas` (alt. to sudo)
doas allows us to run commands as root without switching to root everytime

`cp /etc/examples/doas.conf /etc`  
`vi /etc/doas.conf`  
  

**add `persist` in the lines below**  
  
`# Allow wheel by default`  
`permit persist keepenv :wheel`   

adding `persist` means we don't have to enter our password "everytime" we run `doas`

# Basic `httpd` setup

OpenBSD ships with a built in webserver called `httpd`, this is what we will be using for our website.

## Configuring httpd

Let's create the config file  
  
`doas vi /etc/httpd.conf`  

and add the following

```
server "default" {
  listen on 120.81.121.23 port 80
  root "/htdocs/test"
  directory index index.html
}

types {
  include "/usr/share/misc/mime.types"
}
```  

This will allow us to run a simple website on port 80 (unencrypted) with the root directory at /test. The `directory` line sets our website's index file to index.html in /test. The full path to /test is *`/var/www/htdocs/test/`*

Any time you change `httpd.conf` verify there are no errors with `doas httpd -n`  

We can start our server with:

`rcctl enable httpd`  
`rcctl start httpd`  

And stop it with:  

`rcctl stop httpd`  
`rcctl disable httpd`  

An unencrypted website isn't very useful, let's move on to creating one with TLS encryption.

# Configure TLS

To configure TLS for our website we are going to be using **Let's Encrypt**, a nonprofit which will provide us with free TLS certificates.

## Set up our Website to receive challenges

Before requesting a TLS certificate we must configure our website to respond to *"challenges"*. These challenges verify that we control the domain name in our certificate. You can read about how these challenges work [here.](https://letsencrypt.org/docs/challenge-types/)

To do this, edit `/etc/httpd.conf`

```
server "www.mopuppet.tech" {
  listen on $ext_ip port 80
  root "/htdocs/test"
  directory index index.html

  location "/.well-known/acme-challenge/*" {
  	root "/acme"
  	request strip 2
  	directory no auto index
  }
}
```

## Configure `acme-client` to create certificate

We now need to configure our `/etc/acme-client.conf`

`cd /etc/examples`  
`cp acme-client.conf /../.`  
`vi /etc/acme-client.conf`

```
domain www.mopuppet.tech {
  alternative names {
  mopuppet.tech
  }
  domain key "/etc/ssl/mopuppet-acme/mopuppet.key"
  domain certificate "/etc/ssl/mopuppet-acme/mopuppet.crt"
  domain full chain certificate "/etc/ssl/mopuppet-acme/mopuppet.fullchain.pem"
  sign with "letsencrypt"
}
```

Before creating our certificate we need to create the directory we have referenced above. You could store all your files in `/etc/ssl` but this would likely get messy.

`cd /etc/`  
`mkdir ssl/mopuppet-acme`

**Create the certificate:**  
*make sure httpd is running and your website is accessible*

`acme-client www.mopuppet.tech`

## Configure TLS in `httpd.conf`

```
server "www.mopuppet.tech" {
  alias "mopuppet.tech"
  listen on $ext_ip tls port 443
  tls certificate "/etc/ssl/mopuppet-acme/mopuppet.fullchain.pem"
  tls key "/etc/ssl/mopuppet-acme/mopuppet.key" 
  root "/htdocs/mopuppet-web"
  directory index index.html
  
}
```

Unfortunately these certificates aren't permanent and expire after **90 days**. 

One solution is to add `acme-client www.mopuppet.tech` to our cron daemon to periodically check and renew the certificate if need be.
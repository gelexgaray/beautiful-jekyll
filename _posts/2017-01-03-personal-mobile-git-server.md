---
layout: post
title: "Personal Mobile Git Server"
date: 2017-01-10 00:22:13 +0100
comments: true
tags: ["misc", "hacking"]
---
Why should I need a git server on my mobile phone? Well... first of all, because it is fun. Then, if you think about it again, we spend a lot of time writing lines "not so important" to put into whatever your source code server is, but quite important to track.

In my particular case, I usually spend some of my office lunch time writing personal code that I continue at home... so, why not synchronize my work and home computers using something that always travels with me?
<!-- More -->

## Personal git server on any unrooted Android phone
It's quite easy to put a chroot GNU-Linux on any rooted Android phone, but what about your new unrooted phone? Well, I will explain to you how you can convert it on a git server.

### Termux
[Termux](https://termux.com) is an open source project that brings to your Android phone a bash console. It is NOT a complete GNU system. It is a busybox with some basic Unix-style commands and apt. Yes... APT. Of course, it's not connected to the Debian or Ubuntu repositories... but in the Termux repositories, you can find a lot of console programs that can be run on top of the Linux kernel of your Android phone... and yes... OpenSSH and git can run on Termux.

### Installing Termux, ssh and git on your mobile phone

First, install Termux from Play Store. Then, start a Termux console and type:
```
apt update
apt install openssh
apt install git
```

#### Back on your computer...

If your are running Windows:

- Install [git for windows](https://git-for-windows.github.io/)
- Now you can use git bash command prompt and use the same steps as in any Unix flavor install

Unix flavor client install:

```
ssh-keygen 
```

This command will generate a new key pair. SSH on Termux doesn't support user/password authentication... so a key must be generated. Use the default options.

```
cd .ssh
```

Here you will see a file called id\_rsa.pub. This is your RSA public key. Copy contents of this file to your mobile phone (i.e using a keep note or whatever you like)

#### Again on the mobile...

```
cd .ssh
vi authorized_keys
```

This file must contain the authorized ssh clients' public keys.

- Type 'i' to enter insert mode on vi
- Copy content from id\_rsa.pub: all must stay in a single line
- Volume Up+E: This simulates the Escape key on the Termux Console. 

	> If you are a CyanogenMod user, volume keys must be mapped to main volume stream, and cursor control with volume keys must be disabled

- type ':wq' to write the file to disk

Now you can start the ssh server:

```
sshd
```

#### Again on windows git bash console

```
ssh mobile_ip -p 8022
```

Type password used for ssh-keygen and the bash prompt from your mobile phone should appear.

If you cannot connect: 

Again on the mobile:

```
pkill sshd
sshd -d
```

Try again connecting from windows and see the error log on the mobile phone

### Set up a git bare repository on your mobile phone

The home folder is always in the Termux installation path, and all the contents are destroyed if you uninstall the application. You should always create your git repositories in your phone's shared memory. In the example, mine is at /storage/emulated/0/.gitrepos

On the mobile:

```
cd /storage/emulated/0/.gitrepos/
mkdir testrepo
cd testrepo
git init --bare
```

Probably you will get a warning like this: "warning: Cannot protect .git/config on this file system - do not store sensitive information here.". This is due to the lack of permission management in sd card filesystem. It says anyone with physical access to your mobile phone could alter .config file and change, for example, the upstream repository or any other sensitive configuration. Ok, this is not to be used for a production server... ok?

### Clone repository

Now on the client, you can clone this repository. Whenever you use a non-standard port for ssh, you have to use the following syntax to refer your repository address:

ssh://mobile\_ip:8022/storage/emulated/0/.gitrepos/testrepo

> When using non-standard ports for ssh, you must provide the absolute path to your repository

```
git clone ssh://mobile_ip:8022/storage/emulated/0/.gitrepos/testrepo
```

OK, now you are ready to begin pushing to your mobile phone.

### Bonus track: Using a USB Connection to your GIT server

Sometimes, you can't connect your mobile and your computer to the same network. In this cases, you can always enable adb debugging, and use adb port forwarding to access your ssh server.

```
adb forward tcp:8022 tcp:8022
``` 

Then, you could do things like:

```
ssh localhost -p 8022
```
or

```
git clone ssh://mobile_ip:8022/storage/emulated/0/.gitrepos/testrepo
```

### Further references

- https://termux.com/ssh.html
- https://oliverse.ch/tech/2015/11/06/run-an-ssh-server-on-your-android-with-termux.html
- https://oliverse.ch/technology/2016/09/20/access-termux-via-usb.html1

# TItle: Gaming Server

**Difficulty:** Easy

**Category:** Red

## 1. Recon

I launched an Nmap scan against the target to find out which ports and services are running

```bash
nmap -sV <target-ip>
```

![alt text](pics/nmap.png)

Since apache is running, I opened the targets website and was greeted with a website of an RPG game

![alt text](pics/WEB1.png)

I came across an upload element while scrolling on the website. First think I thought I might be able to take advantage of an upload vulnerability, but when clicking it I was greeted with a directory listing.

![alt text](pics/scroll.png)

![alt text](pics/directory.png)

Looking through the directory, theres a file called dict.ls so I downloaded it

```bash
wget http://<target-ip>/uploads/dict.lst
```

The file contains a bunch of random words that seemed to be a wordlist, suggesting we might be needing it to bruteforce something so let us keep this file in mind for now.

![alt text](pics/wordlist.png)

Went back on exploring the website and find something interesting in the website's source code mentioning a developer named "john", suggesting that a user named john could be used when logging in through ssh later.

![alt text](pics/john.png)

## 2. Directory Enumeration

I ran gobuster against the target website to check for hidden endpoints in the website

```bash
gobuster dir -w /usr/share/dirb/wordlists/common.txt -u http://target-ip/
```

After fuzzing it gave us another directory called secret

![alt text](pics/secret.png)

We are greeted with another directory but this time it has an RSA key in it so of course I downloaded it to my machine.

```bash
wget http://target-ip/secret/secretKey
```

![alt text](pics/rsa.png)

## 3. Exploitation

Since we have the key and a username, we can now try loggin in via ssh

![alt text](pics/attempt.png)

We cannot log in yet since the RSA key is protected with a passphrase. Let us try bruteforcing it with john the ripper

First let us convert the rsa keys content to a hash john the ripper can understand

```bash
ssh2john secretKey > hashedkey
```

After that we can now bruteforce the converted hash still using john the ripper

```bash
john --wordlist=/usr/share/wordlists/password/rockyou.txt hashedkey
```

We are then given with the passphrase "letmein" and we've successfully logged in.

![alt text](pics/logged.png)

## 4. Retrieving the flag and Priviledge Escalation

We can now retrieve the user flag in the directory we landed

![alt text](pics/user.png)


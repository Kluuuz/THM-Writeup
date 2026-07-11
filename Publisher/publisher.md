# Title: Publisher
**Difficulty:** Easy

**Category:** Red

## 1. Recon

I start by launching an nmap scan on the target

```bash
nmap -sV -sC target-ip
```

![alt text](pics/1nmap.png)

It reveals two ports are open, port 22 which is running an ssh and 80 running an apache service

```Results
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Since we know an apache service is running. Let us explore the web app. We are then greeted with a dead end website. Clicking sections like About, Contact Us and etc does not lead to any web pages, although clicking other sections like blogs redirects us to differet websites which are irrelevant to the challenge

![alt text](pics/webapp.png)

let us run gobuster to look for hidden web pages.

```bash
gobuster dir -w wordlist.txt -u target-ip
```

After testing multiple wordlists while enumerating it found a suspicious endpoint called "spip"

![alt text](pics/spip.png)
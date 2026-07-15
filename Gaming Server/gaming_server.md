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
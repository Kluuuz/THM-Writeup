# Title: Oracle 9

**Category:** Red

**Difficulty:** Easy

Before we start lets check if the target machine is up by pinging the targets IP Address

```bash
ping 10.49.148.75
```

![alt text](photos/ping.png)

Since the target machine is up, let us start the first phase, Reconnaissance

## 1. Reconnaissance

Let us start by running Nmap on the target machine to find out which services are running

````bash
nmap 10.49.148.75
````
After running nmap, we discovered that 3 ports and 3 services are running 

![alt text](photos/1nmap.png)

22/tcp   open  ssh
80/tcp   open  http
5000/tcp open  upnp

Port 5000 seems interesting since I am not familliar with the service that is running in that port. I did some research and found out that upnp is a "networking protocol that allows devices like gaming consoles, smart TVs, and printers to automatically discover each other and configure your router to open ports." lets keep that in mind.

![alt text](photos/upnp.png)

To find out more about the services that are currently running in the target machine I ran another nmap scan but this time using the -sV flag

![alt text](photos/nmapsv.png)

It looks like the web is hosting an AI assistant

## 2. Explore the web

We are greeted with a webpage where we can interact with an AI assisstant. But it seems like it only responds if you are authorized.

![alt text](photos/web.png)

When I inspect the webpage to check if there are any cookies present. It looks like it does not have any cookies.


![alt text](photos/nocookie.png)

## 3. Bypassing the AI's Security

I manually created a cookie and initially assigned it a value of 1, but the application still denied access.

Next, I changed the cookie value to admin, suspecting that the application might be checking for a specific role rather than performing proper authentication.

![alt text](photos/admin.png)

After refreshing the page and interacting with the AI assistant again, I asked it to reveal the sealed transmission.

![alt text](photos/flag.png)

The AI accepted the modified cookie and disclosed the hidden transmission, revealing the flag.
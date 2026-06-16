+++
title = "Detecting System Compromise via SMB with ELK"
date = 2026-06-16T08:48:36Z
tags = ["Metasploit", "Pentesting", "Windows", "Linux", "SIEM"]
summary = "Using Metasploit and Mimikatz to simulate brute-force attacks and credential dumping over SMB, then detecting it in Kibana."
+++
## **OVERVIEW**
This project was done within my home SOC lab, running **SMBv1** on my **Windows 10** target machine, **Kali Linux** on my attacker, and **ELK SIEM** (Elastic, Logstash, Kibana). The target system was put under a **brute-force** attack to gain valid login credentials, then exploited through **metasploit** and **kiwi** (Mimikatz for MSF) to extract credentials. The goal of the project was to showcase how those attacks show up in **Kibana**, and how rules can be implemented to make them easier to detect.

## **SETUP**
To get started, I needed to configure **three** things within the target machine of my home SOC lab. The goal was to make this **Windows 10** machine as **vulnerable** as possible and plant information to extract, which required me to do the following:  
&nbsp;  
**First**, I'd need to configure it to run the **SMBv1** service, which is known for being *super vulnerable* to RCE. 
![configuring smbv1](/images/smb_lab/setup1.png)
**Then**, I would have to *disable* **Windows Defender** to prevent it from blocking the connection to my attacker machine and getting in the way of my attempts at dumping credentials.  
![disabling realtime protection](/images/smb_lab/disable.png)
(Checking to make sure Realtime Protection is disabled through PowerShell)
![making sure its off](/images/smb_lab/check.png)
**Lastly**, I needed to create a flag to represent valuable information, which would get extracted by my attacker:
![creating a flag](/images/smb_lab/flag.png)
&nbsp;  

Time to move on to the fun part.

## **EXPLOITATION**
Now that the vulnerable machine had been setup, it was time to hack into it. From my *attacker* machine running **Kali Linux**, I did a quick **nmap scan** on the target to make sure it had SMBv1 running:
![running nmap on the target](/images/smb_lab/nmap.png)
(Here's the full output from the scan, revealing more details about the target)
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-16 12:51 PDT
Nmap scan report for 192.168.56.101
Host is up (0.0031s latency).
Not shown: 996 filtered tcp ports (no-response)
PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Windows 10 Pro 19045 microsoft-ds (workgroup: WORKGROUP)
5357/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Service Unavailable
Service Info: Host: DESKTOP-D0JR2TS; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: DESKTOP-D0JR2TS, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:23:b5:f0 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
| smb2-time: 
|   date: 2026-06-16T19:50:59
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_clock-skew: mean: 2h19m08s, deviation: 4h02m29s, median: -52s
| smb-os-discovery: 
|   OS: Windows 10 Pro 19045 (Windows 10 Pro 6.3)
|   OS CPE: cpe:/o:microsoft:windows_10::-
|   Computer name: DESKTOP-D0JR2TS
|   NetBIOS computer name: DESKTOP-D0JR2TS\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-06-16T12:50:59-07:00

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.44 seconds
```
The scan confirmed that SMB was indeed up and running, as indicated by ports **139** and **445**. So far, so good! I treated this like a real target and used a **password list** to brute force the target with **metasploit**. For the sake of the lab, I kept the list short, only 50 passwords, with one line containing the target's real login.  
&nbsp;  
I went with the **auxiliary/scanner/smb/smb_login** module from metasploit. I went ahead and configured the module and took a shot at the target machine using the custom password list I created:
![brute forcing login](/images/smb_lab/bruteforce.png)
![brute forcing part 2](/images/smb_lab/bruteforce2.png)
Alright smooth, metasploit was able to **successfully** brute force the login. Now it was time to move on to getting a foothold on the machine. To do that, I went with the **windows/smb/psexec** module. This exploit was crafted for vulnerable SMB services like the one I had running on the target, and now with valid credentials, I could get a shell popped.  
&nbsp;  
All I had to do was configure the module and run it. I set the **SMBPass** field to the cracked password, set the **PAYLOAD** to run a **bind_tcp** meterpreter shell (since outbound connections were blocked), and ran the exploit:
![getting a bind shell](/images/smb_lab/shell.png)
**We're in**. Time to privesc and dump creds:  
&nbsp;  
(Checking for SYSTEM level privileges and initializing kiwi, meterpreter's build in mimikatz tool)
![privesc and mimikatz](/images/smb_lab/privesc.png)
![dumping credentials](/images/smb_lab/creds.png)
(Switching directories to the home user's Desktop and grabbing the flag)
![extracting the flag](/images/smb_lab/found.png)
**Success!** The Windows 10 target had been *completely* compromised. But I didn't do all that without leaving a *nasty* paper trail. So let's dig into the logs.

## **DETECTION**
The two main things I was focused on detecting in this lab were **brute-force attempts** and **credential dumping**. My Kibana dashboard on my SIEM had very detailed logs thanks to the **sysmon + winlogbeat** services running on the target so finding these wasn't too difficult. I also included detection rules that could be implemented to make finding these attacks from the dashboard much easier.    
&nbsp;  
The **brute-force** attack I did on the target correlates with **"event.action logon-failed"**, which flooded my Kibana dashboard thanks to the 50 login attempts sent from my attacker:  
&nbsp;  
<video width="100%" controls>
  <source src="/videos/failed-logins.mp4" type="video/mp4">
</video>
&nbsp;  
**PROPOSED DETECTION RULE: event.code: 4625 | threshold > 10 in 60s | alert: possible brute force**  
This detection rule creates an alert if the "logon-failed" event action happens over 10 times within 60 seconds.  
&nbsp;  
As for the **credential dumping**, I was able to locate it in Kibana by using the **"event.action credential-manager-credentials-were-read"** filter. Although my target machine only had **one user** to have credentials extracted, the logs generated were still good evidence of my SIEM in action:
![cred logs](/images/smb_lab/credlog.png)
**PROPOSED DETECTION RULE: event.code: 5379 | any occurence | alert: credential manager accessed**  
This detection rule sends an alert if the credential manager is accessed, which is important because there are almost no cases where a user in a domain needs that open.  
&nbsp;  
The file extraction events were put under an **"event.action process creation"** log, however, those are generated at a very high frequency- little too high for the scope of this lab. In a production SOC environment, this would be addressed by tuning sysmon filters to specifically flag suspicious process spawns from meterpreter sessions. Now that all that's out the way, let's debrief! 

## **TAKEAWAYS**
This lab was a genuinely fun way to see how attacks look from the other side of the screen. Watching my Kibana dashboard get *flooded* with failed login attempts in real-time made the detection side of security click in a way that watching videos on it never really does.  
&nbsp;  
One thing that stood out was how detection gaps can be just as revealing as the detections themselves. My sysmon config didn't isolate file access events cleanly enough to catch the flag extraction... In a *real* SOC, that's definitely a blind spot worth fixing.  
&nbsp;  
In a future project, I want to tighten up the sysmon config using Swift on Security's template and get proper automated alerts running so the SIEM does most the hunting *for* me. Still a lot of ground to cover, but the attack chain is *mapped*. Now it's just about sharpening the detection.
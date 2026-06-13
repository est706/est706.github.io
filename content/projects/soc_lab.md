+++
title = "Building a Home SOC Lab"
date = 2026-06-10T10:58:30Z
tags = ["SOC", "Windows", "Linux", "SIEM"]
summary = "Using Elastic SIEM on Ubuntu to monitor a Windows machine."
+++
## **OVERVIEW**
This **SOC lab** was built using **VirtualBox**, running an Ubuntu machine with Elastic SIEM, a Windows 10 Pro machine acting as a target, and a Kali Linux machine for attacking. It was put together to practice **monitoring logs**, **simulating attacks**, and **detecting anomolies** within a simulated enterprise environment.

## **ARCHITECTURE**
To get started setting up the lab, I needed to assess its architecture. The lab needed **three machines** to function how I wanted it to. To get things moving, I went ahead and created the following machines in Virtualbox:
![architecture](/images/soc_lab/architecture.png)
**Kali:** Runs Kali Linux, used as an attacking machine (this one had already been created for pentesting assessments in the past)  
&nbsp;  
**Elastic Stack:** Runs on Ubuntu 22.04 LTS, with a Docker container configured with Elasticsearch, Logstash, and Kibana. Elasticsearch stores and indexes the logs, Logstash handles log ingestion and parsing, and Kibana serves as the GUI for searching and visualizing the data. Altogether, a lightweight and effective SIEM.  
&nbsp;  
**Windows 10:** Runs Windows 10 Pro, with Sysmon and Winlogbeat. The Sysmon service monitors system events and writes them to the Windows event log. Winlogbeat ships these logs to the Ubuntu machine, where Logstash parses them and hands them off to Elasticsearch.  
&nbsp;  
Note: During the setup of each machine, I made sure they were configured with **Host-Only** and **NAT** network adapters in VirtualBox. This would allow each of them to talk to each other *and* access the internet at the same time.  

## **STACK**
I chose this tech stack because it was pretty straightforward and cost me nothing to implement.  
&nbsp;  
**VirtualBox:** Since I already had my Kali Linux machine configured through VirtualBox, I saw no reason to switch to something like VMWare. VirtualBox is free and has great snapshot support so I figured it was a good option for an isolated lab environment.  
&nbsp;  
**Elastic Stack (ELK):** I chose to run Elastic instead of another SIEM like Splunk or Wazuh because of its free self-hosted tier and unlimited data ingestion. It's also widely used in enterprise environments.  
&nbsp;  
**Sysmon & Winlogbeat:** Default Windows log events are very surface level, so I wanted to implement Sysmon to generate more detailed logs (which show things like process creation, network connections, registry changes, file creation). I set up Sysmon using the *SwiftOnSecurity Sysmon Config*, a widely used configuration that sets up Sysmon to create high-quality event tracing. Winlogbeat is a log shipping tool that was created specifically for Windows logs and it also has native integration for Elastic Stack, so I saw it as the best option for sending logs from the Windows machine to the SIEM.  

## **SETUP**
To get everything setup, I started with my Ubuntu machine. This VM would be used for running the ELK (Elastic, Logstash, Kibana) SIEM and also for monitoring logs through Kibana (which is why I went with Ubuntu Desktop instead of Ubuntu Server). I deployed ELK using **Docker Compose** via the **deviantony/docker-elk** repository. A full guide on installation is included in the repo, which you can find [**here**](https://github.com/deviantony/docker-elk).  
&nbsp;  
Once the ELK SIEM was fully assembled and elasticsearch, kibana, and logstash were all configured, I made sure all services were running: 
![checking services for ELK SIEM](/images/soc_lab/docker.png)
Now it was time to move on to the next step, setting up the **Windows 10** machine!  
&nbsp;  
Once the Windows VM installation had completed, I went ahead and installed **Sysmon** with the **SwiftOnSecurity configuration**. This configuration allows Sysmon to create detailed logs right out of the box. Then I installed **Winlogbeat**, which would be used as the shipping agent for moving the Sysmon logs *over* to the **ELK SIEM**.  
&nbsp;  
I downloaded Sysmon from [**this Microsoft link**](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon), then moved the **Sysmon64** application to my VM's **System32** folder. Winlogbeat can be downloaded from [**here**](https://www.elastic.co/downloads/beats/winlogbeat), and all you have to do upon downloading is edit the **.yml** file that comes with it to send logs to your SIEM. The installation for both was pretty straightforward, and the setup was very quick.  
&nbsp;  
Once I got both of those tools configured on my Windows VM, I ran a quick check to make sure both services were running:  
![checking services for windows vm](/images/soc_lab/powershell.png)
I went back to my Ubuntu machine running the ELK SIEM, and navigated to **http://localhost:5601** in my browser to get to Kibana, the GUI running Elastic. The dashboard was empty at first, so I had to configure a **data view** to grab the logs from winlogbeat and throw them on the dashboard. Once that was setup, I was finally able to monitor the Windows VM!
![kibana dashboard](/images/soc_lab/kibana.png)
&nbsp;  
Alright smooth, everything is in place. Now, let's see it in-

## **ACTION**
With the lab fully setup, I was eager to test it out. I booted up my attacker machine running Kali, and tried to ping the Windows VM. Nothing went through at first, and then I realized I needed to **disable the Windows Defender firewall**:  
![turning off the windows defender firewall](/images/soc_lab/firewall.png)
After disabling that, I was able to successfully ping the target machine from my attacker. No packets dropped here:
![pinging from attacker](/images/soc_lab/ping.png)
Out of curiosity, I wanted to check Kibana to see if *turning off the firewall* would be logged. Sure enough, I was able to find it! This event showed up:  
![kibana firewall disable](/images/soc_lab/event.png)
Sysmon logged the disabling of the firewall as a registry change event, even auto-tagged it with **RuleName:T1089**. Elastic documentation maps rules with MITRE ATT&CK technique identifiers, with this *specific* rule representing **T1089 - Disabling Security Tools** (which actually got sub-grouped into T1685, however Kibana still shows the old technique identifier). This really goes to show the depth that Sysmon is able to detect, and also served as a pretty cool first look into Kibana's monitoring!  
&nbsp;  
Now, I wanted to see if my attacker machine was able to generate traffic to the target... But in order to log **nmap scans** hitting my Windows VM, I had to look *beyond* Sysmon. Out the box, the SwiftOnSecurity config logs outbound connections initiated by specific processes- useful for catching malware "calling home," but doesn't capture *inbound* connections like a port scan since there's no local process initiating them.  
&nbsp;  
To capture this kind of traffic, I enabled Windows' built-in Security audit policy for connection logging, specifically **Event ID 5156** (Windows Filtering Platform: Connection Allowed), which logs every connection at the OS level *regardless* of the process involved:
```powershell
PS C:\Windows\system32> auditpol /set /subcategory:"Filtering Platform Connection" /success:enable /failure:enable
The command was successfully executed.
```
Once that was in place, I ran an **nmap** scan on my Kali machine using T5 to make it quick:  
![nmap scan](/images/soc_lab/nmap.png)
Then, I checked Kibana to see if the scan got logged:  
![kibana source address](/images/soc_lab/sourceaddress.png)
Sure enough, the **Filtering Platform Connection** event.action was logged for the nmap scan coming from my attacking machine's IP within the same time as running the scan, confirming the connection attempts hitting the Windows VM were being captured at the OS level.  
&nbsp;  
**One thing worth noting:** Event ID 5156 logs *every* connection, not just suspicious ones, which means it generates a massive volume of entries very quickly compared to Sysmon's curated event set. After confirming the scan was logged, I disabled the audit policy to keep the lab from getting flooded with unnecessary noise:
```powershell
PS C:\Windows\system32> auditpol /set /subcategory:"Filtering Platform Connection" /success:disable /failure:disable
The command was successfully executed.
```
Seeing my dashboard get stacked with the same events was a good reminder that more logging isn't always better- knowing *when* to enable verbose logging, and turning it off once you've got what you need, is part of keeping a SIEM manageable. Something I'll definitely keep in mind as I do more projects with this SOC lab.

## **TAKEAWAYS**
Time to debrief. This was the first home lab I've ever created and it was both a great learning experience *and* a lot of fun to put together. I got to see how the ELK SIEM works in real-time and even tracked my own **Windows Defender Firewall changes** and **nmap scan** in Kibana! Though my lab is much smaller in scale compared to an enterprise environment, it serves its purpose as a good simulation of how SIEMs work in the real world.  
&nbsp;  
So what's next?  
&nbsp;  
I think simulating an attack or setting up a vulnerable service to exploit could be a good next step- I still got a lot to learn about how ELK picks up on threat actors. In any case, the foundation is laid, and now all I've got is **opportunity**.
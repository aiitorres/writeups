# Ignite

## 1. Introduction

### 1.1 Background

Ignite is a TryHackMe practice machine focused on learning **web exploitation and privilege escalation in Linux systems**. The objective is to compromise a vulnerable server, obtain initial access, and finally escalate privileges to **root**.

The intrusion is based on a remote command execution vulnerability in FUEL CMS. After gaining access to the system as the web server user, enumeration is performed to find a way to escalate privileges through a vulnerability in Polkit.

### 1.2 Objectives

The main objectives of the machine are:

- Identify the services exposed on the machine.
- Find vulnerabilities in the web application.
- Obtain initial access to the system.
- Perform enumeration of the compromised system.
- Escalate privileges to gain access as **root**.

---

## 2 Enumeration

### 2.1 Nmap

The first step was to perform a port scan with Nmap to identify available services.

```bash
nmap -sC -sV <IP>
```

The scan revealed that the only open port was **port 80 (HTTP)**, indicating that the entry point is in the web application.

### 2.2 Web

When accessing the website, it is observed that the server uses **FUEL CMS**. Exploring the application reveals the administrative panel at the path:

```
/fuel
```

Researching the CMS version shows that it is **FUEL CMS 1.4.1**, vulnerable to:

```
CVE-2018-16763
```

This vulnerability allows command execution on the server through a manipulable parameter in an HTTP request.

---

## Exploitation

To exploit the vulnerability, a public exploit was used that sends commands to the vulnerable CMS endpoint.

The exploit allows executing commands directly on the operating system. When testing the command **whoami**, it confirms that the obtained access corresponds to the web server user:

```bash
www-data
```

This confirms that **remote command execution on the server** has been achieved.

---

## Post-exploitation

After obtaining initial access, system enumeration was performed to find possible privilege escalation methods.

For this, the tool:

```bash
LinPEAS
```

was used, which analyzes the system for insecure configurations or known vulnerabilities.

During enumeration, it was detected that the system uses a vulnerable version of **Polkit**, affected by:

```bash
CVE-2021-4034
```

This vulnerability allows privilege escalation via the **pkexec** binary.

After executing the corresponding exploit, full system access is obtained. Verifying with **whoami** confirms that the current user is:

```bash
root
```

Finally, access to the **/root** directory is obtained, where the final flag is located, completing the machine.

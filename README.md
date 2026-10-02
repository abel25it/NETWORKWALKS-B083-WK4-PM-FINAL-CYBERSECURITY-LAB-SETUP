# Penetration Testing Report

**W4-PT-FINAL | CYBERSECURITY | NETWORKWALKS**

| Cybersecurity Professional Name | Abel Alex |
| :---- | :---- |
| Program/Batch | B083-Networkwalks |
| Date | October 2026 |
| Modules Completed | W4-M1 Initial Access — SQL Authentication Bypass W4-M2 Data Extraction — PDF Password Cracking W4-M3 Attack (Cracking) — Critical Data Exposure W4-M4 Penetration Testing Report |

---

## 1. Liability Disclaimer

I have performed these activities only on the systems & devices where I had secured written permission or the devices/systems that I own myself. All these materials are for education and research purpose only. Do not use anything from here to break the law. The instructor, the authors and Networkwalks are not responsible for what you do with this knowledge. Every action you take is your own responsibility. Misuse can lead to criminal charges, heavy fines, loss of your job and a permanent record. In most countries unauthorised access is a crime even when nothing is damaged.

---

## 2. Introduction

This report documents the practical activities completed during Week 4 of the Networkwalks Cybersecurity training program, a full black-box penetration test conducted against a controlled, educational target: **Mediroza General Hospital** (https://medirozahospital.com). The engagement was authorised in writing by Networkwalks and scoped to the target domain only.

The objective was to simulate a real-world attacker with no prior knowledge of the system, discovering attack surfaces through reconnaissance, exploiting identified vulnerabilities to demonstrate real impact, and documenting all findings in a professional report. The four milestones covered initial access via web application exploitation, data extraction from retrieved files, critical server-side data exposure, and a complete penetration testing report.

---

## 3. Tools Used

| Tool | Purpose |
| :---- | :---- |
| feroxbuster | Fast directory and file enumeration tool used to discover hidden paths and endpoints on the web server. |
| Firefox Browser | Real browser used for manual testing of login forms and authentication bypass payloads, bypassing WAF detection. |
| Networkwalks Password Cracker| Password cracking tool used with the rockyou.txt and JTR_Deafault_Passwords wordlist to crack extracted PDF password hashes. |
| pdf2john | Hash extraction utility that pulls a crackable hash from an encrypted PDF file for use with JTR. |
| Kali Linux Terminal | Operating environment for running all command-line tools. |

---

## 4. Activities Performed

### 4.1 Milestone 1 — Initial Access

**Objective:** Attack the website and retrieve 3 confidential patient PDF lab reports.

#### Step 1 — Directory Enumeration with feroxbuster

The engagement began with active reconnaissance using feroxbuster to map the web server's directory structure. The following command was run against the target:

```bash
feroxbuster -u https://medirozahospital.com
```

feroxbuster cycles through thousands of common path names from a wordlist and reports which URLs return a live response (HTTP 200). Most paths returned 403 (Forbidden) or 404 (Not Found). Two paths returned **HTTP 200 — accessible**:

| Path | Status | Significance |
| :---- | :---- | :---- |
| /patient/ | 200 OK | Patient portal — login form present |
| /old/ | 200 OK | Legacy directory — no authentication |

<img width="1353" height="713" alt="Screenshot 2026-10-02 183418" src="https://github.com/user-attachments/assets/1beb9554-0005-47bd-9b8f-6bb0b180d143" />


#### Step 2 — Staff Login Testing (Default Credentials + SQL Injection)

Before targeting the patient portal, the staff login at `/staff/login.php` was tested. This is documented as a separate finding.

All default credential attempts failed. The form returned *"Invalid username or password"* for every combination.

**SQL Injection testing** — the following payloads were then tried in the username field:

| Payload | Result |
| :---- | :---- |
| `admin' --` | Failed |
| `' OR 1=1--` | Failed |
| `1' OR '1'='1' --` | Failed |

All injection attempts failed on the staff login. The page uses a *"Staff ID"* field (numeric) which appears to use parameterised queries or integer casting, making it resistant to string-based SQL injection. This is documented as a positive security control — see M4 Findings.

#### Step 3 — Patient Login SQL Injection Authentication Bypass

The patient portal at `/patient/login.php` was then tested using the same manual approach in Firefox.

**Why SQL Injection works here:**

The backend PHP code likely constructs the login query by directly concatenating user input:

```php
// Vulnerable code — user input inserted directly into SQL string
$query = "SELECT * FROM users 
          WHERE username = '" . $username . "' 
          AND password = '" . $password . "'";
```

When the payload `admin' -- ` is entered as the username, the resulting query becomes:

```sql
-- What the database actually runs:
SELECT * FROM users WHERE username = 'admin' --  AND password = 'test'
```

The single quote (`'`) closes the username string early. The double dash (`-- `) tells MySQL to treat everything after it as a comment. The password check is discarded entirely. If any user named `admin` exists, access is granted with no password required.

**Payload used:**

| Field | Value |
| :---- | :---- |
| Username | `admin' -- ` (note: space after the two dashes) |
| Password | `123` (any value) |

**Result:** Successful login. Redirected to `/patient/portal.php` showing 3 confidential patient lab reports.

> **[Patient login form with payload entered]**
<img width="1916" height="861" alt="Screenshot 2026-10-02 184042" src="https://github.com/user-attachments/assets/20495bc5-3298-4de6-883e-fd4322f308c4" />


> **[Patient portal page showing the 3 PDF lab reports after login]**
<img width="1917" height="867" alt="Screenshot 2026-10-02 184130" src="https://github.com/user-attachments/assets/ca6e960f-4aef-4f73-85f3-7d8dd9d1daca" />


**Files Retrieved:**

| File | Patient |
| :---- | :---- |
| patient_report_1.pdf | S. Dlamini — Pathology Report |
| patient_report_2.pdf | P. Reddy — Pathology Report |
| patient_report_3.pdf | E. Thompson — Pathology Report |

All 3 PDF files were downloaded.

> **[Downloaded PDF files]**
<img width="437" height="118" alt="Screenshot 2026-10-02 184913" src="https://github.com/user-attachments/assets/7ad30a19-565c-4fea-bdf9-24534c1db1b8" />

---

### 4.2 Milestone 2 — Data Extraction

**Objective:** Crack the encryption on all 3 retrieved PDF files.

Each PDF was password-protected. The cracking workflow for each file was:

1. Extract a crackable hash using `pdf2john`
2. Run the hash against the rockyou.txt wordlist using John the Ripper

#### Hash Extraction

```bash
pdf2john patient_report_1.pdf > hash1.txt
pdf2john patient_report_2.pdf > hash2.txt
pdf2john patient_report_3.pdf > hash3.txt
```

Each extracted hash begins with `$pdf$4*4*128*...` indicating PDF revision 4 with 128-bit RC4/AES encryption.

> **[Terminal showing pdf2john for each file]**
<img width="783" height="361" alt="Screenshot 2026-10-02 185220" src="https://github.com/user-attachments/assets/0b14e81e-d649-4b49-bea7-afb9a3aae385" />


#### Password Cracking with Networkwalks Password Cracker
patient_report_1
<img width="1101" height="897" alt="image" src="https://github.com/user-attachments/assets/3f26d42c-ea4d-4a41-a61c-86cf170b02e5" />


patient_report_2
<img width="1120" height="896" alt="image" src="https://github.com/user-attachments/assets/1d995fa1-c5c9-4a99-a352-fba1815ae538" />


patient_report_3
<img width="1115" height="902" alt="Screenshot 2026-10-02 211153" src="https://github.com/user-attachments/assets/7dc598a0-2cf5-40ba-8648-b81866f1ab04" />


#### Results

| File | Password Found|
| :---- | :---- | :----|
| patient_report_1.pdf | `123456` |
| patient_report_2.pdf | `password` |
| patient_report_3.pdf | `!@#$%^&` |

All 3 files cracked successfully using the standard rockyou.txt and JTR_Deafault_Passwords wordlist with no advanced techniques required.

> **[PDF opened successfully with the cracked password]**
patient_report_1
<img width="562" height="768" alt="Screenshot 2026-10-02 211754" src="https://github.com/user-attachments/assets/16430d73-9703-47e1-80c5-ca2a4dd70d3a" />


patient_report_2
<img width="562" height="767" alt="Screenshot 2026-10-02 211845" src="https://github.com/user-attachments/assets/97fd67a4-5098-4daf-b59d-5baf395e3bf7" />


patient_report_3
<img width="565" height="763" alt="Screenshot 2026-10-02 212029" src="https://github.com/user-attachments/assets/89d885f4-98aa-4020-a68e-7fd7e92e0db1" />


**Analysis:** All three passwords are weak and dictionary-guessable. None required brute force, rule-based mangling, or any effort beyond a standard wordlist.

---

### 4.3 Milestone 3 — Attack (Cracking) / Critical Data Exposure

**Objective:** Find staff salaries and shareholder details of the hospital.

#### Step 1 — /old/ Directory Discovered via feroxbuster

During M1 reconnaissance, feroxbuster had already identified `/old/` as returning HTTP 200. Navigating to this path in Firefox revealed **directory listing was enabled** — the web server displayed its own file contents with no authentication required, as if it were an open shared folder.


This is a misconfiguration — when no `index.php` or `index.html` file exists in a directory, many web servers default to listing all contents publicly.

#### Step 2 — Database Backup Identified and Downloaded

The directory contained one file:

```
mediroza_db_backup_2019.sql
```

This file was downloaded directly from the browser with no login, no credentials, and no exploitation required — it was publicly accessible.

> **[mediroza_db_backup_2019.sql file visible in /old/ directory]**
<img width="1917" height="838" alt="image" src="https://github.com/user-attachments/assets/60290d64-5a20-4853-b7bb-67e20a34e2c0" />


#### Step 3 — Contents of the Database Backup

Opening the `.sql` file revealed a full database backup containing two critical tables:

**Staff Table — 30 employees including:**

| Column | Data Exposed |
| :---- | :---- |
| Full Name | Yes |
| Email Address | Yes |
| Phone Number | Yes |
| National ID Number | Yes |
| Job Title | Yes |
| Salary | Yes |

**Shareholders Table — full ownership structure including:**

| Column | Data Exposed |
| :---- | :---- |
| Shareholder Name | Yes |
| Shares Held | Yes |
| Share Percentage | Yes |
| Share Class | Yes |

**Contents of the Database Backup**

<img width="982" height="411" alt="image" src="https://github.com/user-attachments/assets/02c60c5a-6df5-4764-ae25-0175c61d3aeb" />
<img width="1261" height="562" alt="image" src="https://github.com/user-attachments/assets/eb8d4801-e854-485b-abb6-a6a1c1a13d3f" />
<img width="1072" height="467" alt="image" src="https://github.com/user-attachments/assets/a3653748-f109-4bbf-959f-1e83ca4d33cf" />



#### Step 4 — PDF Metadata Anomaly (j.malik)

During review of the 3 retrieved PDFs, file properties were checked on each document.

**How to check:** Open PDF → File → Properties → Description tab

| File | Author Field | Expected? |
| :---- | :---- | :---- |
| patient_report_1.pdf | Mediroza Diagnostics Lab | ✅ Normal |
| patient_report_2.pdf | Mediroza Diagnostics Lab | ✅ Normal |
| patient_report_3.pdf | j.malik | ❌ Anomaly |


**patient_report_1.pdf ✅ Normal**
<img width="806" height="361" alt="image" src="https://github.com/user-attachments/assets/7c191e2a-838a-4cba-abaa-d867de5d0bff" />


**patient_report_2.pdf ✅ Normal**
<img width="702" height="357" alt="image" src="https://github.com/user-attachments/assets/880fbe96-1017-4c69-ba73-f26ad9f376f2" />


**patient_report_3.pdf ❌ Anomaly**
<img width="851" height="371" alt="image" src="https://github.com/user-attachments/assets/711b12df-00a4-46a7-83ec-72303d78cf52" />


The author `j.malik` was cross-referenced against the leaked staff table and identified as **Jameel Malik, IT Systems Administrator** (staff table row 9, email: j.malik@medirozahospital.com).

A clinical patient pathology report should never be authored through IT administrator credentials. This indicates an access control gap between IT and clinical roles — the IT admin account has write access to patient record systems where it should not.

> **[j.malik as author on patient_report_3.pdf]**
<img width="851" height="371" alt="image" src="https://github.com/user-attachments/assets/9e33b1f0-beb6-4e2b-8cf2-539dcca4b6e2" />


> **[Jameel Malik confirming IT Systems Administrator role]**
<img width="1037" height="27" alt="image" src="https://github.com/user-attachments/assets/2d98f371-2a62-4f81-9657-652b1fee0b9a" />


---

## 5. Milestone 4 — Penetration Testing Report

---

### 01 — Executive Summary

A black-box penetration test was conducted against Mediroza General Hospital (https://medirozahospital.com) over a 5-day engagement period. The assessment identified **four critical or high-severity vulnerabilities** that collectively allowed a simulated attacker with no prior access to: bypass authentication and access confidential patient medical records, crack all document-level encryption protecting those records, and retrieve a publicly exposed database backup containing the full personal and financial details of 30 staff members and the hospital's complete shareholder ownership structure.

**Overall Risk Rating: CRITICAL**

The most severe finding — an unauthenticated, publicly accessible database backup — required no exploitation whatsoever. It was available to anyone who visited the correct URL. The SQL injection vulnerability that enabled access to patient records is a well-known, fully preventable class of vulnerability with a one-line code fix. Neither finding indicates sophisticated attack capability was required; both are accessible to a beginner attacker.

**Summary of Key Findings:**

| # | Finding | Severity |
| :---- | :---- | :---- |
| 1 | SQL Injection — Authentication Bypass (Patient Portal) | 🔴 Critical |
| 2 | Publicly Exposed Database Backup (/old/) | 🔴 Critical |
| 3 | Weak Password Policy on Protected Documents | 🟠 High |
| 4 | PDF Metadata Disclosure (IT Admin on Clinical Record) | 🟠 High |
| 5 | Directory Listing Enabled (/old/) | 🟠 High |
| 6 | Missing HTTP Security Headers | 🟡 Medium |
| 7 | CMS Version Disclosed in Page Meta Tags | 🟡 Medium |
| 8 | Staff Login — Hardened (Positive Control) | ℹ️ Informational |
| 9 | IDOR on Download Endpoint — Not Vulnerable | ℹ️ Informational |

---

### 02 — Scope and Methodology

**Target:** https://medirozahospital.com
**Type:** Black-box penetration test
**Authorization:** Written authorisation provided by Networkwalks
**Rules:** Target domain only. No social engineering. No denial of service. No testing outside agreed scope.

**Tools Used:**

| Tool | Version | Purpose |
| :---- | :---- | :---- |
| feroxbuster | Latest (Kali) | Directory and endpoint enumeration |
| Firefox Browser | Latest | Manual testing and WAF bypass |
| Networkwalks Password Cracker | Latest | PDF password hash cracking |
| pdf2john.py | Bundled with JTR | PDF hash extraction |

**Methodology — Order of Operations:**

1. **Active Reconnaissance** — feroxbuster directory enumeration to map the attack surface before any exploitation was attempted
2. **Authentication Surface Testing** — both login portals tested manually (default credentials first, then SQL injection payloads) in a real browser to avoid WAF detection
3. **Exploitation** — SQL injection authentication bypass on the patient portal to retrieve target files
4. **Post-Exploitation Data Extraction** — PDF hash extraction and password cracking on all retrieved files
5. **Server-Side Enumeration** — investigation of all discovered directories for further exposure
6. **Evidence Analysis** — review of all retrieved data including file metadata

**Limitations Encountered:**

A Web Application Firewall (WAF) was active on the target. Automated tool feroxbuster triggered rate-limiting and IP blocks when operating at standard thread counts or request speeds. This was mitigated by reducing thread counts on feroxbuster (`-t 5`) and conducting all authentication bypass testing manually in Firefox, which the WAF could not distinguish from legitimate user traffic.

---

### 03 — Findings and Proof of Exploitation

#### Finding 1 — SQL Injection Authentication Bypass (Patient Portal)

**Location:** https://medirozahospital.com/patient/login.php
**Parameter:** `username` (POST)
**Severity:** 🔴 Critical

**Description:** The patient portal login form concatenates user-supplied input directly into a SQL query without sanitisation. Entering a crafted payload in the username field comments out the password check, granting access as the first matching user with no password required.

**Payload Used:**
```
Username: admin' -- 
Password: [any value]
```

**Impact:** Unauthenticated access to the patient portal and all records held within it. Three confidential pathology lab reports were retrieved — S. Dlamini, P. Reddy, E. Thompson.
---

#### Finding 2 — Publicly Exposed Database Backup

**Location:** https://medirozahospital.com/old/mediroza_db_backup_2019.sql
**Severity:** 🔴 Critical

**Description:** A full MySQL database backup was stored in a publicly accessible directory with no authentication required. The file was downloadable directly from the browser by any visitor who knew or discovered the path. feroxbuster identified the `/old/` directory during reconnaissance.

**Impact:** The backup contained the complete personal details of 30 staff members (full names, email addresses, phone numbers, national ID numbers, job titles, and salaries) and the hospital's full shareholder ownership structure (names, shares held, share percentage, share class). This constitutes a severe data breach of personally identifiable information (PII) and confidential financial data.
---

#### Finding 3 — Weak Password Policy on Protected Documents

**Location:** patient_report_1.pdf, patient_report_2.pdf, patient_report_3.pdf
**Severity:** 🟠 High

**Description:** All three password-protected patient PDF files were cracked using a standard dictionary attack with the rockyou.txt wordlist. No advanced techniques, rule-based mangling, or brute force were required. Two of the three passwords cracked in under one second.


#### Finding 4 — PDF Metadata Disclosure (IT Admin on Clinical Record)

**Location:** patient_report_3.pdf — File Properties → Description → Author
**Severity:** 🟠 High

**Description:** The Author metadata field of `patient_report_3.pdf` (E. Thompson pathology report) contains `j.malik` rather than a clinical author. Cross-referencing this against the leaked staff database identifies **Jameel Malik, IT Systems Administrator** (j.malik@medirozahospital.com).

**Impact:** An IT administrator account has write access to clinical patient record systems. This violates the principle of least privilege and represents a significant access control gap between IT and clinical roles. It is also a compliance concern under healthcare data protection regulations — clinical records must be authored and managed exclusively by authorised clinical personnel.

#### Finding 5 — Directory Listing Enabled (/old/)

**Location:** https://medirozahospital.com/old/
**Severity:** 🟠 High

**Description:** The `/old/` directory has no index file and directory listing is enabled on the web server. Any visitor to this URL sees a full list of the directory's contents, including file names and sizes. This directly enabled the discovery and download of the database backup in Finding 2.


#### Finding 6 — Missing HTTP Security Headers

**Location:** All pages — HTTP response headers
**Severity:** 🟡 Medium

**Description:** The following security headers were absent from all HTTP responses, leaving the application exposed to a range of client-side attacks.

| Header | Purpose | Status |
| :---- | :---- | :---- |
| Content-Security-Policy | Prevents XSS and data injection | ❌ Missing |
| X-Frame-Options | Prevents clickjacking | ❌ Missing |
| X-Content-Type-Options | Prevents MIME sniffing | ❌ Missing |
| Strict-Transport-Security | Enforces HTTPS | ❌ Missing |

---

#### Finding 7 — CMS Version Disclosed in Page Meta Tags

**Location:** Homepage — HTML page source
**Severity:** 🟡 Medium

**Description:** The page source discloses `Mediroza CMS 1.4.2` in the HTML meta tags. This tells an attacker the exact software and version running the site, enabling targeted searches for known vulnerabilities specific to that version.

---

#### Finding 8 — Staff Login Hardened Against SQL Injection (Positive Control)

**Location:** https://medirozahospital.com/staff/login.php
**Severity:** ℹ️ Informational (Positive Finding)

**Description:** The staff login portal was tested with the same SQL injection payloads and default credential combinations used against the patient portal. All attempts failed. The Staff ID field appears to use parameterised queries or integer casting, making it resistant to the injection technique that compromised the patient portal.

This represents an inconsistency in the codebase — one portal is protected while the other is not — suggesting different developers or development periods were involved.

---

#### Finding 9 — IDOR on Patient Download Endpoint — Not Vulnerable

**Location:** https://medirozahospital.com/patient/download.php?id=
**Severity:** ℹ️ Informational (Tested, Not Vulnerable)

**Description:** After gaining access to the patient portal, the PDF download links were found to use a numeric `id` parameter (`download.php?id=1`). IDs 1 through 10 and ID 99 were tested. Only IDs 1, 2, and 3 returned valid files — all other IDs returned "report not found." Direct URL access without an active session redirected to the login page, confirming authentication is checked on this endpoint.

**Conclusion:** The download endpoint is not vulnerable to IDOR under current conditions. Only 3 records exist in the database and session authentication is enforced on direct access.

---

### 04 — Risk Rating

| # | Finding | Severity | Justification |
| :---- | :---- | :---- | :---- |
| 1 | SQL Injection — Authentication Bypass | 🔴 Critical | Unauthenticated access to confidential patient medical records. Trivial to exploit with a single payload in a browser. |
| 2 | Publicly Exposed Database Backup | 🔴 Critical | No exploitation required. Full PII and financial data of 30 staff freely downloadable by anyone. National ID numbers exposed — severe regulatory breach. |
| 3 | Weak Password Policy on Documents | 🟠 High | All 3 encrypted medical files cracked in under 15 seconds with a standard wordlist. Encryption provides no real protection. |
| 4 | PDF Metadata — IT Admin on Clinical Record | 🟠 High | Principle of least privilege violated. IT admin has write access to clinical systems. Compliance and access control failure. |
| 5 | Directory Listing Enabled | 🟠 High | Directly enabled download of the database backup. Any file placed in /old/ is publicly visible and downloadable. |
| 6 | Missing Security Headers | 🟡 Medium | Increases exposure to XSS, clickjacking, and MIME sniffing attacks. No direct exploitation in this engagement but raises overall attack surface. |
| 7 | CMS Version Disclosure | 🟡 Medium | Assists an attacker in identifying version-specific vulnerabilities. Low immediate impact but reduces attacker effort. |

---

### 05 — Recommendations and Remediation

#### Finding 1 — SQL Injection

**Fix: Use prepared statements (parameterised queries)**

Replace all raw SQL string concatenation with prepared statements:

```php
// Replace this:
$query = "SELECT * FROM users WHERE username = '" . $username . "'";

// With this:
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ?");
$stmt->execute([$username]);
```

This ensures user input is always treated as data and never interpreted as SQL, regardless of what characters it contains. Apply this fix to every database query in the application, not only the login form.

#### Finding 2 — Exposed Database Backup

**Immediate actions:**
1. Delete `mediroza_db_backup_2019.sql` from the web-accessible directory immediately
2. Move all database backups to a location outside the web root (e.g. `/home/backups/` not `/public_html/old/`)
3. Restrict backup file access to authorised server administrators only
4. Audit all other directories for similar exposed files
5. Notify affected staff of the PII exposure and comply with applicable data breach notification obligations

#### Finding 3 — Weak Password Policy

**Enforce the following policy on all protected documents:**
- Minimum 12 characters
- Mix of uppercase, lowercase, numbers, and special characters
- No dictionary words, keyboard walks (e.g. `1qaz2wsx`), or common phrases
- Use a password manager to generate and store document passwords
- Different password for each protected document

#### Finding 4 — PDF Metadata / Access Control

1. Audit all user accounts with write access to the patient records system
2. Remove access for any account (including IT administrator accounts) that does not require clinical write access as part of their role
3. Implement role-based access control (RBAC) — clinical records should only be writable by verified clinical personnel
4. Scrub metadata from all patient-facing documents before distribution using a tool such as `exiftool -all= filename.pdf`

#### Finding 5 — Directory Listing

Disable directory listing in the web server configuration:

**For Apache (`.htaccess` or `httpd.conf`):**
```
Options -Indexes
```

**For LiteSpeed (as used on this server):**
Disable "Directory Listing" in the LiteSpeed admin panel under Virtual Host → General → Directory Index settings.

#### Finding 6 — Missing Security Headers

Add the following to all HTTP responses in the server or application configuration:

```
Content-Security-Policy: default-src 'self'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

#### Finding 7 — CMS Version Disclosure

Remove or mask the CMS version from all HTML meta tags and HTTP headers. There is no user-facing reason to disclose software version information publicly.

---

## 6. Conclusion

During Week 4 of my Cybersecurity & Ethical Hacking training at Networkwalks, I completed a full black-box penetration test against the Mediroza General Hospital training environment. Beginning with no credentials or prior knowledge of the system, I used feroxbuster to map the attack surface, identified and exploited a SQL injection authentication bypass on the patient portal to retrieve three confidential medical records, cracked the password protection on all three files using John the Ripper, and discovered a publicly accessible database backup containing the sensitive personal and financial data of 30 hospital employees and the hospital's complete shareholder structure.

The engagement demonstrated that the two most critical findings — the SQL injection vulnerability and the exposed database backup — required minimal technical skill to exploit. The SQL injection was resolved with a single payload typed into a browser. The database backup required only knowing the directory path. Neither required advanced tooling or specialist knowledge. This underscores that the most dangerous vulnerabilities in web applications are often the simplest, and that secure coding practices and basic server hygiene are more valuable than any security tool.

I additionally tested surfaces that other analysts may not have covered — the staff login portal was tested for both default credentials and SQL injection and confirmed to be hardened, which represents a positive security control worth recognising. The download endpoint was probed for IDOR and confirmed not vulnerable. Thorough negative findings are part of a complete security assessment.

---

*This report is submitted as part of the Networkwalks B083 cybersecurity training program. All testing was conducted in a controlled, authorised educational environment. These techniques must never be applied to any system without explicit written permission from the owner.*

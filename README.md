# Mediroza General Hospital — Black-Box Web Application Penetration Test

**Networkwalks B082 — Week 4**

> Authorized security assessment conducted against the designated target in a controlled environment.

---

## ⚠️ Disclaimer

This project was conducted as an **authorized black-box penetration test** within the scope provided for the Networkwalks B082 Week 4 assignment.

Testing was limited to the authorized target:

`https://medirozahospital.com`

No social engineering or denial-of-service testing was performed.

Sensitive patient, employee, and corporate information discovered during testing has **not been reproduced unnecessarily** in this public write-up.

---

# 1. Executive Summary

An authorized black-box penetration test was performed against the Mediroza General Hospital web application.

The assessment identified multiple security weaknesses that could be chained together to obtain unauthorized access to patient laboratory reports and sensitive internal business information.

The most significant vulnerability was an **SQL injection vulnerability in the authentication mechanism**. A crafted username input successfully bypassed authentication and provided access to the patient portal.

The patient portal exposed three laboratory reports. The retrieved PDF documents were password protected, but controlled password-recovery testing successfully recovered the passwords for all three files.

Further examination of the decrypted report metadata revealed a comment referencing an old database backup located under `/old/`.

The `/old/` directory was publicly accessible and exposed:

`mediroza_db_backup_2019.sql`

The database backup contained sensitive internal information, including:

- Employee names and monthly salaries
- Shareholder names
- Share percentages
- Shares held
- Share classes

The assessment therefore demonstrated a complete attack chain from **web authentication bypass → patient report access → document password recovery → metadata discovery → database backup exposure → internal information disclosure**.

---

# 2. Scope

## Target

```text
https://medirozahospital.com
```

### Assessment Type

Black-box web application penetration test.

### Restrictions

The assessment followed the authorized project scope.

The following activities were not performed:
- Social engineering
- Denial-of-service testing
- Testing outside the authorized target

---

# 3. Methodology

The assessment followed a black-box methodology consisting of:

1. Reconnaissance
2. Web application enumeration
3. Identification of exposed application paths
4. Authentication analysis
5. Controlled SQL injection testing
6. Authentication-bypass validation
7. Patient-report access testing
8. Retrieval of the three available PDF reports
9. PDF password-recovery testing
10. PDF metadata analysis
11. Investigation of the database-backup location discovered through metadata
12. Analysis of exposed database contents
13. Documentation of findings and remediation recommendations

---

# 4. Attack Chain

The following attack chain was demonstrated during the assessment:

```text
Public Web Application
        │
        ▼
Exposed Application Paths
        │
        ▼
Patient Login
        │
        ▼
SQL Injection
        │
        ▼
Authentication Bypass
        │
        ▼
Patient Portal
        │
        ▼
Three Laboratory Reports
        │
        ▼
Password-Protected PDFs
        │
        ▼
Password Recovery
        │
        ▼
PDF Metadata
        │
        ▼
/old/ Directory
        │
        ▼
Public Database Backup
        │
        ▼
Sensitive Internal Information
```

---

# 5. Findings Summary

| ID | Finding | Severity |
| :--- | :--- | :--- |
| **F-01** | SQL Injection Authentication Bypass | 🔴 Critical |
| **F-02** | Unauthorized Exposure of Patient Laboratory Reports | 🔴 Critical |
| **F-03** | Weak/Recoverable PDF Password Protection | 🟠 High |
| **F-04** | Publicly Accessible Database Backup | 🔴 Critical |
| **F-05** | Exposure of Employee Salary Information | 🔴 Critical |
| **F-06** | Exposure of Shareholder Information | 🟠 High |

---

# 6. F-01 — SQL Injection Authentication Bypass

**Severity:** 🔴 Critical

### Description
The application's authentication mechanism accepted crafted SQL input through the username field. 

A controlled SQL injection test successfully resulted in authentication bypass. 

The application returned a redirect to:
`portal.php`

The authenticated portal was subsequently accessible.

### Evidence
The successful test demonstrated:
- The login endpoint accepted username and password parameters.
- Crafted SQL input was accepted by the application.
- The response redirected to `portal.php`.
- The resulting session provided access to the patient portal.

### Impact
An unauthenticated attacker could potentially bypass the application's authentication controls and access restricted functionality. 

Because the application contains patient information, exploitation can result in significant confidentiality loss.

### Remediation
- Use parameterized SQL queries/prepared statements.
- Never construct SQL queries directly from user-controlled input.
- Apply appropriate server-side input handling.
- Use least-privilege database accounts.
- Perform secure code review of authentication functionality.
- Monitor application logs for evidence of exploitation.

---

# 7. F-02 — Unauthorized Exposure of Patient Laboratory Reports

**Severity:** 🔴 Critical

### Description
After successful authentication bypass, the patient portal displayed three laboratory reports. 

The reports were accessible through download functionality using report identifiers. 

Three PDF reports were successfully retrieved.

### Evidence
The portal contained three available reports:
- Report ID: 1
- Report ID: 2
- Report ID: 3

The files were valid one-page PDF documents and were password protected.

### Impact
Patient laboratory reports contain confidential medical information. 

Unauthorized access to these documents represents a serious patient confidentiality risk.

### Remediation
- Enforce authentication and authorization on every report request.
- Verify that the authenticated user is authorized to access the requested report.
- Do not rely solely on predictable numeric report IDs.
- Review access-control logic for IDOR-style weaknesses.
- Log and monitor report access.

---

# 8. F-03 — Weak/Recoverable PDF Password Protection

**Severity:** 🟠 High

### Description
All three retrieved PDF files were password protected. 

Controlled password-recovery testing successfully recovered the passwords for all three reports. 

The PDF files used the Standard Security Handler with 128-bit encryption, Revision 3.

### Password Recovery Evidence
The recovered passwords were:

| Report | Result |
| :--- | :--- |
| **Report 1** | `123456` |
| **Report 2** | `password` |
| **Report 3** | `!@#$%^&` |

> *These credentials are included here strictly as proof of the authorized assessment and should not be reused against any system.*

### Impact
The passwords were easily guessable/common values. 

Therefore, obtaining the encrypted PDF files was sufficient to make recovery of their contents practical.

### Remediation
- Use strong, unique, randomly generated passwords.
- Avoid common passwords and predictable patterns.
- Use modern document encryption where supported.
- Consider access-controlled document delivery rather than relying solely on PDF passwords.

---

# 9. F-04 — Publicly Accessible Database Backup

**Severity:** 🔴 Critical

### Description
Metadata from one decrypted report contained a comment referencing an old database backup:

> `DB backup moved to /old before site migration, do not delete`

Investigation of the referenced location revealed a publicly accessible directory. 

The directory exposed:
`mediroza_db_backup_2019.sql`

### Evidence
The `/old/` directory displayed a directory listing and exposed the SQL database backup.

### Impact
A database backup can contain a large amount of sensitive information in a single file. 

Exposure of this backup significantly increased the impact of the other vulnerabilities identified during the assessment.

### Remediation
- Remove database backups from the web document root.
- Disable directory listing.
- Store backups outside publicly accessible directories.
- Apply strict access controls to backup storage.
- Implement backup lifecycle and deletion procedures.
- Scan web deployments for backup files and temporary artifacts.

---

# 10. F-05 — Exposure of Employee Salary Information

**Severity:** 🔴 Critical

### Description
The exposed SQL backup contained a staff table with a `monthly_salary_zar` field. 

Review of the database identified 30 staff salary records. 

The relevant staff data included employee names and monthly salary values.

### Impact
Unauthorized disclosure of employee compensation information can create:
- Privacy risks
- Internal security risks
- Legal/compliance concerns
- Reputational damage
- Employee trust issues

### Remediation
- Remove the database backup from public access.
- Restrict HR/payroll information to authorized personnel.
- Review historical backups for unnecessary sensitive information.
- Encrypt and securely store backups.
- Apply appropriate access controls to HR information.

---

# 11. F-06 — Exposure of Shareholder Information

**Severity:** 🟠 High

### Description
The exposed database backup contained a shareholders table. 

The table included:
- `shareholder_name`
- `share_percent`
- `shares_held`
- `share_class`

A total of 10 shareholder records were identified.

### Impact
Unauthorized disclosure of corporate ownership information can expose sensitive business information and create legal, operational, and reputational risks.

### Remediation
- Remove the backup from public access.
- Restrict ownership information to authorized personnel.
- Secure historical database backups.
- Review data retention and classification policies.
- Implement appropriate backup access controls.

---

# 12. Evidence Register

| Evidence ID | Description |
| :--- | :--- |
| **E-01** | SQL injection authentication bypass |
| **E-02** | Patient portal showing three reports |
| **E-03** | Three retrieved password-protected PDFs |
| **E-04** | PDF password recovery |
| **E-05** | PDF metadata containing database-backup reference |
| **E-06** | `/old/` directory exposing SQL backup |
| **E-07** | Staff salary information |
| **E-08** | Shareholder information |

---

# 13. Overall Risk

The vulnerabilities should not be considered completely independent.

The demonstrated attack chain allowed an attacker to move from an externally accessible web application to:
1. Authentication bypass
2. Patient report access
3. Recovery of document passwords
4. Discovery of internal infrastructure information
5. Public access to a database backup
6. Exposure of employee salary information
7. Exposure of shareholder information

The combination of these weaknesses represents a **high-risk security condition**.

---

# 14. Remediation Priorities

### Immediate
- Remove the exposed SQL database backup.
- Disable public directory indexing.
- Fix the SQL injection vulnerability.
- Review authentication and authorization controls.
- Review access to all patient documents.
- Invalidate potentially exposed sessions or links where appropriate.

### High Priority
- Replace weak document passwords.
- Audit historical backups.
- Remove unnecessary sensitive information from retained backups.
- Implement continuous scanning for exposed backup files.

### Ongoing
- Conduct secure-code reviews.
- Perform recurring penetration testing.
- Implement security testing as part of the software-development lifecycle.
- Review backup-storage and retention procedures regularly.

---

# 15. Conclusion

The assessment identified several critical weaknesses in the Mediroza General Hospital web application and supporting file-management practices.

The most significant issue was the demonstrated SQL injection authentication bypass, which provided access to restricted patient functionality.

The subsequent discovery of weak PDF passwords and a publicly accessible database backup demonstrated how multiple weaknesses could be chained together to expose highly sensitive information.

The publicly accessible database backup represented a particularly serious configuration and information-disclosure issue because it contained both employee salary information and shareholder records.

### Highest-Priority Remediation Actions:
1. Fix the SQL injection vulnerability.
2. Strengthen authentication and authorization.
3. Remove the database backup from public access.
4. Disable directory listing.
5. Secure patient documents.
6. Strengthen document encryption/password practices.

---

# 16. Security Notes

This write-up intentionally avoids publishing:
- Patient medical information
- National identification numbers
- Personal phone numbers
- Personal email addresses
- Full database dumps
- Unnecessary confidential records

### Tools Used
The assessment involved standard security-testing and analysis tooling, including:
- Browser-based testing
- HexStrike
- PDF analysis tools (`pdfcrack`, `qpdf`, `pdfinfo`, `exiftool`)
- Linux command-line utilities
---
## Author
Rachel Debbarma

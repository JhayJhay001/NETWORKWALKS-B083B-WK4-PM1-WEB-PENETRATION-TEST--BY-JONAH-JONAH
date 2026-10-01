# NETWORKWALKS-B083B-WK4-PM1-WEB-PENETRATION-TEST--BY-JONAH-JONAH
# Black-Box Web Application Penetration Test — Mediroza Hospital

**NETWORKWALKS Cybersecurity Internship — Week 4**  
**Batch B082**

This project documents an authorised black-box security assessment of the **Mediroza Hospital web application**. The Week 4 challenge required more than identifying possible vulnerabilities: each milestone had to be demonstrated through actual access, retrieval, recovery and evidence-backed reporting.

The assessment developed into a chained compromise involving exposed application structure, authentication weaknesses, weak document protection and publicly accessible legacy data.

> **Public disclosure note:** This repository contains a sanitised portfolio record. Credentials, patient information, recovered document passwords, personal data, sensitive paths and other unnecessary live-target details have been redacted or omitted.

## Project Objectives

The assessment was divided into four milestones:

| Milestone | Objective | Status |
| --- | --- | --- |
| **M1** | Gain authorised access to the patient portal and retrieve three encrypted reports | Complete |
| **M2** | Recover the passwords protecting all three reports and verify access | Complete |
| **M3** | Recover exposed payroll and shareholder information | Complete |
| **M4** | Produce a professional evidence-backed penetration-test report | Complete |

## Environment & Tools

The assessment was performed primarily from **Kali Linux 2026.2**.

Tools and techniques used included:

- Burp Suite Community Edition
  - Repeater
  - Intruder
- `curl`
- `grep`
- John the Ripper
- `pdf2john.pl`
- `rockyou.txt`
- Browser-based application testing
- HTTP response analysis
- Directory-index analysis
- Offline password recovery
- SQL-backup inspection
- Evidence preservation throughout the assessment

## Assessment Journey

### 1. Attack the website and find the 3 confidential PDF lab reports of patients**

The assessment began with ordinary web reconnaissance and route discovery.

Several initially assumed application routes returned `404 Not Found`, demonstrating that simply guessing conventional filenames was producing more noise than useful information.

The turning point came when a parent application directory returned an **automatic directory index**.

Rather than continuing to guess, the application itself had exposed part of its internal route structure.

![Redacted patient directory index](evidence/m1-initial-access/01-patient-directory-index-redacted.png)

The exact exposed routes have been redacted from this public repository, but the result established an important finding: **directory indexing was enabled in a sensitive application area**.

Further testing showed that some discovered resources redirected unauthenticated requests, while others denied direct browsing. This indicated that the path to the required reports ran through authentication rather than direct file access.

### 2. Understanding Authentication Behaviour

Before attempting a password attack, I compared how the login page responded to different inputs.

Two behaviours stood out.

First, different error messages were returned for an unknown account and a recognised account with the wrong password. This allowed a valid administrative account to be identified before password testing.

Second, malformed input produced a raw MySQL warning from `mysqli_query()`.

![SQL error disclosure](evidence/m1-initial-access/02-sql-error-disclosure-redacted.png)

This demonstrated:

- backend database information disclosure;
- unsafe query handling;
- user-controlled input reaching the SQL parser.

However, this distinction mattered:

**The SQL behaviour demonstrated a vulnerability, but the SQL-injection-style authentication bypasses I tested did not successfully authenticate me.**

Rather than claim an exploit that had not been demonstrated, I treated the SQL behaviour as its own finding and changed approach.

### 3. Targeted Credential Testing

With a valid administrative username already identified, the login request was moved into **Burp Suite**.

Repeater was first used to understand the normal failure pattern. Failed authentication attempts consistently returned HTTP `200`.

The password field was then isolated in **Intruder Sniper mode**, using a small dictionary of common password candidates.

Burp Community Edition throttled the attack, but the limitation affected speed rather than whether the attack could run.

One candidate produced a clearly different result:

- HTTP `302` instead of `200`;
- a different response length;
- an authenticated redirect.

![Redacted authentication result](evidence/m1-initial-access/03-authentication-success-redacted.png)

The credential itself, session information and redirect destination are intentionally redacted.

The important evidence was the response anomaly: the server provided a measurable success signal that distinguished the valid credential from the failed candidates.

### 4. Authenticated Report Access

The recovered credential was verified through normal browser authentication.

The portal then exposed **three encrypted pathology reports**, completing the access portion of Milestone 1.

![Redacted authenticated report access](evidence/m1-initial-access/04-authenticated-report-access-redacted.png)

Patient names, laboratory references and the authenticated route are removed from the public screenshot.

The reports were then downloaded for offline analysis.

### 5. Encrypted PDF Recovery

Milestone 2 introduced a separate problem: all three downloaded reports were password-protected.

The first attempt extracted a John-compatible PDF representation, but John returned:

```text
No password hashes loaded
```

Repeating the extraction locally with `pdf2john.pl` produced the same problem.

The issue turned out not to be the encrypted documents or the wordlist.

It was a **local compatibility mismatch**.

The extractor represented the PDF permission field as the unsigned 32-bit value:

```text
4294967292
```

while the installed John PDF loader expected its signed equivalent:

```text
-4
```

Normalising only that metadata field allowed John to recognise the hashes correctly:

```text
Loaded 1 password hash
(PDF [MD5 SHA2 RC4/AES 32/64])
```

This correction did not modify the encrypted document data or password verifier; it resolved how one field was represented between the extractor and the installed loader.

After the hashes loaded successfully, they were tested locally against `rockyou.txt`.

![Redacted John the Ripper recovery](evidence/m2-document-recovery/01-jtr-offline-recovery-redacted.png)

All three report passwords were recovered and then independently validated by opening the original PDFs.

The public result file preserves the success state without publishing the passwords:

[`02-jtr-results-redacted.txt`](evidence/m2-document-recovery/02-jtr-results-redacted.txt)

### 6. Legacy Data Exposure

Milestone 3 shifted the assessment away from the patient portal.

A legacy application area also exposed an automatic directory index.

![Redacted legacy directory index](evidence/m3-data-exposure/01-legacy-directory-index-redacted.png)

The indexed area exposed a database backup that was retrievable **without authentication**.

The exact filename and live retrieval path are omitted publicly.

The retained response headers demonstrate that the SQL resource was successfully returned by the server:

[`02-backup-download-headers.txt`](evidence/m3-data-exposure/02-backup-download-headers.txt)

Inspection confirmed database structures containing both employee and shareholder information.

![Redacted backup structure](evidence/m3-data-exposure/03-backup-structure-redacted.png)

The recovered material included:

- **30 employee records**
- roles and departments
- salary information
- contact-information fields
- national-identifier fields
- **10 shareholder records**
- ownership percentages
- shares held
- share classes

Individual records and values are intentionally excluded from this repository.

## Attack Chain

What made the assessment significant was not one isolated vulnerability.

The compromise emerged from several weaknesses interacting with one another:

```text
Directory indexing
        ↓
Application structure exposed
        ↓
Authentication behaviour analysed
        ↓
Valid administrative account identified
        ↓
Targeted password testing
        ↓
Distinct HTTP 302 success signal
        ↓
Authenticated portal access
        ↓
Three encrypted reports retrieved
        ↓
Offline PDF password recovery
        ↓
All three reports opened

Separately:

Legacy directory indexing
        ↓
Database backup exposed
        ↓
Unauthenticated retrieval
        ↓
Payroll and ownership data disclosed
```

No server-side shell, malware, persistence, lateral movement, denial of service or destructive changes were required.

The compromise was achieved through ordinary web requests, application-response analysis, weak credentials, weak document passwords and exposed data.

## Findings

Seven findings were formally recorded:

| ID | Severity | Finding |
| --- | --- | --- |
| **F01** | **Critical** | Public legacy database backup |
| **F02** | **Critical** | Weak administrative portal credential |
| **F03** | **High** | Insufficient report access controls |
| **F04** | **High** | Weak PDF password protection |
| **F05** | **Medium** | SQL error disclosure and unsafe query handling |
| **F06** | **Medium** | Username enumeration |
| **F07** | **Medium** | Directory indexing and predictable routes |

### Overall Risk: Critical

The severity reflects the **combined attack path**, not merely the existence of individual findings.

Weaknesses that might appear limited in isolation collectively exposed patient, employee, financial and ownership information.

## Remediation Priorities

### Immediate — 0 to 24 Hours

- Remove exposed backup data from web-accessible locations.
- Disable directory indexing.
- Reset affected credentials.
- Invalidate active sessions.
- Preserve relevant server and access logs.
- Begin privacy-incident assessment.

### Urgent — 1 to 7 Days

- Replace unsafe login queries with prepared statements.
- Return generic authentication errors.
- Implement rate limiting and account lockout controls.
- Enforce server-side authorisation for every report request.
- Store sensitive reports outside the public web root.

### Near Term — 8 to 30 Days

- Introduce MFA for privileged accounts.
- Replace predictable document passwords with high-entropy secrets or authenticated portal delivery.
- Scan deployments for exposed backups and logs.
- Introduce secure-code-review and security-testing gates.

### Ongoing

- Periodically review access rights.
- Monitor abnormal login and download behaviour.
- Test backup-handling controls.
- Rotate credentials appropriately.
- Repeat penetration testing after material application changes.

## What I Learned

### Failed attempts are still evidence

The early `404`, `302` and `403` responses were not wasted effort. They helped distinguish nonexistent resources, authentication boundaries and inaccessible directories.

### A vulnerability is not automatically a successful exploit

The application clearly demonstrated unsafe SQL handling, but my tested SQL-injection authentication bypasses did not succeed.

That difference matters in penetration-test reporting.

The evidence justified:

> SQL error disclosure and unsafe query handling.

It did **not** justify:

> SQL injection successfully bypassed authentication.

### Response behaviour can become an attack signal

Understanding the normal login failure pattern made the successful Burp Intruder candidate obvious.

The decisive clue was not simply the password candidate itself; it was the server responding differently.

### Tool failure does not always mean bad input

John initially refusing to load the extracted PDF hashes looked like a cracking problem.

It was actually a compatibility problem between the extractor's representation of one permission field and the installed John loader.

Understanding the format solved the issue without changing the encrypted data.

### Weaknesses compound

No single advanced exploit produced the overall compromise.

Directory indexing, information disclosure, username enumeration, a weak credential, predictable report access, weak document passwords and an exposed backup combined into a **Critical** result.

### Evidence should be collected during the work

The evidence folders were created before the assessment progressed.

That meant important requests, responses and results were preserved when they occurred rather than reconstructed after the fact.

### Investigation guidance is not assessment evidence

During the working process, external material helped point the investigation away from repeated dead ends.

That material was not used as proof in the final assessment.

The final report was deliberately based only on results independently reproduced against the authorised target and preserved in the local evidence set.

## Public Assessment Report

A sanitised version of the professional Week 4 assessment is included here:

**[View the Public Security Assessment](report/Prince_Manu_Gyebi_NetworkWalks_Week_4_Mediroza_Security_Assessment_PUBLIC.pdf)**

The public report retains the methodology, findings, risk analysis, remediation plan and technical notes while removing credentials, patient data, individual payroll and shareholder information, sensitive paths and unnecessary live-target details.

## Repository Structure

```text
networkwalks-B082-week4-mediroza-pentest/
├── evidence/
│   ├── m1-initial-access/
│   │   ├── 01-patient-directory-index-redacted.png
│   │   ├── 02-sql-error-disclosure-redacted.png
│   │   ├── 03-authentication-success-redacted.png
│   │   └── 04-authenticated-report-access-redacted.png
│   │
│   ├── m2-document-recovery/
│   │   ├── 01-jtr-offline-recovery-redacted.png
│   │   └── 02-jtr-results-redacted.txt
│   │
│   └── m3-data-exposure/
│       ├── 01-legacy-directory-index-redacted.png
│       ├── 02-backup-download-headers.txt
│       └── 03-backup-structure-redacted.png
│
├── report/
│   └── Prince_Manu_Gyebi_NetworkWalks_Week_4_Mediroza_Security_Assessment_PUBLIC.pdf
│
└── README.md
```

## Evidence & Disclosure

The original working evidence remains local.

This repository contains only selected and sanitised artefacts sufficient to demonstrate the assessment process and findings without unnecessarily publishing:

- recovered credentials;
- active session values;
- patient identities or clinical information;
- recovered PDF passwords;
- personal contact information;
- national identifiers;
- individual salaries;
- shareholder identities or individual ownership positions;
- exact sensitive backup paths;
- the raw database backup.

## Ethical Scope

This assessment was conducted as an **authorised NETWORKWALKS internship security exercise** against the assigned Mediroza Hospital training target.

Testing remained within the defined scope.

The assessment did not attempt:

- persistence;
- destructive modification;
- lateral movement;
- denial of service;
- malware deployment;
- alteration of target data.

## Author

**Jonah Jonah**

Final-year BSc Information Security student  
NETWORKWALKS Cybersecurity Internship — Batch B082

- [LinkedIn]([https](https://www.linkedin.com/in/jonah-jonah/)
- [GitHub](https://github.com/JonahJonah)

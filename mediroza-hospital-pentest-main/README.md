# Professional Penetration Testing & Vulnerability Assessment Report

**CONFIDENTIAL — AUTHORISED PERSONNEL ONLY**

| Field | Value |
|---|---|
| Target Domain | `medirozahospital.com` |
| Engagement Type | Full Black-box Penetration Test & Vulnerability Assessment |
| Project Batch | Batch B083 / Week 4 |
| Status | Full & Final Report |
| Overall Risk Rating | **CRITICAL** |

---

## 1. Executive Summary

A comprehensive black-box penetration test was executed against the web
infrastructure of Mediroza General Hospital. The objective was to evaluate
the external defensive posture of the perimeter, identify system
vulnerabilities, and safely assess real-world business risk in an
authorized, controlled environment.

The assessment uncovered critical structural and configuration flaws across
the application layer and asset infrastructure:

- **Critical Authentication Bypass (SQLi):** An input validation flaw on the
  Patient Portal enabled an attacker to bypass authentication and acquire
  administrative access.
- **Sensitive Database Exposure (Information Disclosure):** An unindexed
  legacy deployment backup directory (`/old/`) left a full production
  database schema publicly accessible.
- **Weak Encryption Configurations:** Patient document backups used outdated
  encryption settings, fully compromised via offline dictionary attack.

Immediate implementation of the remediations in Section 6 is strongly
advised to protect corporate assets and Patient Health Information (PHI).

---

## 2. Scope and Methodology

### Engagement Scope

- **Target Host:** `medirozahospital.com`
- **Operational Restrictions:** Testing was strictly limited to the target
  domain. Denial-of-Service simulations and social engineering (phishing)
  were explicitly prohibited.
- **Authorization:** Written authorization for this engagement was granted
  by Networkwalks Academy as part of a supervised training program.

### Toolset Deployed

1. **Perimeter Reconnaissance:** `whois`, `whatweb`, `nslookup`, `curl -I`, `wafw00f`, `dnsrecon`
2. **Network Infrastructure Scanning:** `nmap` / Zenmap
3. **Directory Enumeration:** `gobuster` (v3.8.2)
4. **Cryptographic Signature Parsing:** Networkwalks Academy Hash Calculator
5. **Offline Dictionary Attack:** Networkwalks Academy Password Cracker
6. **Data Transformation & Structural Parsing:** `qpdf`, `exiftool` (v13.55)

---

## 3. Findings & Proof of Exploitation

### Phase 1 — Perimeter Reconnaissance and Network Mapping

- **Domain Tracking (`whois`):** Registrant verified, creation date 2026-08-14.
  <img width="1300" height="625" alt="Screenshot_2026-09-29_11-13-48" src="https://github.com/user-attachments/assets/c594b9f6-f70f-41ae-9b5f-d9f20f0f79e4" />

- **DNS Mapping (`nslookup`):** Host resolved to `199.188.xxx.xx`.
- **WAF Fingerprinting (`wafw00f`):** Identified an active LiteSpeed-based WAF.
  <img width="1054" height="566" alt="Screenshot_2026-09-29_11-43-41" src="https://github.com/user-attachments/assets/294e05b7-aaf5-417a-9dfc-a732a8179a70" />

- **HTTP Header Inspection (`curl -I`):** Confirmed LiteSpeed application engines at the edge.
<img width="651" height="605" alt="Screenshot_2026-09-29_11-16-39" src="https://github.com/user-attachments/assets/41a16012-ec32-492e-b7bb-5e3f4e6572f0" />


**Nmap service enumeration results:**

| Port(s) | Service |
|---|---|
| 21/tcp | Pure-FTPd |
| 26/587/465/tcp | Exim smtpd 4.100.1 |
| 80/tcp, 443/tcp | HAProxy HTTP proxy |
| 110/143/993/995/tcp | Dovecot pop3d/imapd |
<img width="1366" height="732" alt="Screenshot_2026-09-29_11-47-46" src="https://github.com/user-attachments/assets/41416bee-ec27-4a7d-9f9e-b6370b5e2b31" />



### Phase 2 — Web Directory Bruteforcing

```
gobuster dir -u https://medirozahospital.com \
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt \
  -s "301,302,307,401,403" -b ""
```

**Discovered paths:**

```
/cgi-bin     (301)
/staff       (301)
/mailman     (301)
/pipermail   (301)
/old         (301)  <-- Highly Vulnerable Endpoint
/webmail     (403)
```
<img width="1158" height="604" alt="Screenshot_2026-09-28_00-42-57" src="https://github.com/user-attachments/assets/ac2bdf48-c21c-4b47-928d-6594425f9923" />

Inspection of the unindexed `/old/` directory revealed a publicly exposed
plaintext database archive, `mediroza_db_backup_2019.sql`, containing staff
payroll tables and corporate shareholder data.

<img width="1180" height="666" alt="Screenshot_2026-09-28_00-45-21" src="https://github.com/user-attachments/assets/d05a3329-d5ee-42c4-aa9d-f4479327273f" />

### Phase 3 — Application Logic Subversion (Patient Portal SQL Injection)

- `/staff/login.php`: Returned uniform "Invalid username or password" errors — properly mitigated against enumeration and SQLi.
- `/patient/login.php`: Vulnerable to **username enumeration** — an unknown username returned `Username not found`, while a valid username (`admin`) returned `Incorrect password`.
<img width="940" height="576" alt="Screenshot_2026-09-29_12-48-59" src="https://github.com/user-attachments/assets/0c301872-1ab1-4b9d-9c71-e9b47177ea85" />

A test payload (`admin' OR '1'='1`) triggered a verbose SQL error, confirming
the query was unsanitized:

```
Warning: mysqli_query(): You have an error in your SQL syntax; check the
manual that corresponds to your MySQL server version for the right syntax
to use near '' OR '1'='1' at line 1
```
<img width="1109" height="595" alt="Screenshot_2026-09-29_12-54-41" src="https://github.com/user-attachments/assets/f16cd695-d751-4714-bbe9-f347c7b38353" />

**Exploitation — inline SQL comment authentication bypass:**

- Username: `admin' -- -`
- Password: *(blank)*

Resulting backend query:

```sql
SELECT * FROM patients WHERE username = 'admin' -- -' AND password = '';
```

The `-- -` sequence commented out the password check entirely, authorizing
an admin session with no password. This granted access to a restricted
patient workspace and allowed exfiltration of 3 password-protected patient
PDF lab reports.
<img width="1366" height="620" alt="Screenshot_2026-09-28_20-08-03" src="https://github.com/user-attachments/assets/2555d5d6-22f1-41c8-8192-9c1d7e3f7b0d" />

### Phase 4 — Cryptographic Key Recovery & Password Cracking

1. **Hash extraction:** The PDFs' encryption metadata confirmed a weak
   **PDF 1.4 Standard V2.3 (128-bit key)** configuration.
<img width="738" height="599" alt="Screenshot_2026-09-28_21-19-48" src="https://github.com/user-attachments/assets/3eaa2abf-1fe9-4434-8282-64cec19360a9" />

2. **Dictionary attack:**
   - Reports 1 & 2: Cracked almost instantly against a built-in wordlist — weak, default credentials.
   - Report 3: Resisted the default wordlist; required the `rockyou.txt` dictionary and ~1.5 hours of processing before the password was recovered.
<img width="645" height="601" alt="Screenshot_2026-09-28_21-20-27" src="https://github.com/user-attachments/assets/10052483-dcfe-461f-b96c-876403da6be5" />

### Phase 5 — Document Metadata Mapping & Verification

Recovered passwords were used to decrypt the PDFs, followed by metadata
inspection:

```
qpdf --password=<recovered_password> --decrypt unlocked_report.pdf unlocked_report_decrypted.pdf
exiftool unlocked_report_decrypted.pdf
```

One decrypted report contained an administrative comment left by an internal
IT staff member:

> *"DB backup moved to /old before site migration, do not delete"*

This definitively linked the exposed `/old/` database backup to the same
infrastructure being actively used by the application — confirming the
leaked SQL file was a live representation of the hospital's internal
database, not a stale artifact.
<img width="922" height="487" alt="Screenshot_2026-09-29_13-22-56" src="https://github.com/user-attachments/assets/5f4ac4e0-e276-405e-9ff1-aba6c463ecaa" />

---

## 4. Exposed Business Records (Exfiltrated Data Summary)

The exposed `mediroza_db_backup_2019.sql` file contained full staff payroll
and shareholder tables. Names have been anonymized below; the structural
finding (an entire live database dump being world-readable) is the material
point.

### A. Staff Payroll Table (sample structure, anonymized)

| ID | Role | Department | Monthly Salary Band |
|---|---|---|---|
| 1 | Chief Pathologist | Diagnostics Lab | R130k–R140k |
| 2 | Chief Financial Officer | Finance | R150k–R160k |
| 3 | Medical Director | Management | R160k–R170k |
| 9 | IT Systems Administrator | IT | R55k–R60k |
| … | *(27 additional records — full role/department spread across clinical, nursing, admin, and operations staff)* | | |

### B. Shareholder / Equity Table (sample structure, anonymized)

| ID | Shareholder | Share % | Share Class |
|---|---|---|---|
| 1 | Shareholder A (individual) | 18.00% | Ordinary |
| 2 | Corporate Holding Entity | 15.00% | Ordinary |
| 3 | Shareholder B (individual) | 12.00% | Ordinary |
| … | *(7 additional equity holders)* | | |

---

## 5. Visual Evidence Analysis (Exfiltrated Document Records)

Three exfiltrated patient lab reports were reviewed to confirm real clinical
data was present in the compromised documents (patient identifiers redacted):

1. **Patient A:** Full Blood Count — elevated White Cell Count, flagged HIGH, consistent with an active infection profile.
2. **Patient B:** Lipid Profile — total cholesterol, LDL, and triglycerides all flagged HIGH risk.
3. **Patient C:** Blood biochemistry — haemoglobin, ferritin, and vitamin D flagged LOW. This document's metadata contained the comment trace that led to the root-cause discovery in Phase 5.

---

## 6. Remediation Recommendations

### 6.1 Harden Authentication via Parameterized Queries

Replace dynamic SQL string concatenation with prepared statements:

```php
$stmt = $conn->prepare("SELECT id, password_hash FROM patients WHERE username = ?");
$stmt->bind_param("s", $userInputUsername);
$stmt->execute();
$result = $stmt->get_result();
```

### 6.2 Disable Directory Indexing & Secure the Web Root

- Permanently remove raw database dumps (`.sql` files) from any web-facing environment.
- Disable directory browsing via server config or `.htaccess`:

```
Options -Indexes
```

### 6.3 Uniform Error Handling

Replace field-specific errors ("Username not found" / "Incorrect password")
with a single generic message for both failure cases:

> "Invalid username or password."

---


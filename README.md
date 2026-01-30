# SQL Injection in PHPGurukul Hospital Management System V4.0

## Vulnerability Description
**Time-based Blind SQL Injection** vulnerability found in **PHPGurukul Hospital Management System (HMS) V4.0**. 

The vulnerability exists in the `/hospital/hms/appointment-history.php` file. The `id` parameter is vulnerable to SQL injection when the request is accompanied by a specific `cancel` parameter (`&cancel=1`). The application fails to properly sanitize the `id` input before using it in a SQL query within the cancellation logic.

**Unique Discovery**: Unlike previous vulnerabilities reported in this system (e.g., CVE-2020-22169), this injection point is protected by a logical check `if(isset($_GET['cancel']))`. Attackers must explicitly include the `cancel` parameter to bypass this check and trigger the vulnerable SQL query. This specific logic bypass has not been effectively addressed in the latest version.

## Technical Details
- **Product**: Hospital Management System (HMS)
- **Vendor**: PHPGurukul
- **Version**: V4.0 (Latest)
- **Vulnerability Type**: SQL Injection (Time-based Blind)
- **Affected File**: `/hospital/hms/appointment-history.php`
- **Affected Parameter**: `id` (Requires `&cancel=1` to trigger)
- **Discovery Date**: 2026-01-30
- **Author**: yan1451

## Proof of Concept (PoC)

### 1. Source Code Analysis
The vulnerability is located in the logic that handles appointment cancellation. The code directly concatenates the `id` parameter into the SQL statement without validation, but only executes when `cancel` is set.

```php
// Vulnerable Code Snippet in appointment-history.php
if(isset($_GET['cancel']))
{
    mysqli_query($con,"update appointment set userStatus='0' where id = '".$_GET['id']."'");
    // ...
}
```

### 2. Exploit Payload
To verify the vulnerability, use the following payload to trigger a 5-second delay.

**Request URL:**
```
http://localhost/Hospital-Management-System-PHP/hospital/hms/appointment-history.php?id=1' AND (SELECT 1 FROM (SELECT(SLEEP(5)))a)--+&cancel=1
```

**Note:** The `&cancel=1` parameter is mandatory. Without it, the injection payload will be ignored by the server.

### 3. Reproduction Steps
1. Login to the application as a normal user (Patient).
2. Navigate to "Appointment History" (`/hospital/hms/appointment-history.php`).
3. Append the payload above to the URL in the browser address bar (or use Burp Suite).
4. Observe the server response time.
5. **Result**: The server waits for approximately 5 seconds before responding, confirming the Time-based Blind SQL Injection.

## Evidence

### Vulnerable Code Logic
<img width="2535" height="1510" alt="图片1" src="https://github.com/user-attachments/assets/db839346-5a85-4346-8cc6-9520b4476d21" />


### Execution Proof (5 Second Delay)
<img width="2545" height="1526" alt="图片2" src="https://github.com/user-attachments/assets/e6756c87-6284-4bf7-adb9-fe4ccd4759e1" />

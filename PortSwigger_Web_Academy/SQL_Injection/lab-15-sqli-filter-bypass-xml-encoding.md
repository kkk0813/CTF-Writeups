# Lab: SQL injection with filter bypass via XML encoding

**Platform:** PortSwigger Web Security Academy  
**Category:** SQL Injection  
**Difficulty:** Practitioner

## 🎯 Objective
The application contains a SQL injection vulnerability in its stock check feature, which accepts XML input. However, a Web Application Firewall (WAF) blocks obvious SQL injection payloads. The goal is to bypass the WAF using XML encoding to execute a `UNION` attack and retrieve the `administrator` credentials.

## 🕵️‍♂️ Analysis
This vulnerability relies on a **Parser Discrepancy** between the WAF and the backend XML parser. 
The WAF inspects the incoming HTTP request for malicious SQL keywords (like `UNION` or `SELECT`). If we encode those keywords using XML hex entities, the WAF will fail to recognize the signature and allow the request through. 
Once the request reaches the backend application, the standard XML parser will automatically decode the hex entities back into raw SQL *before* executing the query, successfully triggering the payload.

## 🚀 Payload & Execution

### Step 1: Crafting the Base Payload
Because the stock checker returns its data directly to the screen (in the "units" field), I can use a standard In-Band `UNION` attack. Since there is only one output field, I must concatenate the username and password.
* **Base Payload:** `1 UNION SELECT username || '~' || password FROM users`

### Step 2: WAF Evasion via XML Encoding
Submitting the base payload directly results in the WAF blocking the request. To bypass this, I intercepted the `POST /product/stock` request in Burp Suite Repeater. 

Using the **Hackvertor** extension, I highlighted the payload inside the `<storeId>` tags and applied the `hex_entities` encoding.
* **Encoded Payload:** `<storeId><@hex_entities>1 UNION SELECT username || '~' || password FROM users<@/hex_entities></storeId>`

*(Note: Hackvertor translates this into XML entities like `&#x55;&#x4e;&#x49;&#x4f;&#x4e;` before sending it over the network).*

### Step 3: Execution and Exfiltration
I submitted the encoded request. 
* **Result:** The WAF allowed the request, the backend decoded it, and the database executed the `UNION`. The server responded with the dumped credentials:
  ```text
  124 units
  administrator~bipzyfgai78nsxlga6n7
  carlos~zt95bwqs15exjbbozoxn
  wiener~43qg07n4slr7r44zd19k
  ```
Logging in with the extracted administrator credentials solved the lab.

## 📸 Proof of Concept
![SQLi_via_xml_encoding](./images/sqli_xml_encoding.png)

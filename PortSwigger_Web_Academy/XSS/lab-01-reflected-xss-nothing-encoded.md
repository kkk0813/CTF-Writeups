# Lab: Reflected XSS into HTML context with nothing encoded

**Platform:** PortSwigger Web Security Academy  
**Category:** Cross-Site Scripting (XSS)  
**Difficulty:** Apprentice

## Objective
The application contains a simple reflected XSS vulnerability in the search functionality. The goal is to execute a JavaScript `alert()` function.

## Analysis & Execution
During initial testing of the search bar, I observed that the application reflects user input directly into the HTML response without any sanitization or HTML-encoding. 

Because the input is placed in a standard HTML context outside of any existing tags or JavaScript strings, a basic `<script>` payload can be injected to achieve code execution.

* **Injection Point:** The `search` parameter.
* **Payload:** `<script>alert(1)</script>`

**Execution:**
1. I entered the payload into the search box.
2. The server reflected the payload verbatim in the HTML response.
3. The browser parsed the unencoded `<script>` tags and executed the JavaScript, triggering the alert box and solving the lab.

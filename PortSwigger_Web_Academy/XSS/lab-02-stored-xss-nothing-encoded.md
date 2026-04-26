# Lab: Stored XSS into HTML context with nothing encoded

**Platform:** PortSwigger Web Security Academy  
**Category:** Cross-Site Scripting (XSS)  
**Difficulty:** Apprentice

## Objective
The application contains a stored cross-site scripting vulnerability in the comment functionality. The goal is to submit a comment that executes a JavaScript `alert()` function whenever the blog post is viewed.

## Analysis & Execution
Unlike Reflected XSS, Stored XSS involves injecting a payload that the application saves to its database and later renders to other users. 

During testing of the blog comment section, I found that user input in the "Comment" field is stored and subsequently reflected directly onto the blog page without HTML encoding or input sanitization.

* **Injection Point:** The `comment` body parameter.
* **Payload:** `<script>alert(1)</script>`

**Execution:**
1. I navigated to a blog post and filled out the comment form, injecting the payload into the main comment box.
2. I submitted the form with dummy data for the name, email, and website fields.
3. Upon returning to the blog post, the application loaded the comments from the database.
4. The browser parsed my unencoded `<script>` tags as active HTML and executed the JavaScript, triggering the alert box and solving the lab.

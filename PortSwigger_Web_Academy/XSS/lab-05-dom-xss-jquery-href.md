# Lab: DOM XSS in jQuery anchor href attribute sink using location.search source

**Platform:** PortSwigger Web Security Academy  
**Category:** Cross-Site Scripting (XSS)  
**Difficulty:** Apprentice

## Objective
The application contains a DOM-based cross-site scripting vulnerability in the "Submit feedback" page. The goal is to manipulate a jQuery selector to inject a JavaScript payload into an anchor (`<a>`) tag's `href` attribute, executing `alert(document.cookie)`.

## Analysis
By reviewing the page source of the feedback page, I identified the following vulnerable jQuery script:
```javascript
$(function() {
    $('#backLink').attr("href", (new URLSearchParams(window.location.search)).get('returnPath'));
});
```
* **Source:** `window.location.search` (Specifically, the `returnPath` parameter).
* **Sink:** jQuery's `.attr("href", ...)` function, which assigns the source data to the `href` attribute of the "Back" link.

Because the input is placed directly into an `href` attribute rather than the page body, standard HTML tags (like `<script>`) will not execute. Instead, we must utilize the `javascript:` pseudo-protocol to execute code when the anchor link is clicked.

## Payload & Execution

* **Injection Point:** The `returnPath` URL parameter.
* **Payload:** `javascript:alert(document.cookie)`

### Execution Steps:
1. I navigated to the "Submit feedback" page.
2. I modified the URL, changing the `returnPath` parameter to contain the payload:
   `...?returnPath=javascript:alert(document.cookie)`
3. The page loaded, and the jQuery script dynamically updated the "Back" link in the DOM to:
   `<a id="backLink" href="javascript:alert(document.cookie)">Back</a>`
4. To trigger the sink, I clicked the "Back" link.
5. The browser interpreted the `javascript:` pseudo-protocol and executed the payload, popping an alert box containing the session cookie and solving the lab.

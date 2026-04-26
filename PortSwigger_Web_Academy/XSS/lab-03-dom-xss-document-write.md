# Lab: DOM XSS in document.write sink using source location.search

**Platform:** PortSwigger Web Security Academy  
**Category:** Cross-Site Scripting (XSS)  
**Difficulty:** Apprentice

## Objective
The application contains a DOM-based cross-site scripting vulnerability in its search query tracking functionality. The goal is to perform an XSS attack that calls the `alert()` function by manipulating the DOM on the client side.

## Analysis
Unlike Server-Side XSS where the payload is embedded in the HTTP response, DOM-based XSS occurs entirely within the victim's browser. It requires two components:
1. **A Source:** A JavaScript property that accepts user-controllable data (in this case, `window.location.search`, which reads the URL query parameters).
2. **A Sink:** A JavaScript function that insecurely executes or renders that data (in this case, `document.write()`).

By inspecting the page source after submitting a random search term, I identified the vulnerable JavaScript handling the tracking image:
```javascript
function trackSearch(query) {
    document.write('<img src="/resources/images/tracker.gif?searchTerms='+query+'">');
}
var query = (new URLSearchParams(window.location.search)).get('search');
if(query) {
    trackSearch(query);
}
```

Because the `query` variable is concatenated directly into an HTML string and written to the DOM without sanitization, we can break out of the `src` attribute context and inject our own HTML tags.

## Payload & Execution
- **Injection Point**: The `search` URL parameter (Source).
- **Payload**: `"><svg onload=alert(1)>`

### Execution Steps:
  1. I entered a test string (`test`) into the search box to observe how the application handles the input.
  2. Using the browser's Developer Tools (Inspect Element), I saw the input was placed inside the `src` attribute of an `<img>` tag:  
     `<img src="/resources/images/tracker.gif?searchTerms=test">`
  3. To achieve execution, I crafted a payload to escape the attribute and tag context:
       - `">` : Closes the src attribute and the `<img>` tag.
       - `<svg onload=alert(1)>` : Introduces a new, valid HTML element that automatically executes JavaScript when it loads.
  4. Submitting this payload resulted in the following HTML being written to the DOM by `document.write`:
     `<img src="/resources/images/tracker.gif?searchTerms="><svg onload=alert(1)>">`
  5. The browser rendered the SVG image, immediately triggering the `alert(1)` popup and solving the lab.

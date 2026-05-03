# Lab: DOM XSS in innerHTML sink using source location.search

**Platform:** PortSwigger Web Security Academy  
**Category:** Cross-Site Scripting (XSS)  
**Difficulty:** Apprentice

## Objective
The application contains a DOM-based cross-site scripting vulnerability. The goal is to perform an XSS attack that calls the `alert()` function by exploiting the insecure use of the `innerHTML` property.

##  Analysis
By inspecting the page source, I located the JavaScript responsible for handling the search queries:
```javascript
function doSearchQuery(query) {
    document.getElementById('searchMessage').innerHTML = query;
}
var query = (new URLSearchParams(window.location.search)).get('search');
if(query) {
    doSearchQuery(query);
}
```
* **Source:** `window.location.search` (The URL query parameters).
* **Sink:** `innerHTML` (Modifies the DOM tree dynamically).

Because the input is passed to `innerHTML`, standard `<script>` tags will not execute due to HTML5 security specifications. To bypass this restriction, the payload must utilize an HTML element that automatically fires a JavaScript event handler upon rendering.

## Payload & Execution

* **Injection Point:** The `search` URL parameter.
* **Payload:** `<img src=1 onerror=alert(1)>`

### Execution Steps:
1. I observed that the application places the search input cleanly between `<span>` tags, meaning no attribute breakout (like `">`) was required.
2. I submitted the payload into the search bar.
3. The application's JavaScript passed the payload to the `innerHTML` sink, rendering the `<img>` tag on the page.
4. The browser attempted to fetch the image source `1`. 
5. When the fetch failed, the `onerror` event triggered automatically, executing the `alert(1)` JavaScript and solving the lab.

## Proof of Concept
![DOM XSS innerHTML](./images/xss-dom-innerhtml.png)
```markdown
| **DOM XSS (innerHTML)** | PortSwigger | XSS | DOM-Based, Event Handlers (`onerror`), HTML5 `<script>` Restriction | [View](./PortSwigger_Web_Academy/XSS/lab-04-dom-xss-innerhtml.md) |
```

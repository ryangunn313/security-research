# Lab — DOM XSS in `document.write` sink using source `location.search` — 2025-09-30
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/dom-based/dom-write-using-location-search  
**Section:** XSS / DOM XSS  
**Difficulty:** Easy  
**Time spent:** ~20 minutes

---

## 🎯 Objective
The lab tests whether user-controlled data from `window.location.search` is written into the DOM using `document.write()` without proper sanitization/encoding, allowing a DOM-based XSS attack.

---

## 🛠️ Environment & Tools
- Platform: PortSwigger Web Security Academy (PortSwigger labs)  
- Browser: Chrome (developer screenshots)  
- Proxy/Tools: Burp Suite (HTTP history + Repeater), Chrome DevTools (Elements/Console)  
- Notes: Proxy (Burp) running and intercepting; JavaScript enabled (required to trigger DOM behavior).

---

## 🔍 Steps Taken (concise)
1. **Recon:** Visited the blog search page and performed a search for `ryangunn313` to confirm `?search=...` appears in the URL and that the page shows "0 search results for 'ryangunn313'". (See screenshot `01_search_page.png`.)  
2. **Inspect:** Loaded the page with Burp capturing the request to `/` with `?search=ryangunn313`. Examined the page source/JS and found a `trackSearch(query)` function that calls `document.write('<img src="/resources/images/tracker.gif?searchTerms='+query+'">');` — i.e., it uses `location.search` → `URLSearchParams(...).get('search')` → `document.write(...)`. (See `03_burp_response_source.png`.)  
3. **Payloads tried:** Tested simple characters like quotes and attribute-like values to see where injection is possible: `"+bgcolor="#ff0000`, then `"+onload="javascript:alert(1)`.  
4. **Bypass/technique:** The sink is `document.write()` injecting into HTML markup. Closing the existing attribute/element context and injecting an `onload` handler on an `img` tag worked. The payload was URL-encoded in the `search` query string to reach the `document.write()` content.  
5. **Final payload & injection point (sanitized):**  
   - Injection point: `search` query parameter (read from `location.search`)  
   - Final (demonstration) payload (sanitized in this report):  
     ```
     " onload="javascript:alert('REDACTED')
     ```  
   - URL encoded version used in the lab (example):  
     ```
     /?search="+onload%3D"javascript%3Aalert%28'ryangunn313'%29
     ```  
   - This causes `document.write()` to output an `<img ...>` tag with the injected `onload` attribute, which executes the alert when the image is processed.

---

## ✅ Result
A JavaScript `alert()` executed in the page context (DOM XSS) after the crafted `search` parameter was written into the DOM by `document.write`. Evidence: browser alert dialog and DevTools showing the injected markup. (See screenshots below.)

---

## 🛡️ Mitigation Notes (for devs)
- **Avoid `document.write` with user data.** Do not build HTML by concatenating unsanitized user input.  
- **Context-aware encoding:** If you must insert data into HTML, perform proper encoding for the destination context (HTML attribute, HTML text, script, CSS, URL).  
- **Prefer safe DOM APIs:** Use DOM methods that set text (e.g., `textContent`) rather than `innerHTML`/`document.write` when adding user-controlled strings.  
- **CSP:** Deploy a robust Content Security Policy (CSP) that blocks inline scripts and disallows `unsafe-inline` where possible.  
- **Input validation:** Validate/normalize search terms (but keep in mind validation is not a substitute for output encoding).  
- **httpOnly cookies / sameSite:** For other attack vectors, ensure sensitive cookies have `HttpOnly` flag and `SameSite` attributes (not directly preventing DOM XSS, but reduces impact).

---

## 📚 Lessons Learned / Followups
- DOM sources like `location.search` are dangerous when combined with sink functions such as `document.write` or `innerHTML`.  
- Always verify the *context* (is the user string placed into HTML text or an attribute?) before deciding encoding strategy.  
- Practice creating safe server/client side search features using `textContent` or by building elements with `createElement`/`appendChild`.

---

## 📂 Artifacts (screenshots / evidence)
Below are the screenshots captured during the exercise. Each image is shown inline with a short description.

### 01_search_page.png  
*Initial blog search page with `ryangunn313` in the search box (UI).*  
[![01_search_page.png](evidence/01_search_page.png)](evidence/01_search_page.png)

### 02_burp_http_history.png  
*Burp Proxy → HTTP history entry for the GET request with `?search=ryangunn313`.*  
[![02_burp_http_history.png](evidence/02_burp_http_history.png)](evidence/02_burp_http_history.png)

### 03_burp_response_source.png  
*Burp response panel showing HTML/JS source where `trackSearch()` and `document.write()` appear.*  
[![03_burp_response_source.png](evidence/03_burp_response_source.png)](evidence/03_burp_response_source.png)

### 04_repeater_request.png  
*Repeater request logged for the `?search=...` request (request raw view).*  
[![04_repeater_request.png](evidence/04_repeater_request.png)](evidence/04_repeater_request.png)

### 05_repeater_response_source.png  
*Repeater response showing HTML snippet (script / `document.write` call).*  
[![05_repeater_response_source.png](evidence/05_repeater_response_source.png)](evidence/05_repeater_response_source.png)

### 06_devtools_dom_before_payload.png  
*Chrome DevTools Elements view before injection (shows `img src="/resources/images/tracker.gif?searchTerms=ryangunn313"`).*  
[![06_devtools_dom_before_payload.png](evidence/06_devtools_dom_before_payload.png)](evidence/06_devtools_dom_before_payload.png)

### 07_injected_test_input_devtools.png  
*DevTools showing the search input containing test injection string (raw).*  
[![07_injected_test_input_devtools.png](evidence/07_injected_test_input_devtools.png)](evidence/07_injected_test_input_devtools.png)

### 08_encoded_payload_in_url.png  
*Browser address bar showing the URL-encoded attack payload in `search` query.*  
[![08_encoded_payload_in_url.png](evidence/08_encoded_payload_in_url.png)](evidence/08_encoded_payload_in_url.png)

### 09_devtools_injected_input.png  
*DevTools showing the injected/created HTML after `document.write` (injected attribute present).*  
[![09_devtools_injected_input.png](evidence/09_devtools_injected_input.png)](evidence/09_devtools_injected_input.png)

### 10_injected_input_inside_search_box.png  
*Search box showing the visible (unencoded) input that includes the injected fragment.*  
[![10_injected_input_inside_search_box.png](evidence/10_injected_input_inside_search_box.png)](evidence/10_injected_input_inside_search_box.png)

### 11_browser_alert.png  
*Browser alert dialog triggered by the XSS (proof of execution).*  
[![11_browser_alert.png](evidence/11_browser_alert.png)](evidence/11_browser_alert.png)

### 12_solved_page.png  
*Final solved lab page showing success banner, injected payload visible in page and DevTools.*  
[![12_solved_page.png](evidence/12_solved_page.png)](evidence/12_solved_page.png)




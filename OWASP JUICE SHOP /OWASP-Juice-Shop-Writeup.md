Written by Sibongamandla Mnyandu

# OWASP Juice Shop Writeup

## Introduction

OWASP Juice Shop is a deliberately broken web application maintained by the OWASP Foundation, purpose-built to surface real attack patterns against a sandboxed target. It gamifies the **OWASP Top 10**, driving each intentional flaw through a hidden challenge system that releases a flag on exploitation. Nothing in the app is accidental.

I worked through the [OWASP Juice Shop room on TryHackMe](https://tryhackme.com/room/owaspjuiceshop), tracing a path across five vulnerability classes: Injection, Broken Authentication, Sensitive Data Exposure, Broken Access Control, and Cross-Site Scripting (XSS). These categories interlink; a weak authentication boundary, for instance, can amplify an injection vector sitting one layer beneath it. For each class, I document the underlying flaw, the exact payload or technique I used to exploit it, and the observed outcome, anchored by screenshots captured at each step.

## CTF Background

- **Target:** OWASP Juice Shop (Express ^4.21.0 single-page Angular application)
- **Machine IP:** `10.81.139.174`
- **Attacker box:** Kali Linux
- **Tooling:** Firefox, Burp Suite Community Edition, browser DevTools, SecLists (`best1050.txt`)
- **Goal:** Solve the challenges tied to each OWASP Top 10 category and reveal the hidden Score Board.

Before touching any exploit, a light reconnaissance pass through the app's UI and REST endpoints surfaced three details the room asks about directly:

- **`admin@juice-sh.op`**: the administrator's email, left unredacted across the application.
- **`q`**: the query parameter driving the product search endpoint at `/rest/products/search?q=`.
- A user named **Jim** whose product reviews quote *Star Trek* dialogue, an ostensibly quirky detail that becomes an authentication foothold later.

---

## Injection

SQL Injection happens when unsanitised input gets spliced directly into a database query, letting an attacker rewrite the query's intent rather than just supply its data.

### Login as Admin

I intercepted the login request in Burp Suite and observed the JSON body containing the `email` and `password` fields being sent to `/rest/user/login`.

![Login form](Screenshots/Screenshot%202026-08-12%20at%2018.25.56.png)

![Intercepted login request in Burp](Screenshots/Screenshot%202026-08-12%20at%2018.26.37.png)

The email field feeds directly into the SQL query with no escaping and no parameterisation. That single oversight means I can rewrite the query's logic entirely. The payload:

```
' or 1=1--
```

The opening quote closes the email string. `or 1=1` forces the `WHERE` clause to resolve true for every row. The double dash discards everything after it. The database hands back row one (the admin) and the app issues an authentication token for `admin@juice-sh.op` (`id: 1`). Challenge solved.

![Admin auth token in response](Screenshots/Screenshot%202026-08-12%20at%2018.34.49.png)

![whoami confirms admin@juice-sh.op](Screenshots/Screenshot%202026-08-12%20at%2018.36.53.png)

![Login Admin challenge solved](Screenshots/Screenshot%202026-08-12%20at%2018.32.45.png)

### Login as Bender

Targeting a *specific* account requires a tighter payload. Rather than collapsing the `WHERE` clause for every row, I supplied Bender's known email and used the comment sequence to sever the password check entirely:

```
bender@juice-sh.op'--
```

![Bender login attempt in the form](Screenshots/Screenshot%202026-08-12%20at%2018.38.49.png)

![Intercepting Bender's login request](Screenshots/Screenshot%202026-08-12%20at%2018.39.15.png)

![Injecting bender@juice-sh.op'-- into the request](Screenshots/Screenshot%202026-08-12%20at%2018.40.28.png)

The quote terminates the email string; the double dash discards the rest. The server authenticates me as Bender with no password required, solving the **Login Bender** challenge.

![Login Bender challenge solved](Screenshots/Screenshot%202026-08-12%20at%2018.41.11.png)

---

## Broken Authentication

Broken Authentication surfaces wherever an application's identity-confirmation layer cuts corners. A weak password policy here, a guessable recovery answer there. Each gap becomes a direct path to account takeover.

### Brute-forcing the Admin password

The **Password Strength** challenge sidesteps injection entirely. It requires the *actual* admin credentials, proving that a weak password negates every other control. I forwarded the login request to **Burp Intruder**, planted the payload position on the `password` field for `admin@juice-sh.op`, and loaded `best1050.txt` from SecLists.

![Installing SecLists in Kali](Screenshots/Screenshot%202026-08-12%20at%2019.07.35.png)

![Configuring the Intruder payload position](Screenshots/Screenshot%202026-08-12%20at%2019.21.50.png)

![Intruder running through the wordlist](Screenshots/Screenshot%202026-08-12%20at%2019.20.04.png)

One request breaks from the crowd: a **302** redirect where every other attempt returns a **401**. The winning payload is embarrassingly simple:

```
admin123
```

![The admin123 payload returns a 302](Screenshots/Screenshot%202026-08-12%20at%2019.32.27.png)

That's the admin's real password. Challenge solved.

![Password Strength challenge solved](Screenshots/Screenshot%202026-08-12%20at%2019.33.08.png)

### Resetting Jim's Password

`jim@juice-sh.op`'s password reset asks: *"Your eldest sibling's middle name?"* The answer lives in publicly available franchise lore. That's the flaw. Security questions only work when the answer isn't searchable.

![Jim's security question](Screenshots/Screenshot%202026-08-12%20at%2019.41.04.png)

The earlier recon flagged Jim's Star Trek references. Jim maps to **James T. Kirk**; Kirk's older brother is **George Samuel "Sam" Kirk**. The security answer:

```
Samuel
```

![Researching Jim's Star Trek sibling](Screenshots/Screenshot%202026-08-12%20at%2019.44.44.png)

One search, one answer, one account. The **Reset Jim's Password** challenge falls.

![Reset Jim's Password challenge solved](Screenshots/Screenshot%202026-08-12%20at%2019.47.04.png)

---

## Sensitive Data Exposure

Sensitive Data Exposure is what happens when internal assets (backups, credentials, configuration dumps) end up reachable by anyone who thinks to check a predictable path.

### Accessing the confidential document

Navigating to `/ftp` surfaces a wide-open directory listing. The server isn't just leaking a file; it's handing over an unfiltered index of backups and internal assets. `acquisitions.md` sits there, ungated and directly servable.

![Directory listing at /ftp](Screenshots/Screenshot%202026-08-12%20at%2020.46.50.png)

![Inspecting the exposed /ftp files](Screenshots/Screenshot%202026-08-12%20at%2021.22.29.png)

Opening it resolves the **Confidential Document** challenge.

![Confidential Document challenge solved](Screenshots/Screenshot%202026-08-12%20at%2021.26.39.png)

### Poison Null Byte: retrieving the backup file

Not every file in `/ftp` is that accessible. `package.json.bak` triggers a 403:

```
403 Error: Only .md and .pdf files are allowed!
```

![403: only .md and .pdf allowed](Screenshots/Screenshot%202026-08-12%20at%2021.32.56.png)

The filter checks the file extension at the application layer. That's the gap. A **Poison Null Byte** exploits the disconnect between how the app and the file system each read the same string. Appending `%2500.md` to the filename feeds the filter an acceptable extension, but the OS truncates the string at the null byte and retrieves the real file:

```
/ftp/package.json.bak%2500.md
```

![Poison Null Byte in the URL bar](Screenshots/Screenshot%202026-08-12%20at%2021.33.56.png)

Triggering the 403 first, then landing the null byte bypass, clears three challenges in one move: **Error Handling**, **Forgotten Developer Backup**, and **Poison Null Byte**.

![Error Handling, Forgotten Developer Backup and Poison Null Byte solved](Screenshots/Screenshot%202026-08-12%20at%2021.37.20.png)

---

## Broken Access Control

Broken Access Control is the gap between what a user is *permitted* to do and what the application actually *prevents* them from doing. When those two things don't align, attackers walk straight into restricted territory: admin panels, other users' data, records they have no business touching. **IDOR** (Insecure Direct Object Reference) is a textbook example of that misalignment.

### Finding and accessing the Admin section

The admin panel carries no link in the UI. That's security by obscurity, not enforcement. Opening the bundled `main.js` in DevTools and searching for `admin` surfaces the client-side route definition. The route is sitting there in plain text.

![Searching main.js for the admin route](Screenshots/Screenshot%202026-08-12%20at%2021.54.10.png)

![The administration route in main.js](Screenshots/Screenshot%202026-08-12%20at%2021.56.35.png)

Navigating directly to `/#/administration` while authenticated as admin loads the panel and clears the **Admin Section** challenge.

![Admin Section challenge solved](Screenshots/Screenshot%202026-08-12%20at%2021.58.40.png)

### Viewing another user's basket (IDOR)

`/rest/basket/{id}` fetches any basket by its sequential integer ID. The server issues no ownership check; it returns whatever ID you request. Changing `1` to `2` hands me another user's basket with no authentication prompt.

![My basket, fetched by ID](Screenshots/Screenshot%202026-08-12%20at%2022.16.13.png)

That's IDOR in its purest form. **View Basket** challenge cleared.

![View Basket challenge solved](Screenshots/Screenshot%202026-08-12%20at%2022.17.12.png)

### Removing all five-star reviews

The administration panel exposes every piece of customer feedback for deletion. Removing each 5-star review, an action the application should gate behind server-side role checks, clears the **Five-Star Feedback** challenge.

![Five-Star Feedback challenge solved](Screenshots/Screenshot%202026-08-12%20at%2022.18.39.png)

---

## Cross-Site Scripting (XSS)

XSS hands an attacker execution inside another user's browser at the same origin, same session, same trust level as the legitimate user. Juice Shop surfaces all three variants: **DOM**, **Persistent (stored)**, and **Reflected**, each driven by the same payload:

```html
<iframe src="javascript:alert(`xss`)">
```

### DOM XSS

The search bar writes the `q` parameter directly into the DOM with no encoding and no sanitisation. Submitting the iframe payload there executes the script in-browser and fires the `xss` alert.

![DOM XSS alert firing from the search bar](Screenshots/Screenshot%202026-08-12%20at%2022.23.47.png)

**DOM XSS** challenge cleared.

![DOM XSS challenge solved](Screenshots/Screenshot%202026-08-12%20at%2022.24.06.png)

### Persistent / HTTP-Header XSS

Under **Account → Privacy & Security → Last Login IP**, the app renders whatever value the `True-Client-IP` header carried on the last login, stored verbatim and displayed unsanitised. That makes it a stored XSS sink buried inside an HTTP header rather than a form field.

![Locating the Last Login IP feature](Screenshots/Screenshot%202026-08-12%20at%2022.28.45.png)

Using Burp, I injected the payload directly into `True-Client-IP` on the request that persists the login IP:

```
True-Client-IP: <iframe src="javascript:alert(`xss`)">
```

![Injecting the payload into the True-Client-IP header](Screenshots/Screenshot%202026-08-12%20at%2022.36.14.png)

Next page load: the stored payload executes. **HTTP-Header XSS** challenge cleared.

![HTTP-Header XSS challenge solved](Screenshots/Screenshot%202026-08-12%20at%2022.37.06.png)

![Stored payload executing on the Last Login IP page](Screenshots/Screenshot%202026-08-12%20at%2022.38.02.png)

### Reflected XSS

The order-tracking page takes the `id` parameter from the URL and renders it into the response without escaping it. Placing any order surfaces a tracking ID to confirm the parameter is live.

![Order tracking result reflecting the id parameter](Screenshots/Screenshot%202026-08-12%20at%2022.39.57.png)

Swapping that ID for the iframe payload is enough; the page loads and the script executes:

```
/#/track-result?id=<iframe src="javascript:alert(`xss`)">
```

![Injecting the payload into the id parameter](Screenshots/Screenshot%202026-08-12%20at%2022.40.31.png)

![Reflected XSS alert firing](Screenshots/Screenshot%202026-08-12%20at%2022.41.03.png)

No persistence required. No additional user interaction beyond the page load. **Reflected XSS** challenge cleared.

![Reflected XSS challenge solved](Screenshots/Screenshot%202026-08-12%20at%2022.41.12.png)

---

## The Score Board

The Score Board at `/#/score-board` is itself a challenge; finding it is how the room opens. By the end of this run it showed a full sweep across Injection, Broken Authentication, Sensitive Data Exposure, Broken Access Control, and XSS.

![Score Board with solved challenges](Screenshots/Screenshot%202026-08-12%20at%2022.44.18.png)

---

## Takeaways

- **User input is adversarial until proven otherwise.** Every exploit here began the moment untrusted data touched a SQL query, an HTML sink, or a file path with no validation, no escaping, and no authorisation check in sight.
- **Parameterise your queries.** `' or 1=1--` becomes inert the moment the database treats input as data, not syntax.
- **Authentication is only as durable as its weakest recovery path.** `admin123` and `Samuel` both bypassed what looked like a working login because the login itself was never the actual weak point.
- **Don't leave backups in web-accessible directories.** An open `/ftp` with a bypassable extension filter is a configuration dump waiting to be collected.
- **Access control lives on the server, not in the client bundle.** A route buried in `main.js` and a sequential integer ID in `/rest/basket/{id}` are not access controls. They're conveniences for the attacker.
- **Encode output contextually.** DOM, stored, and reflected XSS share one fix: escape data before it reaches the browser and treat every header, including `True-Client-IP`, as untrusted.

## References

- OWASP Juice Shop: https://owasp.org/www-project-juice-shop/
- TryHackMe, OWASP Juice Shop: https://tryhackme.com/room/owaspjuiceshop
- OWASP Top 10: https://owasp.org/www-project-top-ten/
- Pwning OWASP Juice Shop (official companion guide): https://pwning.owasp-juice.shop/

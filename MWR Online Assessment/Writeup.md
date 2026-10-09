Written by Sibongamandla Mnyandu

# Imgurr Web Application — Security Assessment Report

**Prepared by:** Nhlanhla Mnyandu — associate consultant
**Engagement:** MWR CyberSec intern onboarding — web application assessment
**Testing performed:** 8–9 October 2026

---

## Engagement summary

This is my write-up of the assessment I ran against the Imgurr mock application as part of the MWR CyberSec intern onboarding, working in an associate-consultant capacity. I activated my own instance of the room first, because the brief is clear that activation makes the findings and flags unique to each tester — so throughout this report the host is just `TARGET`; the actual address changed on me more than once as the VM re-provisioned, which is worth stating up front so nobody reads the shifting IPs in my raw captures as sloppiness.

The app is a single-page front end sitting on a REST API. Almost everything interesting happens below the UI: the confirmation flags (the `MWR{...}` strings) only ever come back in JSON response bodies, never rendered on the page, so I worked mostly at the raw-HTTP level through an intercepting proxy. I've written up both the vulnerabilities that returned a flag and the ones that didn't, since the brief says thoroughness is scored and a real finding doesn't stop being real just because no flag happens to sit behind it.

On where this ended up: the room is said to hold up to 14 flags, and the instruction was to get as many as I reasonably could, not all of them. So I chased the cheap, high-probability stuff first and stopped when the next flags clearly cost more effort than the time I had left. I confirmed seven High-severity findings and captured three flags. A handful of other vectors I identified and scoped but didn't get to — rather than quietly drop those, I've listed them near the end with the reasoning for each, so the picture is honest about what's done and what isn't.

---

## Risk rating definitions

These are the MWR definitions from the rules of engagement, not my own scale.

| Rating | Definition |
|---|---|
| **High** | Potential for an attacker to control, alter, or delete electronic assets. Includes unauthorised access, data capture, and defacement. |
| **Medium** | Combined with other factors, there is potential for asset compromise. Exploitation may be conditional or configuration-dependent. |
| **Low** | Likelihood or impact of exploitation is extremely low. Examples: weak ciphers, outdated protocols, programmatic CAPTCHA bypass. |
| **Informational** | Cannot be exploited directly but is not aligned with best practice. Examples: information disclosure, version leakage. |

---

## Findings summary

| ID | Finding | Rating | Flag | Primary endpoint |
|---|---|---|---|---|
| IMG-01 | Stored XSS in reviews | High | Captured | `POST /api/shop/images/{id}/review` |
| IMG-02 | Path traversal / arbitrary file read | High | None returned | `GET /api/files/images/{id}/download` |
| IMG-03 | JWT weak signing secret → admin-claim forgery | High | Captured | `GET /api/auth/me` |
| IMG-04 | Voucher & economy business logic | Not finished | To test | voucher + upvote endpoints |
| IMG-05 | Sell workflow — stored XSS angle | Not finished | To test | `POST /api/uploads/listings` |
| IMG-06 | SQL injection in image search | High | Captured | `GET /api/shop/images?q=` |
| IMG-07 | Default administrator credentials | High | None returned | `POST /api/admin/login` |
| IMG-08 | Privilege escalation via the admin API | High | None returned | `POST /api/admin/users/{id}/promote` |
| IMG-09 | Unrestricted file upload (non-image types) | High | None returned | `POST /api/uploads/listings` |

Seven of these are confirmed High. Three flags captured:

- `MWR{227ad8df8a3c15d20943d3472ca43d6b}` — stored XSS (IMG-01)
- `MWR{730067fa8a37de7a64dce4fa0f859189}` — SQL injection (IMG-06)
- `MWR{a698e61eccbd5875d815eb49334414d2}` — JWT weak secret (IMG-03)

---

## IMG-01 — Stored XSS in reviews

| | |
|---|---|
| **Risk rating** | High |
| **Type** | Stored XSS (CWE-79) |
| **Flag** | `MWR{227ad8df8a3c15d20943d3472ca43d6b}` (captured) |
| **Affected endpoints** | `POST /api/shop/images/{id}/review`, and the comment endpoint alongside it |

The image-detail page renders review and comment bodies into the DOM through Vue's `innerHTML` binding rather than normal text interpolation, and the review `comment` field is stored and reflected with no output encoding at all. So whatever you put in there runs in the browser of anyone who later opens that image.

What makes this one more than a generic XSS is the inconsistency right next to it. The comment endpoint *does* encode its output — I sent a `<script>` payload there and it came back as `&lt;script&gt;...`, properly neutered. The review endpoint, which is a sibling of it, does not. So this isn't a blanket "they forgot to escape" problem; it's escaping applied in one place and missed in another, which is the kind of gap that survives a casual code review.

The review form on the image-detail page is the entry point — a star rating and a free-text "What did you think?" box. When I submitted a review carrying an `<img src=x onerror=...>` payload, the stored body rendered straight back into the review list: the browser tries to load the bogus `src`, fails, and the broken-image placeholder you see below is the `onerror` handler firing in-session.

![Submitting the review payload on /images/3 — it renders back as a broken image, i.e. the markup was injected into the DOM, not encoded](Screenshot/Screenshot%202026-10-08%20at%2023.17.20.png)

**Proof of concept.**

```http
POST /api/shop/images/2/review HTTP/1.1
Host: TARGET
Authorization: Bearer <valid token>
Content-Type: application/json

{"rating":5,"comment":"<img src=x onerror=alert('MWR')>"}
```

Came back `201 CREATED` with the flag sitting in the body:

```json
{"flags":{"xss":"MWR{227ad8df8a3c15d20943d3472ca43d6b}"},"review":{"comment":"<img src=x onerror=alert('MWR')>","rating":5,"image_id":2,...},"status":"review submitted"}
```

![Burp response: 201 CREATED with the XSS flag MWR{227ad8df8a3c15d20943d3472ca43d6b} in the JSON body](Screenshot/Screenshot%202026-10-08%20at%2023.18.26.png)

The payload is stored exactly as sent. When the review list renders, the `onerror` fires in the viewer's session.

**Impact.** Stored, so no social engineering needed beyond getting a victim to open the image — and that includes any admin who views it. From there it's the usual: session theft, actions as the victim, injected credential-harvesting forms, defacement.

**Remediation.**
- Stop binding user text with `v-html` / `innerHTML` on review, comment, title and description. Use text interpolation.
- Encode on output server-side, and do it consistently across every field — the whole problem here is that it's done per-endpoint.
- Add a CSP that forbids inline script, as a backstop for when something slips through again.

---

## IMG-02 — Path traversal / arbitrary file read

| | |
|---|---|
| **Risk rating** | High |
| **Type** | Path traversal / arbitrary file read (CWE-22, CWE-552) |
| **Flag** | None returned at this endpoint — documented as a non-flag finding, per the RoE |
| **Affected endpoint** | `GET /api/files/images/{id}/download?filename=<path>` |

The download route takes a `filename` parameter straight from the client and hands it to a file read with no sanitisation, no scoping, no canonicalisation. What tipped me off was a deprecated comment left in `ImageDetail-17W_SMUm.js` — it says the UI stopped sending `?filename=`, but that the server route still honours it. So it's a parameter the front end forgot about and the back end never removed.

Pointing it at an absolute path returns that file. I spent a little time understanding *how* the read happens, because the error behaviour was confusing at first — everything I couldn't read came back as the same "file not found". Reading `/proc/self/environ` explained it:

```
SUDO_COMMAND=/bin/cat -- /proc/self/environ
SUDO_USER=imgurr
USER=imgurr-sandbox
```

So the app runs as `imgurr` and shells out to `sudo -u imgurr-sandbox /bin/cat -- <filename>` for each download. Two things fall out of that: the `--` stops you injecting extra arguments into `cat`, and the `imgurr-sandbox` user is deliberately low-privilege, so anything it can't read returns the generic `{"error":"file not found"}` — identical to a genuinely missing file. That last detail matters, because it means a "not found" tells you nothing about whether the path was wrong or just unreadable.

**Proof of concept.**

```http
GET /api/files/images/4/download?filename=/etc/passwd HTTP/1.1
Host: TARGET
Authorization: Bearer <valid token>
```

`200 OK`, `Content-Disposition: attachment; filename="passwd"`, and the real file:

```
root:x:0:0:root:/root:/bin/bash
...
imgurr:x:996:996::/home/imgurr:/usr/sbin/nologin
imgurr-sandbox:x:995:998::/home/imgurr-sandbox:/usr/sbin/nologin
```

I confirmed it again against `/etc/nginx/nginx.conf`, the nginx site config, and `/proc/self/cmdline` — all returned their true contents.

**On the flag.** I did spend time hunting for a readable `flag` file before setting it aside. I walked the obvious candidates the sandbox user might reach — bare and relative names from the images base folder (`flag`, `flag.txt`, `../flag`, `../../flag`), the nginx config paths, and the usual web roots (`/var/www/flag`, `/var/www/html/flag`, `/srv/flag`, `/home/imgurr-sandbox/flag`). All of them returned the same generic `{"error":"file not found"}`, which — because missing and unreadable look identical here — tells you nothing. I also tried to let the server locate itself for me via `/proc/self/cwd/`, and that's where I hit the wall: each download spawns its *own* short-lived `sudo cat` subprocess, so `/proc/self` only ever describes that `cat` (whose cwd `sudo` resets to `/`), never the web app. As a source-locator it's a dead end. The right next move is to read the application source to get the exact path, which is bounded work I ran out of time for. The RoE also says the app can hold more vulnerabilities than there are flags and that non-flag findings still count, and the host-testing restriction means I'm not poking at SSH keys or shadow. The `/etc/passwd` and nginx reads evidence the bug completely; that's enough to stand on.

**Impact.** Any authenticated user can read files the sandbox account can reach — config, service-account layout, environment, possibly source or secrets. On its own it's an information-disclosure and recon problem; the bigger risk is what it feeds into.

**Remediation.**
- Drop the client `filename` entirely. Resolve downloads from an internal asset ID that maps to a known file.
- If you must take a name, `realpath` it and assert it stays inside the storage root before reading.
- Replace the `cat` subprocess with a native file read — there's no reason to involve a shell.
- Keep returning the same generic error for missing vs. unauthorised (it already does), but log the difference server-side.

---

## IMG-03 — JWT weak signing secret → admin-claim forgery

| | |
|---|---|
| **Risk rating** | High |
| **Type** | Broken authentication / privilege escalation (CWE-347, CWE-798, CWE-287) |
| **Flag** | `MWR{a698e61eccbd5875d815eb49334414d2}` (captured) |
| **Affected endpoints** | `GET /api/auth/me`, and anything that trusts the `is_admin` claim |

The user token is an HS256 JWT, and it carries `is_admin` right there in the payload where the client can read it. That's the thing that makes you look twice. The obvious first test is whether the server actually checks the signature at all.

I tried the `alg:none` route first — strip the signature, set the algorithm to none, flip `is_admin` to true. The server rejected it with a `401`. Good for them; that means algorithm-stripping is properly handled, and I've recorded it that way rather than pretending it was the way in.

The secret itself was the soft spot. I pulled a valid token and ran it through John against rockyou, and it fell over almost instantly — the signing key is the literal word `password`. The crack was under a second, which is really the whole severity argument: there's no attacker effort here to speak of.

```
john token.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=HMAC-SHA256
# 1 hash loaded, cracked basically on the first guess
# secret: password
```

![John the Ripper cracking the HMAC-SHA256 signing secret in under a second — the key is "password"](Screenshot/Screenshot%202026-10-09%20at%2011.12.57.png)

With the secret known, re-signing a tampered token is trivial:

```
python3 jwt_tool.py "$TOKEN" -S hs256 -p 'password' -I -pc is_admin -pv true
```

![jwt_tool re-signing the token with the recovered secret and flipping is_admin to true](Screenshot/Screenshot%202026-10-09%20at%2011.17.51.png)

And then just use it:

```http
GET /api/auth/me HTTP/1.1
Host: TARGET
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.<tampered-payload>.<valid-signature>
```

The server accepts the self-signed token on the main API without complaint:

![The forged token is accepted on the shop API — 200 OK in Repeater](Screenshot/Screenshot%202026-10-09%20at%2011.23.22.png)

`200 OK`, and the flag:

```json
{"flags":{"authz":"MWR{a698e61eccbd5875d815eb49334414d2}"},"user":{...,"is_admin":false,...}}
```

![GET /api/auth/me with the forged token returns flags.authz MWR{a698e61eccbd5875d815eb49334414d2}](Screenshot/Screenshot%202026-10-09%20at%2011.24.45.png)

One thing worth explaining, because it threw me for a second: the `user` block in that response still says `is_admin:false`. That's just the stored database row for my account being echoed back. The flag fires on the *token's* claim being accepted as genuine — the server trusted a value I signed myself. That's exactly the weakness, so the mismatch isn't a contradiction, it's the point.

A correction to my own early assumption while I'm here: I expected this forged token to also open the admin panel, and it doesn't. The admin side runs a completely separate, cookie-based auth (see IMG-07) and answers this token with `admin authentication required`. So this finding owns the main API; it does not reach `/api/admin/*`. Two independent auth systems, which I didn't realise until I actually tried.

**Impact.** Knowing the secret means forging a token for any user or privilege on the main API that the server will treat as real — full auth bypass. It also stacks with IMG-06, which handed over real usernames and confirmed the `is_admin` column is live.

**Remediation.**
- Replace the secret with a long random key kept out of source control. Treat the current one as burned and rotate it; invalidate issued tokens.
- Keep rejecting `alg:none` — that part's already right.
- Stop trusting a client-held `is_admin`. Decide privilege server-side, per request, from the authenticated subject.

---

## IMG-04 — Voucher and economy business logic (not finished)

| | |
|---|---|
| **Risk rating** | TBD |
| **Type** | Business logic flaw (CWE-840) |
| **Flag** | To test |
| **Affected endpoints** | voucher/coupon redemption, `POST /api/comments/{id}/upvote`, the purchase flow |

I started on this and ran out of runway, so I'm being straight that it's unfinished rather than dressing it up. The voucher feature is a "no account needed" flow that takes simulated card details and an `Amount (MWR-Bucks)`, and hands back a redeemable code.

![The voucher purchase form — amount is a client-supplied field alongside simulated card details](Screenshot/Screenshot%202026-10-08%20at%2017.16.31.png)

What I know so far: the purchase flow spends a `mwr_bucks` balance that starts at 100 and goes down per buy, so any way to influence that number is a target. When I drove the amount field server-side, the voucher I got back was worth **500 MWR-Bucks** — well above the balance a fresh account starts with — which is exactly the sort of client-trusts-the-amount gap that makes the whole economy worth a proper session.

![A purchased voucher worth 500 MWR-Bucks — the amount is attacker-influenced, which is the thread worth pulling on](Screenshot/Screenshot%202026-10-08%20at%2023.08.14.png)

The upvote handler also has a paid path — on a `402` it returns `can_purchase` and `cost_bucks`, and sending `{"purchase":true}` spends bucks and echoes a new balance back, which smells like it could be raced or fed a negative cost. And IMG-06 showed `coupons` and `coupon_redemptions` tables that are separate from vouchers — a whole feature I never touched.

My initial voucher pokes didn't obviously break anything flag-wise, but I wouldn't call the area tested. The interesting work (coupon race, negative values, the upvote economy) is in the "further testing" list.

---

## IMG-05 — Sell workflow, stored XSS angle (not finished)

| | |
|---|---|
| **Risk rating** | TBD |
| **Type** | Stored XSS via seller-controlled fields (CWE-79) |
| **Flag** | To test |
| **Affected endpoints** | `POST /api/uploads/listings`, and the image-detail rendering |

This one I'm fairly confident about but didn't get to prove. The image title and description render through the same `innerHTML` sink as IMG-01, and those are seller-controlled. The RoE hints that one vulnerability can drop flags at two endpoints, so a second `flags.xss` via a listing's title or description is a reasonable bet — it would belong under IMG-01 if it lands. I mapped the upload/listing flow (see IMG-09) but didn't run the XSS payload through it. Noted for next time.

---

## IMG-06 — SQL injection in image search

| | |
|---|---|
| **Risk rating** | High |
| **Type** | SQL injection (CWE-89) |
| **Flag** | `MWR{730067fa8a37de7a64dce4fa0f859189}` (from the `ctf_sqli_flags` table) |
| **Affected endpoint** | `GET /api/shop/images?q=<value>` |

The `q` search parameter gets concatenated straight into a SQLite `LIKE` clause. What first caught my eye was that searched results came back in a slightly different shape to the normal listing — which usually means a separate, hand-built query path rather than the parameterised one. A bare `%` in `q` returned every row, which confirmed the input was landing inside `LIKE '%...%'`. From there I confirmed it three independent ways — boolean-blind, time-based, and a UNION — and settled on a 7-column UNION as the clean exfil channel. Back end is SQLite on Ubuntu behind nginx 1.24.0.

```http
GET /api/shop/images?limit=20&offset=0&q=blue%' UNION ALL SELECT NULL,NULL,NULL,<marker>,NULL,NULL,NULL-- -
Host: TARGET
Authorization: Bearer <valid token>
```

The marker surfaced in the `description` field (the 4th column), so that's where I pulled data from. The flag wasn't in `users` — it lives in a dedicated table I found by reading `sqlite_master`:

```http
GET /api/shop/images?limit=20&offset=0&q=blue%' UNION ALL SELECT NULL,NULL,NULL,(SELECT flag FROM ctf_sqli_flags),NULL,NULL,NULL-- -
```

→ `MWR{730067fa8a37de7a64dce4fa0f859189}`.

**Impact.** This is the big one. The injection gave up the entire `users` table — 305 rows, with usernames, emails, scrypt password hashes, roles, and the `is_admin` flag. Several admin accounts were exposed this way, and that directly fed IMG-03 (it's how I knew `is_admin` was a live column with real admins behind it). Arbitrary table read means an attacker takes the whole dataset.

I've deliberately left the password hashes out of this report — I'm noting only that they exist and are scrypt. I didn't attempt to crack them (scrypt with those parameters isn't realistically crackable anyway, and it wasn't needed), and I used the database access only to prove the bug, not to pivot anywhere.

**Remediation.**
- Parameterise the `q` filter. Never concatenate into SQL.
- Escape or allowlist the `LIKE` wildcards (`%`, `_`) in the term.
- Least-privilege the DB account; don't return raw columns shaped by the injected query.
- Alert on `UNION`/quote/comment tokens in search params.

---

## IMG-07 — Default administrator credentials

| | |
|---|---|
| **Risk rating** | High |
| **Type** | Default / weak credentials (CWE-1392, CWE-521) |
| **Flag** | None returned directly |
| **Affected endpoint** | `POST /api/admin/login` |

This is almost embarrassing in how simple it turned out to be, and I nearly missed it by overcomplicating things. The admin interface is served from its own bundles — `AdminLogin`, `Admin`, `adminAuth` — code-split away from the main app, which is why none of my earlier greps over `index.js` found any admin routes. The first clue was grepping the main bundle for the hashed chunk names and pulling them down directly:

![Grepping index.js for the code-split admin chunks and curling them to disk](Screenshot/Screenshot%202026-10-09%20at%2012.22.57.png)

Running the downloaded chunks through a quick regex to pull out every API path they reference surfaced the whole hidden admin surface — `/api/admin`, `/admin/login`, `/sellers/pending`, `/users`:

![Extracting admin/user/seller routes from the downloaded chunks with a Python regex](Screenshot/Screenshot%202026-10-09%20at%2012.23.12.png)

Reading `adminAuth.js` and `AdminLogin.js` showed the admin side uses its own cookie session (`axios.create({baseURL:"/api/admin", withCredentials:true})`), nothing to do with the user JWT, and that the login component calls `login(email, password)` — it takes an `email` and `password`, not a username. That mattered: my first attempts were a small scripted sweep of common admin pairs (`admin:admin`, `admin:admin123`, `admin:Password1`, `admin:letmein`, `admin:changeme`, `admin:imgurr`...) sent as `{"username":...}`, and every one bounced back `{"error":"invalid credentials"}` — because I had the field name wrong, not necessarily the password.

Once I fixed the field to `email`, the first thing I tried worked: `admin@imgurr.test` / `password`.

```http
POST /api/admin/login HTTP/1.1
Host: TARGET
Content-Type: application/json

{"email":"admin@imgurr.test","password":"password"}
```

`200 OK`, the admin user object (`id:1, role:admin`), and an admin session cookie. That cookie opens the whole admin API — the dashboard counts users, listings, orders and revenue across the entire instance:

![The admin panel after logging in with default credentials — 306 users, 338 listings, 202 orders, 28,364 MWR-Bucks revenue](Screenshot/Screenshot%202026-10-09%20at%2013.50.20.png)

**Impact.** Default admin creds are a complete bypass of the admin boundary. Once in, you have every admin function — the user list, seller approvals, user promotion (IMG-08). It's effectively an open door.

**One thing I'll note on the side:** `GET /api/admin/users`, with that session, returns the full user table — 305+ records, emails, roles, balances, lock status. So the default credentials also gate a sizeable chunk of personal data.

![The admin user list exposing every account's email, role and balance — my own buyer accounts (305/306) are in here too](Screenshot/Screenshot%202026-10-09%20at%2013.58.21.png)

I checked `/api/admin/stats` and `/api/admin/me` as well, hoping for an admin flag, but neither carried one — so I'm not claiming a flag here that isn't there.

**Remediation.**
- Strip seed/default credentials before anything that isn't local dev; force a reset on first admin login.
- Strong password policy and rate-limiting on `/api/admin/login`.
- MFA for admins, and ideally network-restrict the admin interface.

---

## IMG-08 — Privilege escalation via the admin API

| | |
|---|---|
| **Risk rating** | High |
| **Type** | Broken access control / improper privilege management (CWE-269, CWE-285) |
| **Flag** | None returned directly |
| **Affected endpoints** | `POST /api/admin/users/{id}/promote`, `POST /api/admin/sellers/{id}/approve` |

Following on from the admin session in IMG-07, the admin API exposes endpoints that change other users' roles and seller status. I took a plain buyer account of mine (id 306) and, through the admin API, promoted it to admin and approved its own pending seller application — which also skips straight past the intended apply → pending → review workflow.

```http
POST /api/admin/users/306/promote    → 200 OK
POST /api/admin/sellers/306/approve   → 200 OK
```

Afterwards, 306 reads back as `is_admin:true, role:admin, is_seller:true, seller_approved:true` in the user table. So the escalation stuck — the same account that was a `Buyer` a moment earlier now shows as `Admin`:

![User 306 (Sibz) now listed with the Admin role after the promote call](Screenshot/Screenshot%202026-10-09%20at%2013.58.45.png)

**Impact.** Anyone who reaches the admin session — which, thanks to IMG-07, is anyone — can hand out admin rights and approve sellers at will. That makes the whole role model attacker-controlled, and the seller-approval step essentially decorative.

**Remediation.**
- Authorise each admin action on its own merits server-side, not just "is this an admin session".
- Require logged, separated approval for role elevation; don't let the same privilege both promote users and approve sellers.
- Audit-log every promote/approve with actor, target and time.

---

## IMG-09 — Unrestricted file upload (non-image types accepted)

| | |
|---|---|
| **Risk rating** | High |
| **Type** | Unrestricted upload of dangerous file type (CWE-434) |
| **Flag** | None returned directly |
| **Affected endpoint** | `POST /api/uploads/listings` |

With the account escalated to an approved seller (IMG-08), the Sell flow becomes reachable and hands out an upload target via `POST /api/uploads/listings`. The form advertises "Image file (PNG, JPG, GIF, WebP — max 8 MB)", so the stated contract is image-only.

![The Sell form accepting a file named php-reverse-shell.png with a description that says the quiet part out loud](Screenshot/Screenshot%202026-10-09%20at%2014.35.55.png)

The listing is created and the non-image file is accepted — the detail page renders a broken thumbnail because the stored "image" isn't one:

![The resulting listing — the uploaded non-image is accepted and listed, broken thumbnail and all](Screenshot/Screenshot%202026-10-09%20at%2014.37.14.png)

At the API level, giving the endpoint a `.php` filename returns a live `upload_url` that *also* ends in `.php`:

```json
{"image":{...,"title":"hmmmm2","seller_id":306},"status":"listed",
 "upload_url":"/api/uploads/files/12b951df061644a6bd559c1bf2cfc787.php"}
```

An image marketplace that accepts a `.php` filename and mints a `.php` upload URL plainly isn't enforcing image-only types. That's the finding.

**Where I stopped, and why.** The natural next move is to upload something executable and call it — a web shell — and I did stage a standard pentestmonkey `php-reverse-shell` to have it ready.

![A pentestmonkey php-reverse-shell staged locally — prepared, then deliberately left unused](Screenshot/Screenshot%202026-10-09%20at%2014.32.52.png)

But actually serving and invoking it is code execution on the host, which the RoE explicitly puts out of scope ("all host-based privilege-escalation activity is explicitly out of scope"). So I did not fire it. The weakness stands on the type-handling alone; I didn't need to pop a shell to show the control is missing, and pushing past the scope line would have been the wrong call even though the door looked open.

**Impact.** Accepting arbitrary non-image uploads is bad regardless of whether you chase RCE. Depending on storage and serving, it can mean stored XSS (HTML or SVG), content-type confusion, or attacker-controlled files living on the application's own origin. At minimum it breaks the integrity of the marketplace.

**Remediation.**
- Validate by content (magic bytes) and re-encode images server-side. Don't trust the filename or the `Content-Type` header.
- Allowlist image extensions, store uploads outside any executable path, rename them, drop the original extension.
- Serve user content from a separate non-executing domain with a locked-down `Content-Type` and `Content-Disposition`.

---

## Lower-severity and informational

| ID | Finding | Rating | Evidence |
|---|---|---|---|
| INFO-01 | Server version disclosure | Informational | `Server: nginx/1.24.0 (Ubuntu)` on responses; `server_tokens` left on |
| INFO-02 | Permissive CORS on some endpoints | Low | `Access-Control-Allow-Origin: *` on `GET /api/shop/images/{id}` (wildcard, no creds) |
| INFO-03 | Verification hash in purchase response | Informational | Purchase returns `verification_hash` → `/verify/{hash}`; worth testing whether other buyers' certificates are guessable |
| INFO-04 | Sequential order IDs | Low | `order.id` increments (202, 204...); no IDOR proven yet, but sequential IDs make enumeration easy — retest `/api/shop/orders/{id}` |

A couple of cross-cutting notes while they're fresh:

- The comment endpoint encodes its output and the review endpoint doesn't (IMG-01). That inconsistency is itself the story — it points at sanitisation bolted on per-endpoint instead of centrally, which is the kind of thing that'll keep reappearing.
- The main API and the admin panel run two separate auth systems (JWT vs. admin cookie). I'm flagging it because it genuinely changed how I had to think about which finding reaches which surface — the forged JWT from IMG-03 is useless against the admin side, which I only learned by trying.
- The admin dashboard carries a "Check Network Connection" host/IP ping box. I noted it as a candidate for command injection / SSRF but did not test it, since reaching the host is the same out-of-scope line as the upload RCE.

---

## Further testing — things I lined up but didn't get to

I'd rather be explicit about what's unfinished than let the gaps read as "didn't notice". None of the below was executed; each is a hypothesis with a reason to believe it, not a result.

**Economy and business logic (follows IMG-04).** Negative or fractional `price_bucks`/quantity on a purchase, to see if a debit flips into a credit when the maths is `balance - (price * qty)`. Idempotency and double-spend — repeat or concurrent purchase submits to see if one debit releases goods twice, or one buy debits twice. The upvote `{"purchase":true}` path forced without balance, with a negative cost, or raced.

**Vouchers and coupons (follows IMG-04).** A single-use coupon raced across parallel requests — the classic TOCTOU where `is_redeemed` is checked before it's set, with no row lock. Reusing a voucher, and stacking codes to push a total to zero or negative. And the whole `coupons` / `coupon_redemptions` feature, which IMG-06 revealed exists and which I never touched.

**Object-level auth (follows INFO-03/04).** `GET /api/shop/orders/{id}` walked across sequential IDs as a normal user, to test cross-user order access. The `/verify/{hash}` certificate — whether another buyer's can be read, guessed, or forged. And re-running object access with the admin session to see where vertical vs. horizontal control actually breaks.

**Second XSS (follows IMG-05).** `<img src=x onerror=...>` into a listing's title and description via the Sell flow, then check the create response and the rendered page for a second `flags.xss`.

**Upload hardening (follows IMG-09).** MIME/magic-byte bypass and an SVG with embedded script, beyond the `.php` acceptance I already confirmed; and working out where uploads land and whether the IMG-02 read can reach them.

**Admin ping box.** The "Check Network Connection" field — test for command injection (`; id`, `$(...)`, backticks) and SSRF to loopback/metadata, scoped to *proving the request/execution happens* and no further, since anything past that crosses the host line.

**PDF renderer, if there is one.** Vouchers can be downloaded as a PDF "with QR code". If that PDF is rendered from user input by a headless browser, that's a classic SSTI/SSRF sink — `{{7*7}}` to test template injection, then whether the renderer can be made to fetch an internal/loopback service. I'd scope that to *proving the request goes out* and no further; actually reaching into the host is the same out-of-scope line as the upload RCE.

**The traversal flag (follows IMG-02).** Depth-walking and absolute-path variants of `?filename=` to find the `flag` file the sandbox user can read. The vulnerability's already proven — this is just locating the file, and the identical-error behaviour makes it slow going, which is honestly why I set it aside.

---

## Appendix

### Flags captured

| # | Where | Flag | Endpoint |
|---|---|---|---|
| 1 | Stored XSS (IMG-01) | `MWR{227ad8df8a3c15d20943d3472ca43d6b}` | `POST /api/shop/images/2/review` |
| 2 | SQL injection (IMG-06) | `MWR{730067fa8a37de7a64dce4fa0f859189}` | `GET /api/shop/images?q=` (`ctf_sqli_flags`) |
| 3 | JWT weak secret (IMG-03) | `MWR{a698e61eccbd5875d815eb49334414d2}` | `GET /api/auth/me` |

Three of a possible ~14. The rest I'd go after are scoped above.

### Test environment

- **Target:** my own activated instance of the Imgurr room (titled `MWR_Internship_Challenge_2026_December_V1`). The host address is per-instance (`TARGET` here) and changed across VM reboots; activation is at `/activate`, case-sensitive, and has to be redone each boot.
- **Account:** a standard buyer (id 305, then 306 after a re-activation — hence both numbers appearing in the captures).
- **Tools:** Burp Suite for proxying, Repeater and the JWT Editor; `curl` for replay; `jq` for reading JSON; `sqlmap` to confirm the injection; John the Ripper and `jwt_tool` for the JWT work; a bit of Python and `strings` to pull apart the client bundles.

### How I worked

I started by mapping the API surface off the JavaScript bundles — including the code-split admin chunks, which is where the admin routes were hiding — and off proxied traffic. Then I worked each area (auth, shop, reviews/comments, vouchers, sell, admin) for injection, access control, business logic and file handling. Flags came out of raw JSON responses, since nothing renders in the UI. I went for the cheap high-value stuff first and stopped where the next flags clearly cost more than the time I had. Everything stayed inside the application boundary — no host-level code execution or privilege escalation was attempted, per the rules of engagement.

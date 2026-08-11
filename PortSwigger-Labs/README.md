# PortSwigger Web Security Academy — Lab Write-ups

A collection of individual vulnerability labs completed on PortSwigger's Web Security Academy.

## SQL Injection – Blind, Time-Based

**Difficulty:** Practitioner

**Vulnerability:** The application inserted the `TrackingId` cookie value directly into a SQL query used for analytics. The query's results weren't returned to the page, and the app behaved identically whether the query matched rows or errored — so there was no visible output to confirm injection.

**Payload:** `'||(SELECT pg_sleep(10))--` injected into the `TrackingId` cookie

**Result:** The response took **10,295 ms** to return (vs. near-instant normally), confirming the injected subquery executed synchronously on the server — proof of a blind SQL injection point despite zero visible output.

**Fix:** Use parameterized queries/prepared statements so cookie values are never concatenated into SQL. Never trust client-supplied cookies as safe input, even for "just analytics."

<img width="1048" height="644" alt="image" src="https://github.com/user-attachments/assets/963957e8-643b-4456-91e1-cdb31b113463" />

*The TrackingId cookie payload in Burp Repeater, with the response taking 10,295 ms to confirm the delay.*

<img width="1600" height="747" alt="image" src="https://github.com/user-attachments/assets/9a4d0866-dca9-4e9b-be8c-4f09f769ee47" />

*PortSwigger's lab-solved confirmation for Blind SQL Injection with Time Delays.*

---

## SQL Injection – Login Bypass

**Difficulty:** Apprentice

**Vulnerability:** The application's login function inserted the submitted username directly into a SQL query without sanitization, allowing the query's logic to be altered by injecting SQL syntax through the username field.

**Payload:** `administrator'--` entered in the Username field (password field left as any value)

**Result:** The `--` commented out the rest of the SQL query (including the password check), so the query effectively became "log in as administrator" with no password verification required. Successfully logged in as `administrator`, confirmed by the account page showing `Your username is: administrator` and the lab's "solved" banner.

**Fix:** Use parameterized queries/prepared statements for all authentication queries — never concatenate user input directly into SQL. Additionally, avoid revealing whether a username exists via differing error messages.

<img width="1600" height="755" alt="image" src="https://github.com/user-attachments/assets/5b9f8291-5544-4c5c-b848-1c8b4cfe08ec" />

*The payload `administrator'--` submitted in the Username field.*

<img width="1600" height="718" alt="image" src="https://github.com/user-attachments/assets/05166f0c-0c33-4ac5-a2ff-7a4800f2f1a7" />

*Logged in as `administrator` with the query's password check bypassed — confirmed by the lab-solved banner.*

---

## SSRF – Basic SSRF Against the Local Server.

(This lab has a stock check feature which fetches data from an internal system. To solve the lab, change the stock check URL to access the admin interface at `http://localhost/admin` and delete the user `carlos`.)



**Difficulty:** Apprentice

**Vulnerability:** The application's stock-check feature made a server-side request to a `stockApi` URL parameter to fetch stock data from an internal system (`stock.weliketoshop.net`). Because the server trusted this URL parameter without validating it against an allow-list, it could be redirected to any internal address — including services never meant to be reached from outside the network.

**Payload:** The original parameter, `stockApi=http://stock.weliketoshop.net:8080/product/stock/check?productId=2&storeId=1`, was intercepted in Burp and replaced with:
`stockApi=http://localhost/admin/delete?username=carlos`

**Result:** The server made the forged request on the application's behalf, reaching its own internal admin interface at `localhost/admin` — inaccessible directly from outside — and executed the delete action, removing the user `carlos`. Confirmed by the lab's "solved" banner.

**Fix:** Validate and allow-list all server-side request destinations; never let user-controllable input directly set a URL the server will fetch. Segment internal admin interfaces so they aren't reachable even from the application server itself, and require separate authentication for admin actions.

<img width="1174" height="359" alt="image" src="https://github.com/user-attachments/assets/56dc2222-cd75-4b48-b319-33b9aa5859e1" />

*The stock-check feature's normal request to `stock.weliketoshop.net`, before tampering.*

<img width="1600" height="738" alt="image" src="https://github.com/user-attachments/assets/f1bab08a-bae6-49f5-b674-37a7da22409a" />

*The `stockApi` parameter replaced with `http://localhost/admin/delete?username=carlos` in Burp Suite.*

<img width="1600" height="779" alt="image" src="https://github.com/user-attachments/assets/1f86e2a8-e3f2-4153-8c9e-fcad277256b4" />

*Confirmation that the forged server-side request successfully deleted the user `carlos` via the internal admin interface.*

---

## IDOR – Insecure Direct Object References (Chat Transcripts)

(This lab stores user chat logs directly on the server's file system, and retrieves them using static URLs.
Solve the lab by finding the password for the user `carlos`, and logging into their account.)

**Difficulty:** Apprentice

**Vulnerability:** The application's live chat feature stored each conversation transcript as a file on the server, retrievable via a static, sequentially-numbered URL (`/download-transcript/N.txt`). Because the server didn't verify that the requesting user actually owned that transcript number, any authenticated user could access any other user's chat logs simply by changing the number in the URL.

**Payload:** Intercepted a request for `/download-transcript/3.txt` (my own session's transcript) in Burp and changed it to `/download-transcript/1.txt`.

**Result:** The server returned transcript `1.txt` without any ownership check — a chat log belonging to another user, `carlos`, who had asked the support bot to confirm a forgotten password. The transcript exposed his plaintext password (`ecvxwf6g74giwpheo8bb`) in the bot's response. Using this password, I logged in directly as `carlos`, confirmed by the account page showing `Your username is: carlos` and the lab's "solved" banner.

**Fix:** Never expose sensitive records via predictable, sequential identifiers. Enforce server-side authorization checks on every object access — verify the requesting user actually owns the resource before returning it, regardless of whether the URL itself "looks" valid. Additionally, sensitive data like passwords should never be transmitted or stored in plaintext, even within support chat logs.

<img width="1466" height="900" alt="image" src="https://github.com/user-attachments/assets/2885c6d0-5279-43c2-a2ca-0b19bfcfebcb" />

*The original request for my own chat transcript (3.txt), intercepted in Burp Suite.*


<img width="1494" height="859" alt="image" src="https://github.com/user-attachments/assets/79f1fbc6-4ddf-4c6a-a6ad-57c2bfb86e95" />

*The transcript ID changed from 3 to 1 in Burp Repeater — the response returned `carlos`'s chat log, exposing his plaintext password.*


<img width="1549" height="766" alt="image" src="https://github.com/user-attachments/assets/4f448639-99f9-43d7-90b4-041691c9666f" />

*Logged in as `carlos` using the password extracted from the exposed transcript — confirmed by the lab-solved banner.*

---

## CSRF – Vulnerability With No Defenses

(This lab's email change functionality is vulnerable to CSRF.
To solve the lab, craft some HTML that uses a CSRF attack to change the viewer's email address and upload it to your exploit server.
You can log in to your own account using the following credentials: `wiener:peter`)

**Difficulty:** Apprentice

**Vulnerability:** The application's email-change endpoint (`/my-account/change-email`) accepted POST requests with no CSRF token, no origin/referer check, and no other anti-CSRF protection. Because the browser automatically attaches the logged-in user's session cookie to any request sent to the site — even one triggered from an attacker's page — this endpoint could be forged from outside the application entirely.

**Payload:** Copied a legitimate email-change request from Burp Suite and used a CSRF PoC generator tool (CSRFShark) to convert it into an auto-submitting HTML form:
```html
<form method="POST" action="https://[lab-id].web-security-academy.net/my-account/change-email">
  <input type="hidden" name="email" value="thor@gmail.com">
  <input type="submit" value="Submit Request">
</form>
```

**Result:** Hosting this HTML and having the logged-in victim (`wiener`) load it caused the form to auto-submit using their active session — silently changing their account email to `thor@gmail.com` with no interaction beyond loading the page. Confirmed by the account page updating to show `Your email is: thor@gmail.com`.

**Fix:** Implement anti-CSRF tokens on all state-changing requests, tied to the user's session and validated server-side on submission. Additionally, check the `Origin`/`Referer` header to confirm requests originate from the application itself, and apply `SameSite=Lax` or `Strict` on session cookies to prevent them from being sent on cross-site requests.

<img width="1600" height="749" alt="image" src="https://github.com/user-attachments/assets/ae6d3401-4136-45e2-b93f-a2abecf80c70" />

*The auto-submitting HTML form generated from the intercepted request, with `email=thor@gmail.com` set as the hidden payload.*

<img width="1012" height="547" alt="image" src="https://github.com/user-attachments/assets/539591d5-3c19-4aa3-812f-4da08132c65f" />

*Confirmation that the victim's email was silently changed to `thor@gmail.com` after the forged form was submitted.*

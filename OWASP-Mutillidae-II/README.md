# Compromising Web Applications — Session ID & Cookie Vulnerability Assessment

**Target:** OWASP Mutillidae II (TryHackMe AttackBox lab)
**Category:** Deliberately vulnerable web application
**Tools:** Burp Suite Community Edition, Wappalyzer, Chromium/Firefox
**Scope:** Authorized lab traffic only, against the assigned TryHackMe target environment

---

## Objective

Demonstrate how weak session cookie handling can expose or alter authenticated application state, by:
1. Collecting target and technology-stack details
2. Registering and logging in as a test user
3. Inspecting and manipulating cookies via Burp Suite
4. Running controlled proof-of-concept (PoC) exploits
5. Documenting impact and remediation guidance

---

## Recon & Technology Stack

Using Wappalyzer against the lab target, the following stack was fingerprinted:

| Component | Detail |
|---|---|
| Web server | Nginx 1.18.0 |
| Language | PHP 7.4.13 |
| JS library | jQuery 1.8.3 |
| Reverse proxy | Nginx 1.18.0 |
| Payment processor (present in app) | PayPal |

<img width="668" height="725" alt="image" src="https://github.com/user-attachments/assets/fc45dd85-6111-4245-a82d-447f452f425f" />

*Wappalyzer scan identifying the target's technology stack: Nginx, PHP 7.4.13, jQuery, and PayPal integration.*

---

## Lab Setup Procedure

1. Opened TryHackMe OWASP Mutillidae II.
2. Started the target machine.
3. Recorded the target IP and expiry time.
4. Opened the AttackBox.
5. Launched Burp Suite and browser.
6. Browse to the target IP.

<img width="703" height="336" alt="image" src="https://github.com/user-attachments/assets/9a1b34b1-259e-4851-b1d6-1afec5e5806c" />
<img width="703" height="351" alt="image" src="https://github.com/user-attachments/assets/7d66726c-a684-478b-8925-4cf940389616" />

*Screenshots show the assigned TryHackMe target and Burp&browser setup*

## Account Creation & Login

**Step 1: Register and Log In**

1. Registered a test account (`Harry_test`).
2. Logged in with the test account.

<img width="685" height="556" alt="image" src="https://github.com/user-attachments/assets/2437d9d0-36a5-4533-82f2-0c974ef8f606" />
<img width="688" height="526" alt="image" src="https://github.com/user-attachments/assets/549a4a40-8bde-41c3-a96c-a2653d3cd6f1" />

*Screenshots show the Test account `Harry_test` registered and logged into OWASP Mutillidae II*

**Step 2: Confirm Session State**

- After login, the application displayed the current user (`Harry_test`) in the top navigation.
- This confirmed a normal, authenticated session was active.
- Establishing this baseline was necessary before attempting any cookie tampering — the PoC later compares this normal user context against the tampered-cookie context.

<img width="728" height="698" alt="image" src="https://github.com/user-attachments/assets/64b583c2-e549-4609-afd0-7a550fb581fa" />

*Mutillidae dashboard confirming the authenticated session as `Harry_test`, before any cookie modification.*

---

## Vulnerability 1: Cookie Tampering → Broken Access Control

**OWASP Category:** A01:2021 – Broken Access Control
**CVSS 3.1 Score:** 9.1 (Critical)

### Steps

1. Enabled Intercept in Burp Proxy.
2. Refreshed a Mutillidae page to capture an authenticated HTTP request.
3. Inspected the `Cookie` header and identified four fields: `PHPSESSID`, `username`, `uid`, `showhints`.
4. Modified the intercepted request:
   - `username=Harry_test` → `username=admin`
   - `uid=24` → `uid=1`
5. Forwarded the edited request.
6. Observed that the application accepted the modified identity context — effectively granting administrative access without authentication.

```
Cookie: PHPSESSID=<session_id>; username=admin; uid=1; showhints=1
```

<img width="745" height="358" alt="image" src="https://github.com/user-attachments/assets/7ed8f711-8262-4f78-ba32-eb8050959d6c" />

<img width="745" height="346" alt="image" src="https://github.com/user-attachments/assets/62957534-697f-4c27-9d50-c222f87ab9df" />

*Cookie tampering via Burp Suite — the intercepted request's Cookie header was edited from `username=Harry_test`/`uid=24` to `username=admin`/`uid=1`, and the application accepted the modified values, granting an administrative identity context and confirming the Broken Access Control vulnerability.*

### Impact

**Technical:**
- Client-side cookie values were trusted and editable before reaching the server.
- Because the server trusted `username`/`uid` directly, authorization was fully bypassable.
- An attacker could impersonate any user, including an administrator.

**Business:**
- Unauthorized access to protected user/admin functionality.
- Loss of confidentiality (account data, records) and integrity (data modified as another user).
- Reputational and compliance risk if this pattern existed in production.

---

## Vulnerability 2: Reflected XSS → Session Token Disclosure

**OWASP Category:** A03:2021 – Injection
**CVSS 3.1 Score:** 6.1 (Medium)

### Steps

1. Navigated to `OWASP 2017 > A7 Cross Site Scripting > Reflected > DNS Lookup`.
2. Used the Hostname/IP input field as the injection point.
3. Submitted the following payload:

```html
<script>alert('Hacked Cookie: ' + document.cookie);</script>
```

4. Clicked "Lookup DNS" to trigger the reflected input.
5. The resulting browser alert printed live values from `document.cookie`, including `PHPSESSID` and `showhints` — confirming the session cookie was readable via client-side script.

<img width="750" height="370" alt="image" src="https://github.com/user-attachments/assets/bc31e8b6-101a-4dd9-ae40-ec52c050b397" />

<img width="750" height="338" alt="image" src="https://github.com/user-attachments/assets/a3cbf89d-e26b-4332-aeba-39a869f3b03b" />

*Screenshots: the XSS payload submitted via the DNS Lookup field (top), and the resulting browser alert disclosing the live session cookie values (bottom).*

 ### Impact

- Demonstrates why session cookies require the `HttpOnly` flag — without it, any injected script can read and exfiltrate session identifiers.
- A real attacker could use this to hijack an authenticated session (session token theft) without needing the victim's credentials.

---

## Impact Summary

| Vulnerability | OWASP Category | CVSS 3.1 / Risk | Primary Impact |
|---|---|---|---|
| Cookie Tampering | A01:2021 – Broken Access Control | 9.1 / **Critical** | Administrative Account Takeover |
| Reflected XSS | A03:2021 – Injection | 6.1 / **Medium** | Session Token Theft |

---

## Recommendations & Mitigation

**Cookie & Session Controls**
- Store authorization state server-side — never trust editable client-side cookie values.
- Set `HttpOnly` on session cookies so client-side JavaScript cannot read them.
- Set `Secure` so cookies are only transmitted over HTTPS.
- Set `SameSite=Lax` or `Strict` to reduce cross-site request risk.
- Rotate session IDs after login and after any privilege change.
- Expire sessions after inactivity and invalidate them on logout.

**Application Controls**
- Validate authorization server-side on every sensitive action — never rely on client-supplied identity fields.
- Encode output and sanitize all reflected input to prevent XSS.
- Implement a strict Content Security Policy (CSP) to limit script execution impact.
- Log suspicious cookie modifications and repeated privilege-boundary failures.

---

## Conclusion

This assessment demonstrated two related session-security weaknesses in a controlled lab environment: cookie tampering that allowed identity/privilege escalation, and reflected XSS that exposed readable session cookies. Both point to the same root cause — session and identity state that isn't adequately protected server-side — and the same core fix: server-side authorization, protected cookies (`HttpOnly`/`Secure`/`SameSite`), output encoding, and a strong CSP.

---

## References

- [TryHackMe – OWASP Mutillidae II Room](https://tryhackme.com/room/owaspmutillidae)
- [OWASP Mutillidae II Project](https://owasp.org/www-project-mutillidae-ii/)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [PortSwigger Burp Suite Documentation](https://portswigger.net/burp/documentation)
- [Wappalyzer](https://www.wappalyzer.com/)

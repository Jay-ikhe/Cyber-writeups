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



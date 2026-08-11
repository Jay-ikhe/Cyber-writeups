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



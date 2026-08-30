# 2FA Bypass Using a Brute-Force Attack

## Overview

This lab demonstrates how a two-factor authentication mechanism can remain vulnerable even when the application limits the number of incorrect MFA attempts per login session.

The application allowed only two incorrect 4-digit security-code submissions before terminating the current authentication flow and redirecting the user back to the login page.

At first glance, this appeared to prevent a conventional `0000-9999` brute-force attack. However, the protection was tied to the current login session rather than to the user account or second-factor challenge itself.

By automatically creating a fresh authenticated session before every MFA attempt, the per-session attempt limit could be bypassed.

---

## Observed Authentication Flow

The normal login flow was:

```text
GET /login
   |
   v
POST /login
username=<target>&password=<password>
   |
   v
GET /login2
   |
   v
POST /login2
csrf=<token>&mfa-code=<4-digit-code>
```

The application permitted two incorrect MFA submissions.

After the second failed attempt, the current authentication flow was invalidated and the browser was redirected back to:

```text
/login
```

This prevented a simple Intruder attack from testing all 10,000 possible codes in one session.

---

## Initial Security Control

Conceptually, the protection behaved like:

```text
Login session created
      |
      v
MFA attempt #1 -> incorrect
      |
      v
MFA attempt #2 -> incorrect
      |
      v
Invalidate login flow
      |
      v
Redirect to /login
```

The weakness was that the failed-attempt counter was associated with the temporary authentication flow rather than being enforced strongly enough at the account or challenge level.

A new login produced a fresh opportunity to submit MFA codes.

---

## Bypass Strategy

Instead of trying multiple codes inside the same authentication session, a fresh login flow was created before each MFA attempt:

```text
Create fresh login session
        |
        v
Submit one MFA candidate
        |
        v
Create another fresh login session
        |
        v
Submit next MFA candidate
        |
        v
Repeat
```

This changes the attack from:

```text
One login -> thousands of MFA attempts
```

to:

```text
Thousands of logins -> one MFA attempt per login
```

The two-attempt limit therefore never becomes an effective barrier.

---

## Burp Macro

A Burp Suite macro was created to automatically reproduce the login sequence:

```text
GET /login
POST /login
GET /login2
```

The `POST /login` request contained the target account credentials.

Testing the macro confirmed that it consistently finished on the second-factor page with a fresh authentication state.

The macro therefore automated the state transition required before each MFA attempt.

---

## Session Handling Rule

A Session Handling Rule was configured to run the login macro before requests to:

```text
POST /login2
```

The rule was scoped to the relevant authentication endpoint and testing tools.

Its purpose was to:

1. execute the login macro;
2. obtain a fresh authenticated session;
3. extract the latest CSRF token;
4. update the outgoing MFA request;
5. preserve the current `mfa-code` payload.

The resulting request followed this structure:

```http
POST /login2 HTTP/2
Host: <lab-host>
Cookie: session=<fresh-session>
Content-Type: application/x-www-form-urlencoded

csrf=<fresh-csrf-token>&mfa-code=0000
```

The same process was repeated automatically for subsequent candidate codes.

---

## Important Parameter-Handling Detail

During testing, the Session Handling Rule initially replaced too many request parameters.

This caused the outgoing request to lose the `mfa-code` value and resulted in:

```http
HTTP/1.1 400 Bad Request
```

with:

```text
Missing parameter 'mfa-code'
```

The rule was corrected so that only the values that actually needed refreshing were updated:

```text
session cookie -> refresh
csrf parameter -> refresh
mfa-code       -> preserve payload
```

After this change, test requests behaved correctly and returned the expected invalid-code response instead of a malformed-request error.

---

## Validating the Automation

Before attempting the full keyspace, a small payload set was used:

```text
0000
0001
0002
```

Each candidate was submitted after a fresh macro-generated login session.

The requests reached the MFA endpoint successfully without triggering the two-attempt session lockout.

This validated the central bypass technique:

> The MFA attempt restriction could be reset by creating a new authentication session before each code submission.

---

## Brute-Force Range

Because the MFA code consisted of four decimal digits, the complete keyspace was:

```text
0000
0001
0002
...
9999
```

or 10,000 possible values.

A successful code would be distinguishable from an incorrect code through the application's response behavior.

Typical failed response:

```http
HTTP/2 200 OK
```

with an incorrect-security-code message.

Expected successful response:

```http
HTTP/2 302 Found
Location: /my-account
```

The status-code difference therefore acts as an authentication oracle.

---

## Turbo Intruder Testing

Turbo Intruder was also evaluated to increase request throughput.

The payload-generation logic was:

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        engine=Engine.BURP,
        concurrentConnections=1
    )

    for x in xrange(10000):
        code = '%04d' % x
        engine.queue(target.req, code)


def handleResponse(req, interesting):
    if req.status == 302:
        table.add(req)
```

Using Burp's request engine allowed the Session Handling Rule and macro-generated authentication state to remain part of the request workflow.

In the training environment, however, the full 10,000-request run was not completed during this session because the macro-driven request chain was too slow to make the complete brute-force practical.

The exploit workflow itself was validated with test payloads.

---

## Root Cause

The root cause was **insufficient brute-force protection on the MFA challenge**.

The application limited failed MFA attempts only within the current authentication session.

It failed to enforce a sufficiently strong limit against:

```text
user account
MFA challenge
source behavior
repeated re-authentication
```

As a result, the attacker could reset the effective attempt counter by logging in again.

The protection therefore restricted the workflow instance rather than the attacker's total attempts against the second factor.

---

## Impact

An attacker who already possesses a valid username and password could systematically attack the second authentication factor.

Potential consequences include:

- bypass of multi-factor authentication;
- account takeover despite MFA being enabled;
- compromise of sensitive or privileged accounts;
- reduction of MFA security to the entropy of a short numeric code.

A four-digit code contains only 10,000 possible values, so effective rate limiting and challenge management are essential.

---

## Remediation

The application should enforce MFA brute-force protection independently of the temporary login session.

Recommended controls include:

- apply failed-attempt limits to the user account and active MFA challenge;
- invalidate or rotate MFA challenges after repeated failures;
- introduce exponential backoff or temporary account-level throttling;
- prevent repeated login attempts from resetting the MFA failure counter;
- expire MFA codes quickly;
- monitor repeated MFA failures across multiple authentication sessions;
- use sufficiently strong second-factor mechanisms;
- ensure that CSRF/session regeneration cannot be abused to reset security controls.

The system should treat repeated re-authentication followed by MFA failures as one continuous attack pattern rather than as independent legitimate login attempts.

---

## Key Takeaway

Rate limiting must be enforced against the **security-sensitive action**, not merely against a temporary workflow instance.

When testing MFA brute-force protection, the important question is not only:

> How many incorrect codes can I submit in one session?

but also:

> Can I reset that limit by restarting the authentication flow?

If a fresh login resets the MFA failure counter, session-level protection may provide little real resistance to brute-force attacks.

---

## Execution Note

The session-reset bypass, Burp macro, Session Handling Rule, CSRF refresh, and preservation of the MFA payload were reproduced and validated with test codes.

The full `0000-9999` enumeration was not completed during this session because the macro-driven request chain was too slow in the available test setup. No successful four-digit code is claimed in this write-up.

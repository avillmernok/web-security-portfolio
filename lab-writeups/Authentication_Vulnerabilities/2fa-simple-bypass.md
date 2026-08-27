# 2FA Simple Bypass

## Overview

This lab demonstrates a **two-factor authentication bypass caused by incomplete enforcement of MFA state**.

The application correctly redirected users to a second-factor verification page after successful password authentication.

However, access to the protected account page did not require confirmation that the second factor had actually been completed.

By authenticating with valid credentials and then directly requesting the protected account page, it was possible to bypass 2FA entirely.

---

## Normal Authentication Flow

The legitimate login process followed these steps:

```text id="kfa9sx"
POST /login
username=wiener&password=peter
        |
        v
302 Location: /login2
        |
        v
GET /login2
        |
        v
POST /login2
mfa-code=<valid-code>
        |
        v
302 Location: /my-account?id=wiener
        |
        v
Authenticated account
```

The second factor was therefore implemented as a separate step after password authentication.

---

## First-Factor Authentication

The target account credentials were:

```text id="d2o2hh"
username: carlos
password: montoya
```

Submitting these credentials resulted in:

```http id="w3oszf"
HTTP/2 302 Found
Location: /login2
```

At this point:

```text id="7liarh"
Password verification: successful
2FA verification: not completed
```

The application nevertheless issued a session that could be used in subsequent requests.

---

## Bypassing the Second Factor

Instead of submitting an MFA code, the protected account endpoint was requested directly:

```http id="6pwb9r"
GET /my-account?id=carlos HTTP/2
Host: <lab-host>
Cookie: session=<partially-authenticated-session>
```

The server returned Carlos's account page.

This confirmed that access to the protected resource did not require a completed MFA state.

---

## Why the Attack Worked

The vulnerable workflow behaved conceptually like this:

```text id="80y4w7"
Username + password correct
        |
        v
Create authenticated session
        |
        v
Redirect user to /login2
        |
        v
Assume user will complete MFA
```

The account page effectively checked only:

```text id="g60ipg"
Is there an authenticated session?
```

but failed to check:

```text id="v4lctb"
Has the second factor been successfully completed?
```

This allowed the attacker to skip the `/login2` step entirely.

---

## Workflow Enforcement vs. Security Enforcement

The application treated the 2FA page as part of the expected navigation flow:

```text id="3iv92e"
Password
   ↓
2FA page
   ↓
Account
```

However, HTTP clients are not required to follow the application's intended navigation flow.

An attacker can directly request any known endpoint:

```text id="8w7z4k"
/login
/login2
/my-account
```

Therefore:

> Redirecting a user to an MFA page does not enforce MFA.

The protected resource itself must verify that the required authentication state has been reached.

---

## Root Cause

The root cause was **missing MFA-state validation on protected resources**.

The application distinguished between the login steps in the user interface but did not consistently enforce those steps server-side.

A partially authenticated session was treated as sufficiently authenticated to access the account page.

Conceptually:

```text id="bp0alh"
password_verified = true
mfa_verified      = false
```

was incorrectly accepted by:

```text id="bjsqp6"
/my-account
```

A secure implementation should only allow access when:

```text id="dlj3ej"
password_verified = true
mfa_verified      = true
```

---

## Attack Chain

The complete attack was:

```text id="kma4n6"
Known target credentials
        |
        v
POST /login
carlos:montoya
        |
        v
First factor accepted
        |
        v
302 → /login2
        |
        v
Skip MFA page
        |
        v
Direct request to /my-account?id=carlos
        |
        v
Protected resource accessible
        |
        v
2FA bypass
```

---

## Impact

An attacker who obtained a user's password could bypass the second authentication factor.

This defeats the main security benefit of MFA.

Potential consequences include:

* account takeover;
* bypass of MFA protections;
* unauthorized access to sensitive data;
* compromise of privileged accounts;
* reduced protection against stolen credentials.

In this case, possession of Carlos's username and password was sufficient to access the account despite 2FA being enabled.

---

## Remediation

The application should track authentication state explicitly on the server side.

For example:

```text id="91on22"
Session state:
- first_factor_verified
- second_factor_verified
```

Protected resources should require both:

```text id="u01lup"
first_factor_verified == true
AND
second_factor_verified == true
```

before granting access.

Recommended controls include:

* Store MFA completion state server-side.
* Enforce MFA state on every protected endpoint.
* Do not treat redirects or page sequence as a security boundary.
* Restrict partially authenticated sessions to MFA-related endpoints only.
* Regenerate or upgrade the session after successful MFA where appropriate.
* Ensure APIs and direct URL access enforce the same MFA requirements as the UI.

---

## Secure Flow

A secure implementation should behave like:

```text id="ikqc2g"
POST /login
username + password
        |
        v
First factor verified
        |
        v
Partial session only
        |
        v
POST /login2
valid MFA code
        |
        v
Mark MFA verified
        |
        v
Full authenticated session
        |
        v
Protected resources allowed
```

If a partially authenticated user requests:

```http id="872sje"
GET /my-account
```

the server should reject the request or redirect back to MFA verification.

---

## Key Takeaway

When testing multi-factor authentication, do not only test whether the MFA form itself works correctly.

Also test whether it can be skipped.

After completing only the first authentication factor, directly request:

```text id="0g8f7e"
Account pages
Dashboard pages
API endpoints
Administrative functionality
Sensitive actions
```

The key question is:

> Does the protected resource independently verify that MFA has been completed?

If the answer is no, the second factor may exist only as a workflow step rather than as a real security boundary.

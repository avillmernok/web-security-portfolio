# Broken Brute-Force Protection, IP Block

## Overview

This lab demonstrates a **flawed brute-force protection mechanism based on client IP address**.

The application blocked further login attempts after several failed authentications from the same IP address.

However, the failed-attempt counter was reset whenever a successful login occurred from that IP.

By alternating password guesses for the target account with successful logins to a known account, the IP-based lockout could be repeatedly reset and the target password brute-forced.

---

## Vulnerability

The application implemented temporary IP blocking after multiple failed login attempts.

During testing, the following behavior was observed:

```text
Failed login attempt 1
Failed login attempt 2
Failed login attempt 3
Next attempt → blocked
```

This initially appeared to prevent brute-force attacks.

However, further testing revealed that a successful login using a valid account reset the failed-login counter associated with the IP address.

Known credentials:

```text
wiener:peter
```

Target account:

```text
carlos
```

---

## Identifying the Reset Behavior

The following sequence was tested manually:

```text
carlos : wrong-password-1
carlos : wrong-password-2

wiener : peter
```

The `wiener:peter` login succeeded.

After this successful authentication, additional failed attempts against Carlos were again allowed:

```text
carlos : wrong-password-3
carlos : wrong-password-4
```

This confirmed that the successful login reset the brute-force counter.

The vulnerable behavior can be represented as:

```text
Failed login
    |
    v
Counter +1

Failed login
    |
    v
Counter +1

Successful login
    |
    v
Counter reset to 0
```

---

## Exploitation Strategy

To prevent the IP from becoming blocked, the brute-force attack was structured so that every two failed attempts against Carlos were followed by a successful login as Wiener.

The request sequence was:

```text
carlos : password1
carlos : password2
wiener : peter

carlos : password3
carlos : password4
wiener : peter

carlos : password5
carlos : password6
wiener : peter
...
```

This allowed the failed-login counter to be reset continuously.

---

## Burp Intruder Configuration

The login request was sent to Burp Suite Intruder.

Two payload positions were configured:

```text
username=§username§&password=§password§
```

The attack type was:

```text
Pitchfork
```

This allowed the username and password lists to be paired line-by-line.

---

## Username Payload List

The username list followed this repeating pattern:

```text
carlos
carlos
wiener
carlos
carlos
wiener
...
```

---

## Password Payload List

The password list was modified to match the username list:

```text
password1
password2
peter
password3
password4
peter
...
```

This produced:

```text
carlos : password1
carlos : password2
wiener : peter
```

for every group of three requests.

---

## Request Ordering

The attack depended on requests being processed in the correct order.

The Intruder resource pool was therefore configured with:

```text
Maximum concurrent requests: 1
```

This ensured that Burp sent one request at a time.

Conceptually:

```text
Send request 1
      |
      v
Wait for response
      |
      v
Send request 2
      |
      v
Wait for response
      |
      v
Send request 3
```

If multiple requests were sent concurrently, Carlos login attempts could potentially reach the server before the successful Wiener login reset the counter.

---

## Unmodified Baseline Request Issue

During initial testing, the attack still triggered the IP block unexpectedly.

Burp Intruder was configured to send an **unmodified baseline request** before processing the payload list.

This appeared in the results as:

```text
Request 0
Payload1: empty
Payload2: empty
```

Because the original request contained invalid credentials, this baseline request counted as an additional failed login attempt.

The effective sequence became:

```text
Unmodified failed login
carlos : password1
carlos : password2
wiener : peter
```

This caused the failed-attempt counter to reach the limit before the reset occurred.

The following option was therefore disabled:

```text
Make unmodified baseline request
```

After removing the extra request, the attack proceeded correctly.

---

## Identifying the Correct Password

Most Carlos login attempts returned the normal failed-login response.

The known successful Wiener login produced:

```http
HTTP/2 302 Found
```

with a response length of approximately:

```text
188 bytes
```

One Carlos request produced the same successful authentication pattern:

```text
Username: carlos
Password: charlie
Status: 302
Length: 188
```

This identified the valid credentials:

```text
carlos:charlie
```

---

## Attack Chain

The complete attack was:

```text
Login endpoint
      |
      v
IP-based failed-login counter
      |
      v
Multiple failures cause temporary block
      |
      v
Test successful authentication behavior
      |
      v
Successful login resets counter
      |
      v
Build alternating credential sequence
      |
      v
2 × Carlos guesses
      |
      v
wiener:peter reset
      |
      v
Repeat
      |
      v
carlos:charlie returns 302
      |
      v
Account compromise
```

---

## Root Cause

The root cause was **incorrect state management in the brute-force protection mechanism**.

The application associated failed login attempts primarily with the source IP address and reset that state whenever any successful authentication occurred.

Conceptually:

```text
IP address
   |
   v
Failed-attempt counter
```

The vulnerable reset logic effectively behaved like:

```text
Any successful login from IP
           |
           v
Reset failed-attempt counter
```

This allowed an attacker who controlled one valid account to reset the protection while brute-forcing another account.

---

## Why the Protection Failed

The defense treated successful authentication as evidence that the client was no longer malicious.

That assumption is incorrect.

An attacker may legitimately control one account while attacking another:

```text
Attacker controls:
wiener:peter

Attacker targets:
carlos
```

A successful login to Wiener therefore says nothing about whether the same client is attempting to compromise Carlos.

---

## Impact

An authenticated attacker with access to any valid account could potentially bypass IP-based brute-force protection and perform password attacks against other users.

Potential consequences include:

* password brute forcing;
* credential compromise;
* account takeover;
* compromise of privileged accounts;
* bypass of authentication rate limits.

In this lab, the vulnerability enabled the password for the `carlos` account to be discovered.

---

## Remediation

Brute-force protection should not be reset globally simply because a successful authentication occurs from the same IP address.

Recommended controls include:

* Track failed attempts per account as well as per IP address.
* Do not reset attack-detection state for unrelated accounts after a successful login.
* Use progressive delays or account-specific throttling.
* Detect distributed and credential-stuffing attacks.
* Rate-limit authentication attempts using multiple signals.
* Use multi-factor authentication where appropriate.
* Monitor repeated login attempts against multiple accounts from the same source.

A more robust model might include:

```text
IP-based rate limiting
        +
Account-based throttling
        +
Behavioral detection
        +
MFA
```

rather than relying on a single IP-based counter.

---

## Key Takeaway

A brute-force defense is only effective if its state management cannot be manipulated by the attacker.

When testing login protections, ask:

> What causes the failed-login counter to increment, reset, or expire?

Also test whether:

```text
Successful login to another account
Password reset
Session renewal
IP spoofing headers
Account switching
```

affect the protection.

In this case, a successful login to a known account reset the same IP-based counter that was being used to protect another account, making the lockout mechanism ineffective.

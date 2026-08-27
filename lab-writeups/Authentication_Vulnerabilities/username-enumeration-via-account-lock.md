# Username Enumeration via Account Lock

## Overview

This lab demonstrates a **username enumeration vulnerability caused by account lockout behavior**.

The application locked valid user accounts after several failed login attempts.

Because invalid usernames could not be locked, the lockout message itself revealed whether a submitted username existed.

After identifying a valid username, the corresponding password was brute-forced, resulting in account compromise.

---

## Vulnerability

The application accepted credentials through:

```http
POST /login HTTP/2
Host: <lab-host>
Content-Type: application/x-www-form-urlencoded

username=test&password=wrongpassword
```

A normal failed login returned a generic authentication error.

However, after multiple failed attempts against an existing account, the application returned a different message:

```text
You have made too many incorrect login attempts.
Please try again in 1 minute(s).
```

This created a username enumeration oracle.

---

## Enumeration Strategy

No known valid target username was provided.

To discover one, each candidate username was submitted multiple times using the same incorrect password.

For example:

```text
carlos
carlos
carlos
carlos
carlos

root
root
root
root
root

admin
admin
admin
admin
admin
...
```

The corresponding login attempts were conceptually:

```text
carlos : wrongpassword
carlos : wrongpassword
carlos : wrongpassword
carlos : wrongpassword
carlos : wrongpassword

root : wrongpassword
root : wrongpassword
...
```

The goal was to trigger account lockout only for usernames that actually existed.

---

## Why This Works

The application behaved differently depending on whether the username was valid.

### Invalid username

```text
Candidate username
      |
      v
Account does not exist
      |
      v
Normal authentication failure
      |
      v
No account can be locked
```

### Valid username

```text
Candidate username
      |
      v
Account exists
      |
      v
Repeated incorrect passwords
      |
      v
Account lockout triggered
      |
      v
Different response
```

Therefore:

```text
Account lockout message
        ↓
Username existence oracle
```

---

## Burp Intruder Configuration

The login request was sent to Burp Suite Intruder.

The username parameter was selected as the payload position:

```text
username=§username§&password=wrongpassword
```

Each username in the supplied wordlist was repeated five times consecutively.

This ensured that the lockout threshold could be triggered for each candidate account.

---

## Identifying the Valid Username

The Intruder results were inspected for responses containing the lockout message:

```text
You have made too many incorrect login attempts.
Please try again in 1 minute(s).
```

One username produced this response:

```text
analyzer
```

This identified:

```text
Valid username: analyzer
```

The application had therefore revealed account existence through its brute-force protection mechanism.

---

## Password Brute Force

After identifying the valid username, the username was fixed:

```text
username=analyzer
```

The password parameter became the Intruder payload position:

```text
username=analyzer&password=§password§
```

A password wordlist was then tested.

The successful request produced:

```http
HTTP/2 302 Found
Location: /my-account?id=analyzer
```

while incorrect passwords returned the normal login page.

The correct credentials were:

```text
Username: analyzer
Password: 123456
```

---

## Success Oracle

The password attack had a clear success indicator.

Incorrect credentials:

```http
HTTP/2 200 OK
```

Successful credentials:

```http
HTTP/2 302 Found
Location: /my-account?id=analyzer
```

Therefore:

```text
302 redirect
     ↓
Successful authentication
```

In this case, the correct password appeared early in the wordlist, so account lockout did not prevent identification of the valid credential.

---

## Attack Chain

The complete attack was:

```text
Login endpoint
      |
      v
Repeat each username multiple times
      |
      v
Trigger account lockout
      |
      v
Observe different response
      |
      v
Valid username discovered
      |
      v
analyzer
      |
      v
Password brute force
      |
      v
302 redirect
      |
      v
Password: 123456
      |
      v
Account compromise
```

---

## Root Cause

The root cause was **different application behavior for valid and invalid usernames**.

The account lockout mechanism only applied to real accounts.

This meant that the security control itself exposed information about account existence.

Conceptually:

```text
Invalid username
      |
      v
No account state exists
      |
      v
Generic login failure


Valid username
      |
      v
Failed-attempt counter exists
      |
      v
Account lockout
      |
      v
Different response
```

The difference was sufficient to enumerate usernames automatically.

---

## Security Control as an Oracle

This lab demonstrates an important principle:

> A defensive mechanism can become an information disclosure channel if its behavior differs depending on sensitive internal state.

The account lockout feature was intended to prevent brute-force attacks.

Instead, it revealed:

```text
Does this username correspond to a real account?
```

through observable lockout behavior.

---

## Impact

An attacker could remotely identify valid user accounts.

This information could then be used for:

* targeted password brute-force attacks;
* password spraying;
* credential stuffing;
* phishing;
* identification of privileged or high-value accounts;
* account takeover.

In this lab, the enumeration vulnerability was chained with password brute forcing to compromise the `analyzer` account.

---

## Remediation

Account lockout mechanisms should not reveal whether a username exists.

Recommended controls include:

* Return consistent authentication responses for valid and invalid usernames.
* Avoid exposing account-specific lockout messages before authentication.
* Keep status codes and response structures consistent.
* Use generic authentication failure messages.
* Implement rate limiting using multiple signals.
* Apply progressive delays where appropriate.
* Monitor automated login attempts.
* Use multi-factor authentication for sensitive accounts.

For example, instead of revealing:

```text
Your account has been locked.
```

the application could return a generic response such as:

```text
Invalid username or password.
```

while still enforcing protective controls internally.

---

## Lockout Design Considerations

A secure implementation should separate:

```text
Internal security state
```

from:

```text
Externally observable authentication behavior
```

The application may internally track:

```text
Failed login count
Temporary account restrictions
Risk score
Client reputation
```

without exposing enough information for an attacker to infer whether an account exists.

---

## Key Takeaway

When testing login mechanisms, deliberately trigger brute-force protections and compare how the application behaves for different usernames.

Look for:

```text
Account lockout messages
Different response lengths
Different status codes
Different redirects
Different timing
Different headers
```

The key question is:

> Does the brute-force protection behave differently when the submitted username corresponds to a real account?

If the answer is yes, the protection itself may provide a username enumeration oracle.

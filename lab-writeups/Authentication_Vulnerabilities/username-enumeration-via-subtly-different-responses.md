# Username Enumeration via Subtly Different Responses

## Overview

This lab demonstrates a **username enumeration vulnerability caused by a very small difference in authentication responses**.

The application attempted to hide whether a username was valid by returning nearly identical login error messages.

However, valid and invalid usernames produced responses that differed by a single character.

Using Burp Suite Intruder together with `Grep - Extract`, this difference could be identified reliably and used to discover a valid username. The corresponding password was then brute-forced, resulting in account compromise.

---

## Vulnerability

The application accepted credentials through:

```http
POST /login HTTP/2
Host: <lab-host>
Content-Type: application/x-www-form-urlencoded

username=test&password=test
```

At first glance, failed authentication attempts appeared to return the same error message.

However, the responses were not completely identical.

Most invalid usernames returned:

```text
Invalid username or password.
```

One username produced:

```text
Invalid username or password
```

The only difference was the missing period at the end of the message.

---

## Username Enumeration

The login request was sent to Burp Suite Intruder.

The username parameter was selected as the payload position:

```text
username=§test§&password=test
```

A username wordlist was then used while keeping the password constant.

Instead of relying only on response length, `Grep - Extract` was configured to extract the login warning message from each response.

This made the subtle difference immediately visible in the Intruder results.

Most responses contained:

```text
Invalid username or password.
```

The response for:

```text
access
```

contained:

```text
Invalid username or password
```

without the trailing period.

This identified:

```text
Valid username: access
```

---

## Using Grep - Extract

Because the response difference was extremely small, manually reviewing every response would have been inefficient.

Burp Suite's `Grep - Extract` functionality was used to extract the authentication error message into a separate result column.

Conceptually:

```text
Payload      Extracted response
-----------------------------------------------
admin        Invalid username or password.
backup       Invalid username or password.
access       Invalid username or password
service      Invalid username or password.
```

This made the outlier easy to identify.

---

## Why the Difference Matters

The application did not explicitly reveal that the username existed.

Instead, it leaked the information indirectly through a single-character difference.

The observable behavior was:

```text
Invalid username
        ↓
"Invalid username or password."


Valid username + wrong password
        ↓
"Invalid username or password"
```

This creates a reliable **username enumeration oracle**.

An attacker does not need a large or obvious difference. Any consistent distinction between the two authentication paths can reveal whether an account exists.

---

## Password Brute Force

After identifying the valid username, the request was changed so that the username remained fixed:

```text
username=access
```

and the password became the Intruder payload position:

```text
username=access&password=§password§
```

A password wordlist was then tested.

The successful credential was identified as:

```text
Username: access
Password: monitor
```

These credentials could then be used to authenticate successfully.

---

## Attack Chain

The complete attack was:

```text
Login endpoint
      |
      v
Enumerate usernames
      |
      | Grep - Extract error message
      v
Compare subtle response differences
      |
      v
Missing "." identifies valid username
      |
      v
Valid username: access
      |
      v
Brute-force password
      |
      v
Valid password: monitor
      |
      v
Account compromise
```

The username enumeration vulnerability significantly reduced the effort required for the subsequent password attack.

---

## Root Cause

The root cause was **inconsistent error handling between valid and invalid usernames**.

The application attempted to use a generic authentication failure message, but different internal code paths generated slightly different output.

Conceptually:

```text
Invalid username
      |
      v
Authentication path A
      |
      v
Invalid username or password.


Valid username
Wrong password
      |
      v
Authentication path B
      |
      v
Invalid username or password
```

Even though the difference was only one character, it was sufficient to distinguish the two cases automatically.

---

## Enumeration Oracle

An enumeration oracle is any observable behavior that reveals whether a submitted username is valid.

In this lab:

```text
Input:
candidate username

Observable signal:
presence or absence of "."

Inference:
whether the username exists
```

Therefore:

```text
Single-character response difference
             ↓
Username existence oracle
```

This demonstrates why authentication testing must examine raw HTTP responses rather than relying only on what appears visually in the browser.

---

## Impact

An attacker could enumerate valid user accounts and then perform targeted password attacks.

Potential consequences include:

* disclosure of valid usernames;
* password brute-force attacks;
* password spraying;
* credential stuffing;
* targeted phishing;
* account compromise.

In this case, the enumeration vulnerability was successfully chained with password brute forcing to obtain valid credentials.

---

## Remediation

Authentication failures should be handled consistently.

Recommended controls include:

* Return exactly the same error message for invalid usernames and incorrect passwords.
* Use the same HTTP status code.
* Keep response structure and body length as consistent as reasonably possible.
* Avoid username-dependent redirects.
* Minimize measurable timing differences.
* Centralize authentication error handling to reduce inconsistent code paths.
* Implement rate limiting and brute-force protection.
* Use multi-factor authentication where appropriate.

For example, both cases should return an identical message:

```text
Invalid username or password.
```

It is important that the underlying HTTP responses are also consistent, not just visually similar.

---

## Key Takeaway

When testing authentication mechanisms, very small differences matter.

Do not assume two responses are equivalent simply because they look almost identical.

Compare:

```text
Response body
Length
Status code
Headers
Redirects
Timing
```

Tools such as:

```text
Burp Intruder
Grep - Extract
Comparer
```

can make subtle differences much easier to detect.

The key question is:

> Can any consistent difference in the authentication response reveal whether a username exists?

If the answer is yes, the application may be vulnerable to username enumeration.

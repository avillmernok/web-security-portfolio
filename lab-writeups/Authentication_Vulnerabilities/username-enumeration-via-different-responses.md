# Username Enumeration via Different Responses

## Overview

This lab demonstrates a **username enumeration vulnerability** in a login mechanism.

The application returned slightly different responses depending on whether the submitted username existed.

By comparing response characteristics using Burp Suite Intruder, it was possible to identify a valid username and then brute-force the corresponding password.

This resulted in full account compromise.

---

## Vulnerability

The application accepted credentials through the following endpoint:

```http
POST /login HTTP/2
Host: <lab-host>
Content-Type: application/x-www-form-urlencoded

username=test&password=test
```

Invalid login attempts appeared similar in the browser.

However, closer inspection showed that the server responses were not identical for valid and invalid usernames.

The difference could be detected by comparing the **response length**.

---

## Username Enumeration

The login request was sent to Burp Suite Intruder.

The `username` parameter was selected as the payload position:

```text
username=§test§&password=test
```

A username wordlist was then used to send multiple login attempts while keeping the password constant.

Conceptually:

```text
username=user1&password=test
username=user2&password=test
username=user3&password=test
username=user4&password=test
...
```

Most responses had approximately the same length.

One response was different:

```text
Username: au
Response length: different from the baseline
```

The deviation indicated that the application processed this username differently.

This identified:

```text
Valid username: au
```

---

## Why Response Length Matters

The application did not need to explicitly display:

```text
Username exists
```

for enumeration to be possible.

Any consistent difference between valid and invalid authentication attempts may act as an **enumeration oracle**.

Examples include:

```text
Different error messages
Different response lengths
Different HTTP status codes
Different redirects
Different response times
Different headers
```

In this case, the distinguishing signal was the response body length.

---

## Password Brute Force

After identifying the valid username, the username was fixed:

```text
username=au
```

The Intruder payload position was moved to the password parameter:

```text
username=au&password=§password§
```

A password wordlist was then tested.

Most authentication attempts produced the normal failed-login response.

One request produced a different response, identifying the correct password:

```text
Username: au
Password: 2000
```

The credentials could then be used to successfully authenticate to the application.

---

## Attack Chain

The complete attack consisted of two stages:

```text
Login endpoint
     |
     v
Enumerate usernames
     |
     | compare response lengths
     v
Valid username discovered
     |
     v
Brute-force password
     |
     | compare responses
     v
Valid credentials discovered
     |
     v
Account compromise
```

The username enumeration vulnerability significantly reduced the search space for the password attack.

Instead of brute-forcing combinations of both usernames and passwords, the attacker could first identify a valid account and then focus exclusively on its password.

---

## Root Cause

The root cause was **inconsistent authentication responses**.

The application processed valid and invalid usernames differently and exposed that difference through the HTTP response.

Conceptually:

```text
Invalid username
      |
      v
Authentication logic A
      |
      v
Response length X


Valid username
      |
      v
Authentication logic B
      |
      v
Response length Y
```

Even if the visible error messages appear similar, differences in the underlying HTTP responses can reveal whether an account exists.

---

## Enumeration Oracle

An **oracle** is any observable application behavior that allows an attacker to infer otherwise hidden information.

In this case:

```text
Input:
candidate username

Observable signal:
response length

Inference:
whether the username exists
```

Therefore:

```text
Response length
      ↓
Username existence oracle
```

This type of side channel is particularly useful during authentication testing because very small response differences can become obvious when hundreds of requests are compared automatically.

---

## Impact

An attacker could identify valid user accounts and subsequently perform targeted password attacks.

Potential consequences include:

* disclosure of valid usernames;
* targeted password brute-force attacks;
* credential stuffing;
* password spraying;
* increased effectiveness of phishing attacks;
* account compromise.

The severity increases significantly when the application also lacks effective protections against automated login attempts.

In this case, username enumeration was followed directly by password brute forcing, resulting in successful authentication.

---

## Remediation

Authentication responses should be designed so that valid and invalid usernames cannot be distinguished.

Recommended controls include:

* Return identical error messages for all failed authentication attempts.
* Keep HTTP status codes consistent.
* Avoid meaningful differences in response body length.
* Avoid username-dependent redirects.
* Minimize measurable timing differences.
* Implement rate limiting.
* Detect and restrict automated authentication attempts.
* Consider account lockout or progressive delays where appropriate.
* Use multi-factor authentication for sensitive accounts.

For example, avoid responses such as:

```text
Invalid username
```

and:

```text
Incorrect password
```

Instead, use a generic response:

```text
Invalid username or password
```

However, identical visible text alone is not sufficient.

The underlying HTTP responses should also be as consistent as reasonably possible.

---

## Key Takeaway

When testing authentication mechanisms, do not only read the error message displayed in the browser.

Compare failed login responses using Burp Suite and look for differences in:

```text
Status
Length
Headers
Redirects
Error messages
Response timing
```

A useful testing strategy is:

```text
Keep password constant
        ↓
Enumerate usernames
        ↓
Identify an outlier
        ↓
Fix the valid username
        ↓
Enumerate passwords
```

The key question is:

> Can I distinguish a valid account from an invalid one using any observable property of the server response?

If the answer is yes, the application may be vulnerable to username enumeration.

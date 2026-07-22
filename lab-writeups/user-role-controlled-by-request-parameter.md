# User Role Controlled by Request Parameter

**Lab:** PortSwigger Web Security Academy – User Role Controlled by Request Parameter

**Vulnerability category:** Broken Access Control

**Severity:** High

## Summary

A client-controlled `isAdmin` cookie determined whether the user was granted administrative privileges.

By changing the cookie value from `false` to `true`, an authenticated non-administrative user could access administrative functionality and delete user accounts.

## Discovery

The application's HTTP traffic was inspected using Burp Proxy.

A cookie named `isAdmin` was observed in the request:

```text
isAdmin=false
```

The cookie appeared to control the user's administrative privileges.

After changing its value to:

```text
isAdmin=true
```

the application granted access to the administrative panel.

## Exploitation

The `isAdmin` cookie was modified from `false` to `true` in requests sent to the application.

The modified value allowed access to the administrative panel.

The same modification was applied to the user-deletion request, which allowed the `carlos` account to be deleted successfully.

## Impact

An authenticated attacker with a normal user account could gain unauthorized access to administrative functionality.

In this lab, the attacker could delete existing user accounts.

Depending on the administrative functionality available, similar weaknesses could allow unauthorized modification of users, application data, or configuration.

## Root Cause

The application relied on a client-controlled cookie when making authorization decisions.

The server did not independently verify the user's administrative role before allowing access to sensitive endpoints and actions.

## Remediation

The application should:

- store user roles and privileges in trusted server-side session data;
- enforce server-side authorization checks on every administrative endpoint;
- verify the authenticated user's role before performing sensitive actions;
- deny access by default when the required privilege is not present;
- never rely on client-controlled cookies or request parameters for authorization decisions.

## What I Learned

Authorization decisions must never rely on user-controlled request parameters or cookies.

Modifying a request in Burp affects only that specific request unless the stored cookie value is also changed in the browser.
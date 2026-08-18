# Unprotected Administrative Functionality with an Unpredictable URL

**Lab:** PortSwigger Web Security Academy – Unprotected Admin Functionality with an Unpredictable URL

**Vulnerability category:** Broken Access Control

**Severity:** High

## Summary

A client-side JavaScript snippet revealed the location of an administrative panel.

The administrative endpoint was directly accessible without authentication or authorization checks. An unauthenticated attacker could use the exposed administrative functionality to delete user accounts.

## Discovery

The HTML source code of the application was reviewed for hidden endpoints and client-side functionality.

A JavaScript snippet contained the following administrative path:

```text
/admin-1c6d2t
```

The script used an `isAdmin` variable only to determine whether the administrative link should be displayed in the user interface.

Although the link was hidden from regular users, the administrative endpoint remained visible in the client-side source code and could be accessed directly.

## Exploitation

The disclosed administrative path was opened directly in the browser without logging in.

The application displayed the administrative panel and allowed administrative actions to be performed without verifying the identity or privileges of the user.

The vulnerability was confirmed by deleting the `carlos` user account.

## Impact

An unauthenticated attacker could access sensitive administrative functionality and delete legitimate user accounts.

This could result in:

* unauthorized modification or deletion of application data;
* loss of user access;
* disruption of application services;
* loss of customer trust;
* financial and reputational damage.

The full impact would depend on the other functionality available through the administrative panel.

## Root Cause

The application relied on client-side logic and an unpredictable URL to conceal the administrative interface.

It did not enforce server-side authentication and authorization checks when the administrative endpoint or its functions were accessed.

Hiding an administrative link in the user interface does not prevent users from requesting the underlying endpoint directly.

## Remediation

The application should:

* require authentication before allowing access to the administrative interface;
* enforce server-side authorization checks for every administrative request;
* verify that the authenticated user has the required administrator role;
* deny access by default;
* return `401 Unauthorized` for unauthenticated requests;
* return `403 Forbidden` for authenticated users without sufficient privileges;
* avoid relying on hidden links or unpredictable URLs as security controls;
* avoid exposing unnecessary internal paths in client-side code.

## What I Learned

Client-side restrictions must not be relied upon for access control.

HTML and JavaScript source code should be reviewed during reconnaissance because they may reveal hidden endpoints and application functionality.

However, exposing the administrative path was not the primary vulnerability. The critical security weakness was the absence of server-side authentication and authorization checks.

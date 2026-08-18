# Unprotected Administrative Functionality

**Lab:** PortSwigger Web Security Academy – Unprotected Admin Functionality

**Vulnerability category:** Broken Access Control

**Severity:** High

## Summary

An administrative panel was accessible at the `/administrator-panel` endpoint without requiring authentication or administrative privileges.

An unauthenticated user could access administrative functionality and perform sensitive actions, including deleting existing user accounts.

## Discovery

During the assessment, the `robots.txt` file was reviewed to identify paths that the application did not want search engines to index.

The file contained the following administrative endpoint:

```text
/administrator-panel
```

Navigating directly to this endpoint revealed an accessible administration panel.

## Exploitation

The `/administrator-panel` page was opened without logging in.

The application allowed administrative actions to be performed without verifying the identity or authorization level of the user.

The vulnerability was confirmed by deleting the `carlos` user account.

## Impact

An unauthenticated attacker could gain access to sensitive administrative functionality.

Depending on the available functions, an attacker could:

* delete or modify user accounts;
* access sensitive application data;
* change application settings;
* disrupt the availability and integrity of the application.

In this lab, the vulnerability allowed the deletion of an existing user account.

## Root Cause

The application did not enforce server-side authentication and authorization checks on the administrative endpoint and its associated actions.

The administrative URL was hidden from normal navigation, but knowledge of the endpoint was treated as a security control.

## Remediation

The application should require authentication before allowing access to the administrative interface.

It should also perform server-side authorization checks for every administrative request to verify that the authenticated user has the required administrator role.

Recommended controls include:

* deny access by default;
* enforce role-based access control;
* validate authorization on every sensitive endpoint;
* return an appropriate `401 Unauthorized` or `403 Forbidden` response;
* avoid relying on hidden or unlinked URLs as a security mechanism.

The `robots.txt` file should not be used to protect sensitive resources because it is publicly accessible.

## What I Learned

Hidden or unlinked application endpoints are not protected endpoints.

The `robots.txt` file can reveal useful paths during reconnaissance, but the underlying vulnerability was the absence of server-side access-control checks.

Administrative functions must always be protected by both authentication and authorization.

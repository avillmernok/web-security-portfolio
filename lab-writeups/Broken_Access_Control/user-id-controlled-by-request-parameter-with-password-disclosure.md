# User ID Controlled by Request Parameter with Password Disclosure

**Lab:** PortSwigger Web Security Academy – User ID Controlled by Request Parameter with Password Disclosure

**Vulnerability category:** Broken Access Control

**Vulnerability subtype:** Insecure Direct Object Reference (IDOR) / Horizontal Privilege Escalation

**Additional issue:** Sensitive Credential Disclosure

**Severity:** Critical

## Summary

The `/my-account` endpoint used a client-controlled `id` query parameter to determine which user's account information should be returned.

The application did not verify whether the authenticated user was authorized to access the account referenced by this parameter.

Additionally, the account page included the user's current plaintext password in the HTML response. Although the password field was visually masked by the browser, its real value remained accessible in the raw HTTP response.

By changing the `id` parameter from `wiener` to `administrator`, an authenticated normal user could obtain the administrator's password, log in to the administrator account, and access administrative functionality.

## Discovery

After logging in as `wiener`, the request used to access the account page was inspected in Burp Suite.

The request contained an `id` query parameter that identified the account to be displayed:

`/my-account?id=wiener`

The account page also contained a password input field. Although the browser displayed the password as masked characters, the raw HTML response contained the actual password in the field's `value` attribute.

This indicated that the application returned the current plaintext password to the client.

## Hypothesis

The application might use the client-controlled `id` parameter to select an account without verifying whether the authenticated user was authorized to access it.

If the parameter was changed from `wiener` to `administrator`, the server might return the administrator's account page.

If the administrator's current password was also included in the HTML response, it could be extracted and used to take over the administrator account.

## Exploitation

The request used to access Wiener's account page was captured in Burp Suite and sent to Repeater.

The original query parameter:

`id=wiener`

was changed to:

`id=administrator`

The modified request was sent using Wiener's authenticated session.

The server returned the administrator's account page without verifying that Wiener was authorized to access it.

The HTML response contained the administrator's plaintext password in the password input field.

The disclosed credentials were then used to log in as the administrator.

After successfully authenticating as the administrator, the administrative panel was accessed and the `carlos` user account was deleted.

## Impact

An authenticated attacker with a normal user account could obtain the plaintext password of another user by modifying the `id` query parameter.

In this lab, the attacker could obtain the administrator's password and completely compromise the administrator account.

This allowed access to privileged administrative functionality, including deletion of other user accounts.

Potential consequences include:

- complete compromise of privileged accounts;
- unauthorized access to sensitive user information;
- modification or deletion of user accounts;
- modification of application data or configuration;
- service disruption;
- credential reuse attacks against other services;
- reputational and financial damage.

The demonstrated impact included administrator account takeover and deletion of the `carlos` account.

## Root Cause

The vulnerability resulted from two separate security failures.

### Missing Object-Level Authorization

The application trusted the client-controlled `id` query parameter to select the account that should be returned.

Although the server authenticated the requester as `wiener`, it did not verify whether Wiener was authorized to access the account referenced by `id=administrator`.

### Plaintext Password Disclosure

The application included the user's current plaintext password in the HTML response.

Using an HTML input field with `type="password"` only visually masks the password in the browser. It does not protect the value from anyone who can inspect the page source or HTTP response.

The ability to return the original password may also indicate that passwords are being stored in a reversible or plaintext format.

## Remediation

The application should enforce server-side object-level authorization on every request involving user account data.

For a self-service account page, the account should be determined from the authenticated session rather than from a client-controlled user identifier.

Where an account identifier must be provided, the server should:

- verify that the authenticated user owns the requested account or has an explicitly authorized administrative role;
- deny access by default when authorization cannot be confirmed;
- return `403 Forbidden` for unauthorized object access;
- apply the same authorization checks to read, update, and delete operations;
- avoid treating unpredictable or hidden identifiers as authorization controls.

The application must never return a user's existing password to the client.

Password-handling controls should include:

- storing passwords using a strong, salted, one-way password hashing algorithm;
- never storing plaintext or reversibly encrypted passwords;
- displaying an empty password field on account settings pages;
- requiring the user to enter a new password when changing it;
- requiring the current password or another form of reauthentication before sensitive account changes;
- providing a secure password-reset process using short-lived, single-use tokens;
- invalidating relevant sessions after password changes or account compromise.

## What I Learned

A password input field only hides characters visually in the browser.

If the real password is included in the HTML or HTTP response, the client has already received it and it can be read using browser developer tools or an intercepting proxy.

The `type="password"` attribute is a user-interface feature, not a security control.

Using a query parameter to reference a user account is not automatically a vulnerability. It becomes an IDOR when the server does not verify that the authenticated user is authorized to access the referenced account.

Authentication and authorization remain separate controls:

- **Authentication** determines who sent the request.
- **Authorization** determines whether that user may access the requested account.

In this lab, Wiener was correctly authenticated but was not authorized to access the administrator's account. The application failed to enforce this restriction and additionally disclosed the administrator's plaintext password.
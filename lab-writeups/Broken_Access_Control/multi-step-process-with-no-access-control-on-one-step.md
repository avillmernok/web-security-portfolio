# Multi-Step Process with No Access Control on One Step

## Overview

This lab demonstrates an **access control vulnerability in a multi-step administrative workflow**.

The application used a two-step process to modify user roles:

1. submit the requested role change;
2. confirm the action.

The first step correctly enforced administrator privileges, but the final confirmation step did not perform an equivalent authorization check.

By skipping the protected step and directly invoking the final step, a low-privileged user was able to promote their own account to administrator.

---

## Vulnerability

The administrative role-management functionality used the following endpoint:

```http
POST /admin-roles HTTP/2
Host: <lab-host>
Cookie: session=<admin-session>
Content-Type: application/x-www-form-urlencoded

username=carlos&action=upgrade
```

The server returned:

```http
HTTP/2 200 OK
```

and displayed a confirmation page.

This indicated that the role change was implemented as a multi-step process rather than being executed immediately.

---

## Identifying the Workflow

After confirming the role change in the browser, a second request was sent to the same endpoint:

```http
POST /admin-roles HTTP/2
Host: <lab-host>
Cookie: session=<admin-session>
Content-Type: application/x-www-form-urlencoded

action=upgrade&confirmed=true&username=carlos
```

The server responded with:

```http
HTTP/2 302 Found
Location: /admin
```

The important difference between the two requests was:

```text
confirmed=true
```

This revealed the workflow:

```text
Step 1
POST /admin-roles
username=carlos&action=upgrade
        |
        v
Authorization check
        |
        v
Confirmation page


Step 2
POST /admin-roles
username=carlos&action=upgrade&confirmed=true
        |
        v
Role modification executed
```

---

## Testing the Authorization Boundary

The next question was whether the final step independently verified that the requester had administrator privileges.

To test this, the second request was sent directly using the session cookie of the low-privileged `wiener` account.

The protected first step was skipped entirely.

The request was modified to target the current user:

```http
POST /admin-roles HTTP/2
Host: <lab-host>
Cookie: session=<wiener-session>
Content-Type: application/x-www-form-urlencoded

action=upgrade&confirmed=true&username=wiener
```

The server accepted the request and upgraded the `wiener` account to administrator.

---

## Exploitation

The attack consisted of directly invoking the final step of the administrative workflow:

```text
Low-privileged user
        |
        |
        | Skip protected first step
        |
        v
POST /admin-roles
confirmed=true
username=wiener
        |
        v
Missing authorization check
        |
        v
Role upgraded
```

The application assumed that any request reaching the confirmation stage had already passed the authorization check performed during the previous step.

Because HTTP requests are independent, an attacker could simply construct the final request manually.

---

## Why the Attack Worked

The application treated the multi-step process as though the steps themselves provided a security boundary.

Conceptually, the vulnerable logic behaved like this:

```text
Step 1:
Is the user an administrator?
        |
       Yes
        |
        v
Show confirmation page


Step 2:
confirmed=true?
        |
       Yes
        |
        v
Perform role change
```

The second step failed to verify:

```text
Is the current user actually authorized
to perform this administrative action?
```

The application relied on the assumption that only users who completed step 1 could reach step 2.

That assumption is invalid because an attacker can manually construct and send any HTTP request they know how to reproduce.

---

## Root Cause

The root cause was **missing authorization enforcement on one step of a security-sensitive workflow**.

Authorization was performed only during an earlier stage of the process instead of being enforced at the point where the privileged action was actually executed.

The vulnerability can be summarized as:

```text
Authorization on initial step
        +
No authorization on execution step
        +
Attacker-controlled HTTP requests
        =
Workflow access control bypass
```

---

## Impact

A low-privileged authenticated user could perform an administrative role-management action without authorization.

Potential consequences include:

* vertical privilege escalation;
* unauthorized modification of user accounts;
* execution of administrative functionality;
* modification or deletion of sensitive data;
* compromise of application security boundaries.

In this case, the attacker was able to promote their own account to administrator.

---

## Remediation

Authorization must be enforced independently on every security-sensitive step of a workflow.

Most importantly, the application should verify authorization immediately before executing the privileged operation.

A secure implementation should behave like:

```text
POST /admin-roles
confirmed=true
        |
        v
Authenticate user
        |
        v
Verify administrator privilege
        |
     +--+--+
     |     |
    Yes    No
     |     |
     v     v
Execute  403 Forbidden
action
```

Recommended controls include:

* Perform authorization checks on every privileged request.
* Never assume that completion of an earlier workflow step proves authorization.
* Treat each HTTP request as independently attacker-controlled.
* Enforce authorization inside the business logic that performs the sensitive action.
* Validate workflow state server-side where appropriate.
* Do not rely solely on hidden fields such as `confirmed=true` to prove that previous steps were completed legitimately.

---

## Workflow State vs. Authorization

It is important to distinguish between **workflow state validation** and **authorization**.

For example:

```text
confirmed=true
```

may indicate that the user has confirmed an action, but it does not prove that the user is authorized to perform that action.

Both conditions should be checked independently:

```text
Is the workflow state valid?
        +
Is the current user authorized?
        =
Allow privileged operation
```

A valid workflow state must never replace an authorization check.

---

## Key Takeaway

When testing multi-step functionality, inspect every request in the workflow individually.

Do not only test the first request.

Look for sequences such as:

```text
request
   ↓
review
   ↓
confirmation
   ↓
execution
```

Then attempt to:

* skip earlier steps;
* directly invoke later steps;
* reuse final-step requests with a lower-privileged session;
* modify workflow-control parameters;
* determine whether authorization is checked at the point where the sensitive action is executed.

The key question is:

> Does every security-sensitive step independently verify that the current user is authorized to perform the action?

If a later step relies on authorization performed only during an earlier request, the workflow may be vulnerable to an access control bypass.

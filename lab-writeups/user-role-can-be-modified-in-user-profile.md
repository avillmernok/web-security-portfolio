# User Role Can Be Modified in User Profile

**Lab:** PortSwigger Web Security Academy – User Role Can Be Modified in User Profile

**Vulnerability category:** Broken Access Control / Privilege Escalation

**Related weakness:** Mass Assignment / Over-posting

**Severity:** High

## Summary

The application allowed a normal authenticated user to modify their own server-side role by adding an unexpected `roleid` property to the JSON request used for changing the account email address.

By setting `roleid` to `2`, the user account was permanently upgraded to an administrative role. The attacker could then access the administrative panel and delete other user accounts.

## Discovery

The email address modification function was inspected using Burp Suite.

The original request body contained only the email address:

```json
{
  "email": "test@example.com"
}
```

The server's response also returned information about the current user, including a `roleid` value of `1`.

This suggested that the user profile contained a role property that might be accepted by the profile update endpoint, even though it was not displayed as an editable field in the web interface.

## Hypothesis

The profile update endpoint might automatically accept additional JSON properties supplied by the client.

If the server accepted a user-controlled `roleid` property, a normal user might be able to modify their own authorization level.

## Exploitation

The email modification request was sent to Burp Repeater.

The JSON request body was modified from:

```json
{
  "email": "test@example.com"
}
```

to:

```json
{
  "email": "test@example.com",
  "roleid": 2
}
```

The server accepted the additional property and returned a response showing:

```json
{
  "roleid": 2
}
```

After this modification, the `/admin` endpoint became accessible using the existing session.

The vulnerability was confirmed by deleting the `carlos` user account through the administrative interface.

## Why `roleid` Did Not Need to Be Modified Again

The `roleid` value was not used only for the profile update request.

The vulnerable endpoint permanently stored the new role in the user's server-side profile:

```text
Before:
wiener → roleid 1 → normal user

After:
wiener → roleid 2 → administrator
```

For later requests, the browser only needed to send the normal session cookie:

```http
Cookie: session=[REDACTED]
```

The server used the session identifier to determine that the request belonged to the `wiener` account. It then read the account's stored role from its own server-side data.

Because the stored role was already `2`, the server treated the user as an administrator without requiring `roleid=2` in every subsequent request.

This differs from a client-controlled cookie such as:

```text
isAdmin=true
```

When a cookie is manually modified only in an intercepted request, the change normally applies only to that request. In this lab, however, the authorization level itself was persistently changed on the server.

## Impact

An authenticated user with a normal account could escalate their privileges to administrator.

This allowed unauthorized access to sensitive administrative functionality, including deletion of other user accounts.

Depending on the available administrative functions, an attacker might also be able to:

- modify or disable user accounts;
- access sensitive user information;
- alter application configuration;
- modify or delete application data;
- disrupt application availability;
- compromise the confidentiality and integrity of the application.

Only the deletion of the `carlos` account was directly demonstrated in this lab.

## Root Cause

The profile update endpoint automatically accepted and stored a sensitive property supplied by the client.

The application did not restrict the fields that a normal user was allowed to update.

Instead of explicitly allowing only safe properties such as `email`, the server accepted the additional `roleid` field and applied it to the user's server-side profile.

This is commonly referred to as:

- **Mass assignment**
- **Over-posting**

The application also failed to enforce appropriate authorization controls around changes to privileged account properties.

## Remediation

The application should explicitly define which profile properties a user is allowed to modify.

For an email update endpoint, the server should only process the expected field:

```json
{
  "email": "test@example.com"
}
```

Unexpected or privileged properties such as `roleid`, `isAdmin`, or `permissions` should be ignored or rejected.

Recommended controls include:

- use an allowlist of permitted profile fields;
- never automatically map the complete client request to a user object;
- separate normal profile updates from administrative role-management functions;
- require administrative authorization before changing user roles;
- validate authorization on the server side;
- deny unauthorized role changes by default;
- log and monitor attempts to modify privileged properties;
- add automated tests for mass assignment and privilege escalation.

## What I Learned

A web form does not necessarily show every property that the corresponding API endpoint accepts.

The request and response bodies should both be reviewed for hidden or sensitive object properties.

When an API accepts JSON objects, additional fields can be tested to determine whether the server performs unsafe mass assignment.

A persistent server-side role modification is different from changing a client-controlled value in a single request.

The key distinction is:

```text
Client-side request modification:
The change usually affects only the modified request.

Server-side profile modification:
The change is stored and affects future requests as well.
```

Sensitive properties such as user roles must never be modifiable through a normal profile update endpoint.
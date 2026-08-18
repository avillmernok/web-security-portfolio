# User ID Controlled by Request Parameter with Data Leakage in Redirect

**Lab:** PortSwigger Web Security Academy – User ID Controlled by Request Parameter with Data Leakage in Redirect

**Vulnerability category:** Broken Access Control

**Vulnerability subtype:** Insecure Direct Object Reference (IDOR) / Horizontal Privilege Escalation

**Severity:** High

## Summary

The `/my-account` endpoint used a client-controlled `id` query parameter to determine which user account should be loaded.

When the parameter was changed from `wiener` to `carlos`, the application returned a `302` redirect response. Although the browser was redirected away from Carlos's account, the response body still contained Carlos's sensitive API key.

An authenticated user could therefore access another user's sensitive account information by inspecting the redirect response.

## Discovery

After logging in as `wiener`, the request used to open the account page was inspected in Burp Suite.

The request contained an `id` query parameter:

`/my-account?id=wiener`

This suggested that the application used a client-controlled value to determine which user account should be returned.

## Hypothesis

The server might attempt to prevent unauthorized access by redirecting the requester when the `id` parameter references another user.

However, the requested user's account information might already be included in the redirect response body before the redirect is processed by the browser.

## Exploitation

The request used to access Wiener's account page was captured in Burp Suite and sent to Repeater.

The query parameter was changed from:

`id=wiener`

to:

`id=carlos`

The request was sent using Wiener's authenticated session.

The server returned a `302` redirect response. The response contained a `Location` header that redirected the browser away from Carlos's account.

However, the body of the same `302` response contained Carlos's account page and sensitive API key.

The API key was extracted from the response body and submitted to complete the lab.

## Impact

An authenticated attacker could access sensitive information belonging to other users by modifying the `id` query parameter and inspecting the redirect response.

In this lab, Carlos's API key was exposed.

Depending on the information included in the affected response, similar vulnerabilities could expose:

- personal user information;
- API keys or access tokens;
- account settings;
- private application data;
- other authentication-related information.

Only unauthorized access to Carlos's API key was demonstrated in this lab.

## Root Cause

The application performed the authorization check too late in the request-processing flow.

The server loaded Carlos's account information and included it in the HTTP response before redirecting the unauthorized requester.

Although the redirect prevented the browser from displaying Carlos's account page normally, it did not prevent the client from receiving the sensitive data.

A redirect is a client-side navigation instruction and must not be used as a replacement for server-side authorization.

## Remediation

The application should perform server-side object-level authorization checks before loading or processing the requested user's data.

For a self-service account page, the server should determine the account directly from the authenticated session rather than accepting a client-controlled user identifier.

Where an account identifier must be accepted, the application should:

- verify that the authenticated user is authorized to access the requested account;
- perform the authorization check before retrieving sensitive information;
- return `403 Forbidden` when access is not permitted;
- ensure unauthorized responses contain no sensitive user data;
- avoid relying on redirects as an access-control mechanism;
- apply consistent authorization checks to all read, update, and delete operations.

## What I Learned

A browser redirect does not guarantee that the original response contains no sensitive information.

The full HTTP response must always be inspected, including:

- the status code;
- the `Location` header;
- the response body.

A `302` redirect is only an instruction telling the client to request another URL. The client still receives the original response before following the redirect.

Sensitive information must never be loaded or included in a response before authorization has been successfully verified.
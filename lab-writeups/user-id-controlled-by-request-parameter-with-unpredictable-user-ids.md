# User ID Controlled by Request Parameter with Unpredictable User IDs

**Lab:** PortSwigger Web Security Academy – User ID Controlled by Request Parameter with Unpredictable User IDs

**Vulnerability category:** Broken Access Control

**Vulnerability subtype:** Insecure Direct Object Reference (IDOR) / Horizontal Privilege Escalation

**Severity:** High

## Summary

The `/my-account` endpoint used a client-controlled `id` query parameter to determine which user account should be returned.

Although the application used long and unpredictable user identifiers, another user's identifier could be obtained from publicly accessible blog content.

By replacing Wiener's user identifier with Carlos's identifier, an authenticated user could access Carlos's account page and obtain his sensitive API key.

## Discovery

After logging in as `wiener`, the request used to access the account page was inspected.

The request contained an `id` query parameter with a long, unpredictable identifier:

`/my-account?id=[WIENER_USER_ID]`

Unlike the previous lab, another user's identifier could not be easily guessed from their username.

While reviewing the application's public blog posts, a link associated with the author `carlos` was identified. The link exposed Carlos's user identifier in its URL.

This provided a valid identifier that could be tested against the `/my-account` endpoint.

## Hypothesis

The server might use the client-controlled `id` parameter to select the requested user account without verifying whether the authenticated user was authorized to access it.

If Carlos's identifier was substituted for Wiener's identifier while keeping Wiener's authenticated session, the server might return Carlos's account information.

## Exploitation

The request used to access Wiener's account page was captured in Burp Suite and sent to Repeater.

The original parameter:

`id=[WIENER_USER_ID]`

was replaced with:

`id=[CARLOS_USER_ID]`

The request was sent using Wiener's existing authenticated session.

The server returned Carlos's account page, including his sensitive API key.

This confirmed that the application did not enforce object-level authorization on the requested user account.

## Impact

An authenticated attacker could access sensitive account information belonging to other users if their identifiers could be obtained from another part of the application.

In this lab, the vulnerability exposed Carlos's API key.

Depending on the functionality available through the affected endpoint, similar weaknesses could expose:

- personal user information;
- authentication or API credentials;
- account settings;
- private application data;
- functionality for modifying another user's account.

Only unauthorized access to Carlos's account page and API key was demonstrated in this lab.

## Root Cause

The application trusted a client-controlled `id` query parameter to identify the requested user account.

Although the server correctly authenticated the requester as `wiener`, it did not verify whether Wiener was authorized to access the account referenced by Carlos's identifier.

The use of long, unpredictable identifiers made the values more difficult to guess, but it did not provide access control.

Carlos's identifier was disclosed through a separate public application feature, allowing the missing authorization check to be exploited.

## Remediation

For a self-service account endpoint, the server should derive the requested account directly from the authenticated session rather than accepting an account identifier from the client.

Where access by user identifier is required, the application should:

- perform server-side object-level authorization checks on every request;
- verify that the authenticated user owns the requested account or has an authorized role;
- deny access by default when authorization cannot be confirmed;
- return `403 Forbidden` when an authenticated user requests an unauthorized object;
- apply the same controls to read, update, and delete operations;
- avoid treating unpredictable identifiers as a replacement for authorization;
- minimize unnecessary disclosure of internal object identifiers.

## What I Learned

A query parameter can reference a server-side object such as a user account.

Changing an identifier is not automatically a vulnerability. An IDOR exists when the server accepts the modified identifier without verifying that the authenticated user is authorized to access the referenced object.

Unpredictable identifiers can make enumeration more difficult, but they do not replace server-side authorization.

An identifier that cannot be guessed may still be exposed through another application feature, such as:

- blog posts;
- author profile links;
- comments;
- shared documents;
- API responses.

Authentication and authorization are separate controls:

- **Authentication** determines who sent the request.
- **Authorization** determines whether that user may access the requested object.

In this lab, Wiener was correctly authenticated, but the application failed to verify whether Wiener was authorized to access Carlos's account.
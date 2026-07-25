# User ID Controlled by Request Parameter

**Lab:** PortSwigger Web Security Academy – User ID Controlled by Request Parameter

**Vulnerability category:** Broken Access Control

**Vulnerability subtype:** Insecure Direct Object Reference (IDOR) / Horizontal Privilege Escalation

**Severity:** High

## Summary

The `/my-account` endpoint used a client-controlled `id` query parameter to determine which user's account information should be returned.

By changing the parameter from `wiener` to `carlos`, an authenticated user could access another user's account page and obtain their sensitive API key.

## Discovery

After logging in as the `wiener` user, the request used to open the account page was inspected in Burp Suite.

The request contained the following query parameter:

`id=wiener`

This value appeared to determine which user's account information was returned by the server.

## Hypothesis

The server might select the requested account solely based on the client-controlled `id` query parameter without verifying whether the authenticated user was authorized to access that account.

Changing the parameter from `wiener` to `carlos` might therefore return Carlos's account information while still using Wiener's authenticated session.

## Exploitation

The request used to access the account page was captured in Burp Suite and sent to Repeater.

The query parameter was modified from:

`id=wiener`

to:

`id=carlos`

The request was sent using Wiener's existing session cookie.

The server returned Carlos's account page, including Carlos's sensitive API key.

## Impact

An authenticated attacker could access account information belonging to other users by modifying the `id` query parameter.

In this lab, the vulnerability exposed another user's API key.

If user identifiers are predictable or can be discovered, the vulnerability could potentially allow access to sensitive information belonging to multiple users.

The exact impact would depend on the other information and functionality exposed through the affected endpoint.

## Root Cause

The application trusted a client-controlled query parameter to identify the requested user account.

Although the server authenticated the requester as `wiener`, it did not perform an object-level authorization check to verify that Wiener was permitted to access Carlos's account.

The application therefore confused these two separate questions:

- Who sent the request?
- Is that user permitted to access the requested account?

## Remediation

For a self-service account endpoint, the server should determine the account from the authenticated session instead of accepting a user identifier from the client.

For example, the endpoint should return the account belonging to the current session without requiring an `id` parameter.

Where accessing accounts by identifier is necessary, the server should:

- enforce server-side object-level authorization checks on every request;
- verify that the authenticated user owns the requested account or has an authorized administrative role;
- deny access by default when the relationship between the user and the requested object cannot be verified;
- return `403 Forbidden` when the authenticated user is not permitted to access the object;
- apply the same authorization controls to every endpoint that reads, modifies, or deletes user data;
- avoid treating unpredictable identifiers as a replacement for authorization.

## What I Learned

A query parameter can be used to reference a server-side object such as a user account.

Changing an identifier is not automatically a vulnerability. An IDOR exists when the server accepts the modified identifier without verifying that the authenticated user is authorized to access the referenced object.

Authentication and authorization are separate controls:

- Authentication determines who sent the request.
- Authorization determines whether that user may access the requested object.

In this lab, Wiener was correctly authenticated, but the application failed to verify whether Wiener was authorized to access Carlos's account.
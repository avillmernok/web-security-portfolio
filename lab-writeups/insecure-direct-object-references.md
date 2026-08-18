# Insecure Direct Object References (IDOR)

## Overview

This lab demonstrates an **Insecure Direct Object Reference (IDOR)** vulnerability in a chat transcript download feature.

The application stored chat transcripts as numbered text files and exposed the filename directly in the download URL.

By modifying this object reference, it was possible to access another user's transcript without authorization. The exposed transcript contained sensitive authentication information that could be used to compromise another account.

---

## Vulnerability

The application provided a **View transcript** functionality on the live chat page.

Downloading my own chat transcript generated a request similar to:

```http
GET /download-transcript/3.txt HTTP/2
Host: <lab-host>
Cookie: session=<session>
```

The response contained my own chat history:

```http
HTTP/2 200 OK
Content-Type: text/plain; charset=utf-8
Content-Disposition: attachment; filename="3.txt"
```

The filename:

```text
3.txt
```

acted as a direct reference to a server-side object.

Because the identifier was predictable, I tested whether modifying it would allow access to other transcript files.

---

## Identifying the Object Reference

The relevant part of the request was:

```text
/download-transcript/3.txt
                     ^
              object reference
```

The numeric filename suggested that transcripts might be stored sequentially:

```text
1.txt
2.txt
3.txt
4.txt
...
```

The important question was:

> Does the server verify that the requested transcript belongs to the authenticated user?

To test this, I changed only the object identifier.

---

## Exploitation

The original request:

```http
GET /download-transcript/3.txt HTTP/2
Host: <lab-host>
Cookie: session=<my-session>
```

was modified to:

```http
GET /download-transcript/1.txt HTTP/2
Host: <lab-host>
Cookie: session=<my-session>
```

The server returned:

```http
HTTP/2 200 OK
Content-Type: text/plain; charset=utf-8
Content-Disposition: attachment; filename="1.txt"
```

Instead of rejecting the request, the application returned another user's chat transcript.

This confirmed that the server did not perform an object-level authorization check before returning the requested file.

---

## Sensitive Data Exposure

The unauthorized transcript contained a conversation in which another user disclosed their password.

Example:

```text
User: Ok so my password is <exposed-password>. Is that right?
Support: Yes it is!
```

The exposed credentials could then be used to authenticate as the affected user.

This demonstrated that the IDOR resulted not only in unauthorized data access, but also in **account compromise**.

---

## Why the Attack Worked

The application appeared to perform logic similar to:

```text
Requested file: 1.txt
        |
        v
Does the file exist?
        |
       Yes
        |
        v
Return the file
```

What was missing was an authorization check:

```text
Requested file: 1.txt
        |
        v
Which user owns this transcript?
        |
        v
Does it belong to the current authenticated user?
        |
     No
        |
        v
403 Forbidden
```

The authenticated session was therefore not sufficient to prevent access to another user's resource.

---

## Root Cause

The root cause was **missing object-level authorization**.

The application trusted a user-controlled object reference:

```text
1.txt
```

without verifying whether the authenticated user was authorized to access the corresponding transcript.

The vulnerability can be summarized as:

```text
User-controlled object identifier
        +
Missing ownership / authorization check
        =
IDOR
```

---

## Predictable IDs vs. IDOR

The predictable numeric filenames made exploitation easy:

```text
1.txt
2.txt
3.txt
```

However, predictable identifiers are **not the actual vulnerability**.

The real vulnerability is the absence of authorization.

For example, this could still be vulnerable:

```text
/download-transcript/8f3c91e2-27ab-4a67-b24d-...
```

If an attacker obtained another user's UUID and the server returned the resource without verifying ownership, the application would still suffer from IDOR.

Therefore:

> Predictable identifiers make exploitation easier, but missing object-level authorization is what makes the vulnerability an IDOR.

---

## Impact

An attacker with a valid low-privileged session could access resources belonging to other users.

Potential consequences include:

* disclosure of private user data;
* exposure of confidential files;
* leakage of credentials or authentication tokens;
* horizontal privilege escalation;
* account takeover;
* access to business-sensitive information.

In this case, the exposed transcript contained another user's password, allowing their account to be compromised.

---

## Remediation

The application should enforce authorization for every requested object.

Before returning a transcript, the back-end should verify that the authenticated user owns or is otherwise authorized to access it.

Conceptually:

```text
GET /download-transcript/<id>
        |
        v
Resolve transcript
        |
        v
Check authenticated user
        |
        v
Verify object ownership
     /       \
   Yes        No
    |          |
 Return      403
 object    Forbidden
```

Recommended controls include:

* Enforce object-level authorization on the server side.
* Associate each resource with its owner or access-control policy.
* Verify authorization every time an object is accessed.
* Never rely on hidden or unpredictable identifiers as the primary security mechanism.
* Avoid exposing sensitive information such as passwords in stored transcripts.
* Use indirect references where appropriate, but still enforce authorization independently.

---

## Key Takeaway

When an application exposes an object identifier in a URL or parameter, test whether modifying that identifier allows access to another user's resource.

Common examples include:

```text
/user/123
/invoice/1004
/document/7
/download/42.pdf
/download-transcript/3.txt
```

The key question is:

> Does the server verify that the authenticated user is authorized to access the specific object being requested?

If changing the identifier returns another user's resource without an authorization failure, an IDOR vulnerability is likely present.

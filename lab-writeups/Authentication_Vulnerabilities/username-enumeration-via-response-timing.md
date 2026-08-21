# Username Enumeration via Response Timing

## Overview

This lab demonstrates a **username enumeration vulnerability based on response timing**.

Unlike previous enumeration techniques, the application returned visually identical responses for valid and invalid usernames. However, the server took measurably longer to process requests containing a valid username when a very long password was supplied.

The application also implemented IP-based brute-force protection. This restriction could be bypassed by manipulating the client-controlled `X-Forwarded-For` header.

After identifying a valid username through the timing side channel, the corresponding password was brute-forced, resulting in account compromise.

---

## Vulnerability

The application accepted credentials through:

```http
POST /login HTTP/2
Host: <lab-host>
Content-Type: application/x-www-form-urlencoded

username=test&password=test
```

Failed authentication responses did not provide an obvious username enumeration oracle through:

```text
Error message
Status code
Response length
```

However, the application's processing time differed depending on whether the supplied username existed.

---

## Identifying the Timing Side Channel

The authentication logic effectively behaved differently for invalid and valid usernames.

Conceptually:

```text
Invalid username
      |
      v
Reject authentication early
      |
      v
Fast response
```

For a valid username:

```text
Valid username
      |
      v
Perform password processing
      |
      v
Authentication failure
      |
      v
Slower response
```

With a short password, this timing difference could be too small to distinguish reliably from normal network latency.

A deliberately long password was therefore supplied:

```text
password=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

The long input amplified the additional processing performed for valid usernames, making the timing difference easier to detect.

---

## Brute-Force Protection

After several failed login attempts, the application temporarily blocked further authentication requests from the same IP address.

The response indicated that too many incorrect login attempts had been made.

This initially prevented large-scale username enumeration.

---

## Bypassing the IP-Based Restriction

The application trusted the `X-Forwarded-For` header when determining the client's IP address.

A custom header was added:

```http
X-Forwarded-For: 1
```

The value was changed for each request:

```http
X-Forwarded-For: 1
X-Forwarded-For: 2
X-Forwarded-For: 3
X-Forwarded-For: 4
...
```

Because the application treated these values as different client IP addresses, the per-IP brute-force protection could be bypassed.

The vulnerability was therefore not only in the authentication timing behavior, but also in the application's trust of attacker-controlled proxy metadata.

---

## Burp Intruder Configuration

Two payload positions were configured:

```http
X-Forwarded-For: §1§
```

and:

```text
username=§username§&password=<very-long-password>
```

The attack type was:

```text
Pitchfork
```

### Payload Set 1

Sequential values were used for `X-Forwarded-For`:

```text
1
2
3
4
5
...
```

### Payload Set 2

The provided username wordlist was used.

Pitchfork paired the payloads:

```text
IP 1   + username1
IP 2   + username2
IP 3   + username3
IP 4   + username4
...
```

This ensured that each authentication attempt appeared to originate from a different IP address.

---

## Analyzing Response Timing

The Intruder results were analyzed using the:

```text
Response received
```

column.

Most usernames produced responses within a similar timing range.

A candidate username produced a significantly slower response.

Because timing measurements are inherently noisy, a single slow response should not automatically be considered proof.

Potential sources of noise include:

```text
Network latency
Server load
Request scheduling
Concurrent requests
Connection reuse
```

The important property of a genuine timing side channel is **reproducibility**.

The candidate username was therefore retested and compared with known-invalid usernames to verify that the slower behavior was consistent.

---

## Avoiding False Positives

During testing, individual requests occasionally produced unusually high response times even for usernames that were not valid.

For example:

```text
Candidate A → slow response in one attack
Candidate B → slow response in another attack
```

This demonstrated that:

> A single timing outlier is not sufficient evidence of a valid username.

Reliable timing attacks require repeated measurements.

A useful testing model is:

```text
Candidate username
      |
      v
Repeated measurements
      |
      v
Compare against invalid baseline
      |
      v
Consistent timing difference?
   /         \
 Yes          No
  |            |
Signal        Noise
```

This is an important distinction compared with deterministic response-based enumeration, where a single unique status code or error message may be sufficient.

---

## Password Brute Force

Once the valid username had been identified, it was fixed in the login request.

The password became the second payload position:

```text
username=<valid-username>&password=§password§
```

The `X-Forwarded-For` header was again varied for each request to avoid the IP-based brute-force protection:

```http
X-Forwarded-For: §1§
```

A Pitchfork attack was used with:

```text
Payload 1 → unique X-Forwarded-For values
Payload 2 → password wordlist
```

---

## Identifying the Correct Password

Most incorrect passwords returned:

```http
HTTP/2 200 OK
```

with approximately the same response length as the normal failed-login page.

One request produced:

```http
HTTP/2 302 Found
```

with a much smaller response body.

The successful credential was:

```text
Password: 1234567890
```

The `302` redirect provided a strong authentication-success oracle.

---

## Attack Chain

The complete attack was:

```text
Login endpoint
      |
      v
IP-based brute-force protection
      |
      v
Add attacker-controlled X-Forwarded-For
      |
      v
Rotate apparent client IP
      |
      v
Supply very long password
      |
      v
Enumerate usernames
      |
      | measure response timing
      v
Identify reproducibly slower username
      |
      v
Valid username discovered
      |
      v
Password brute force
      |
      | rotate X-Forwarded-For again
      v
302 redirect identifies correct password
      |
      v
Account compromise
```

---

## Root Causes

This attack required multiple weaknesses.

### 1. Timing discrepancy

Valid and invalid usernames followed different authentication code paths, producing measurable response-time differences.

```text
Valid username
→ additional password processing

Invalid username
→ earlier rejection
```

### 2. Trusting X-Forwarded-For

The application trusted a client-supplied header when enforcing IP-based brute-force restrictions.

```http
X-Forwarded-For: <attacker-controlled-value>
```

This allowed an attacker to impersonate an unlimited number of source IP addresses.

### 3. Insufficient brute-force protection

Once the IP restriction was bypassed, the application permitted enough authentication attempts to brute-force the account password.

---

## Impact

An attacker could remotely identify valid usernames despite the application returning otherwise indistinguishable login responses.

By combining the timing side channel with the `X-Forwarded-For` bypass, an attacker could also perform large numbers of authentication attempts.

Potential consequences include:

* username enumeration;
* targeted password brute forcing;
* password spraying;
* credential stuffing;
* identification of high-value accounts;
* account compromise.

In this lab, the vulnerabilities were successfully chained to obtain valid credentials and authenticate to another user's account.

---

## Remediation

### Prevent username timing differences

Authentication processing should take approximately the same amount of time regardless of whether the username exists.

For example, password verification should use equivalent computational work for both valid and invalid usernames.

Conceptually:

```text
Username exists?
   /        \
 Yes         No
  |           |
Hash real   Hash dummy
password    password
  |           |
  +-----+-----+
        |
        v
Generic authentication failure
```

### Protect trusted proxy headers

Headers such as:

```text
X-Forwarded-For
X-Real-IP
Forwarded
```

should only be trusted when inserted or sanitized by a trusted reverse proxy.

Client-supplied versions should be removed or overwritten at the infrastructure boundary.

### Improve brute-force protection

Recommended controls include:

* rate limiting based on multiple signals;
* account-based throttling;
* progressive delays;
* detection of distributed authentication attacks;
* multi-factor authentication;
* monitoring for anomalous login patterns.

Rate limiting should not rely exclusively on a client IP address.

---

## Key Takeaway

Authentication vulnerabilities are not always visible in the response body.

When conventional enumeration techniques fail, compare:

```text
Response timing
Status codes
Response lengths
Redirect behavior
Headers
```

For timing attacks specifically:

> One slow response is an outlier. A consistently reproducible timing difference is a side channel.

Also inspect whether brute-force protections trust client-controlled headers such as:

```http
X-Forwarded-For
```

A timing vulnerability that appears difficult to exploit may become practical when combined with a weakness in rate limiting.

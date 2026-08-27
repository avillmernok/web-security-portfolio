# Web Security Portfolio

Hands-on web application security portfolio documenting practical vulnerability testing in controlled training environments.

The repository currently contains **20 completed technical write-ups** based primarily on PortSwigger Web Security Academy labs. Each write-up focuses not only on the exploit, but also on the reasoning behind the vulnerability, its root cause, impact, and remediation.

## Current Progress

| Topic | Completed write-ups | Status |
|---|---:|---|
| Broken Access Control | 13 | Completed |
| Authentication Vulnerabilities | 7 | In progress |
| **Total** | **20** | |

## Methodology

The write-ups are structured around a repeatable assessment workflow:

```text
Reconnaissance
      ↓
Identify attack surface
      ↓
Form a hypothesis
      ↓
Manipulate the request
      ↓
Validate the vulnerability
      ↓
Understand the root cause
      ↓
Assess the impact
      ↓
Recommend remediation
```

The goal is to document **why an attack works**, not only the payload that solved the lab.

---

# Broken Access Control

This section covers failures in authorization enforcement, including horizontal and vertical privilege escalation, IDOR, route and method inconsistencies, workflow flaws, and trust in client-controlled request metadata.

| Lab | Main focus |
|---|---|
| [Unprotected admin functionality](lab-writeups/Broken_Access_Control/unprotected-admin-functionality.md) | Missing server-side authorization |
| [Unprotected admin functionality with an unpredictable URL](lab-writeups/Broken_Access_Control/unprotected-admin-functionality-with-unpredictable-url.md) | Hidden endpoint discovery and missing authorization |
| [User role controlled by request parameter](lab-writeups/Broken_Access_Control/user-role-controlled-by-request-parameter.md) | Client-controlled authorization state |
| [User role can be modified in user profile](lab-writeups/Broken_Access_Control/user-role-can-be-modified-in-user-profile.md) | Mass assignment / privilege escalation |
| [User ID controlled by request parameter](lab-writeups/Broken_Access_Control/user-id-controlled-by-request-parameter.md) | Horizontal privilege escalation / IDOR |
| [User ID controlled by request parameter with unpredictable user IDs](lab-writeups/Broken_Access_Control/user-id-controlled-by-request-parameter-with-unpredictable-user-ids.md) | IDOR with non-sequential identifiers |
| [User ID controlled by request parameter with data leakage in redirect](lab-writeups/Broken_Access_Control/user-id-controlled-by-request-parameter-with-data-leakage-in-redirect.md) | Sensitive data exposure before redirect |
| [User ID controlled by request parameter with password disclosure](lab-writeups/Broken_Access_Control/user-id-controlled-by-request-parameter-with-password-disclosure.md) | Horizontal access control failure and credential exposure |
| [Insecure direct object references](lab-writeups/Broken_Access_Control/insecure-direct-object-references.md) | Missing object-level authorization |
| [URL-based access control can be circumvented](lab-writeups/Broken_Access_Control/url-based-access-control-can-be-circumvented.md) | Front-end / back-end URL interpretation mismatch |
| [Method-based access control can be circumvented](lab-writeups/Broken_Access_Control/method-based-access-control-can-be-circumvented.md) | Inconsistent authorization across HTTP methods |
| [Multi-step process with no access control on one step](lab-writeups/Broken_Access_Control/multi-step-process-with-no-access-control-on-one-step.md) | Workflow-level authorization failure |
| [Referer-based access control](lab-writeups/Broken_Access_Control/referer-based-access-control.md) | Authorization based on attacker-controlled request metadata |

### Concepts practiced

- Horizontal privilege escalation
- Vertical privilege escalation
- Insecure Direct Object References (IDOR)
- Object-level authorization
- HTTP method manipulation
- URL-routing inconsistencies
- Multi-step workflow bypasses
- Client-controlled authorization parameters
- Authorization based on HTTP headers

---

# Authentication Vulnerabilities

This section covers weaknesses in login mechanisms, username enumeration, brute-force protection, authentication side channels, and multi-factor authentication.

| Lab | Main focus |
|---|---|
| [Username enumeration via different responses](lab-writeups/Authentication_Vulnerabilities/username-enumeration-via-different-responses.md) | Response differences as a username oracle |
| [Username enumeration via subtly different responses](lab-writeups/Authentication_Vulnerabilities/username-enumeration-via-subtly-different-responses.md) | Single-character authentication response differences |
| [Username enumeration via response timing](lab-writeups/Authentication_Vulnerabilities/username-enumeration-via-response-timing.md) | Timing side channel and `X-Forwarded-For` manipulation |
| [Broken brute-force protection, IP block](lab-writeups/Authentication_Vulnerabilities/broken-brute-force-protection-ip-block.md) | Failed-login counter reset and ordered credential attacks |
| [Username enumeration via account lock](lab-writeups/Authentication_Vulnerabilities/username-enumeration-via-account-lock.md) | Account lockout behavior as an enumeration oracle |
| [Broken brute-force protection, multiple credentials per request](lab-writeups/Authentication_Vulnerabilities/broken-brute-force-protection-multiple-credentials-per-request.md) | JSON type manipulation and request-based rate-limit bypass |
| [2FA simple bypass](lab-writeups/Authentication_Vulnerabilities/2fa-simple-bypass.md) | Missing MFA-state enforcement on protected resources |

### Concepts practiced

- Username enumeration
- Authentication oracles
- Response-length analysis
- Subtle response comparison
- Timing side channels
- Account lockout behavior
- Brute-force protection bypass
- `X-Forwarded-For` manipulation
- JSON input type manipulation
- Multi-factor authentication state

---

# Tools and Techniques

The portfolio primarily documents hands-on testing using **Burp Suite** and manual HTTP analysis.

Techniques demonstrated across the write-ups include:

- Proxy and HTTP history analysis
- Repeater-based request manipulation
- Intruder payload attacks
- Sniper and Pitchfork attack strategies
- Grep - Match and Grep - Extract
- Response length and timing analysis
- HTTP method modification
- Query parameter manipulation
- Cookie manipulation
- Custom HTTP header testing
- JSON structure and data-type manipulation
- Session and authentication-state analysis

---

# Repository Structure

```text
web-security-portfolio/
├── README.md
└── lab-writeups/
    ├── Broken_Access_Control/
    │   └── 13 technical write-ups
    └── Authentication_Vulnerabilities/
        └── 7 technical write-ups
```

New vulnerability classes will be added as the training progresses.

---

# About the Write-Ups

The write-ups generally document:

- vulnerability overview;
- discovery and testing methodology;
- relevant HTTP requests and responses;
- exploitation process;
- root cause;
- security impact;
- remediation recommendations;
- key technical takeaways.

Where useful, they also include failed hypotheses, control tests, and observations made while working in Burp Suite.

---

# Legal and Ethical Scope

All testing documented in this repository is performed in **legal, controlled security-training environments** designed for educational purposes.

The techniques documented here are intended to demonstrate practical web application security testing, technical reasoning, and remediation knowledge.

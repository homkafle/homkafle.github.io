---
layout: post
title: Threat Modeling Cloud Applications With STRIDE
categories: [Product Security]
tags: [appsec, threat-modeling, cloud-security, stride]
fullview: false
description: A practical walkthrough of threat modeling a cloud-based employee recognition platform, with a focus on trust boundaries, abuse cases, and risk-based mitigation.
comments: false
---

### Introduction

Threat modeling is one of the most useful security practices because it forces a team to slow down before the system is already in production.

The point is simple: understand what you are building, decide what can go wrong, and choose what to fix first.

That sounds obvious, but many teams skip it. They rely on scanners, code review, or penetration testing after the design is already mostly fixed. Those activities matter, but they are late signals. Threat modeling gives the team a chance to change the design while change is still cheap.

Cloud applications make this even more important. A modern product is rarely one service and one database. It may include identity providers, API gateways, containers, queues, serverless functions, third-party APIs, managed databases, object storage, secrets, CI/CD pipelines, and monitoring systems. A weakness in one part can become a business problem somewhere else.

HealthEquity's 2024 disclosure is a good reminder of that pattern. In its SEC filing, the company said a business partner's user account was compromised, and that personally identifiable information, in some cases protected health information, was accessed and transferred off the partner's systems. Source: [HealthEquity Form 8-K, July 2, 2024](https://www.sec.gov/Archives/edgar/data/1428336/000142833624000055/hqy-20240702.htm).

The lesson is not that every system has the same risk. The lesson is that trust boundaries matter. Vendor accounts, service identities, APIs, and data flows are part of the system, even when they sit outside the main application code.

This post walks through a practical threat model for a cloud-based employee recognition platform using STRIDE.

### The Application

Imagine an employee recognition platform.

Managers can recognize employees for good work and award points. Employees can view their points and redeem them for gift cards. Admins manage users, organizational hierarchy, gift card catalogs, and vendor integrations.

The application sounds simple, but the security stakes are real.

It handles:

- User identities and sessions.
- Employee profile data.
- Organizational reporting relationships.
- Recognition history.
- Point balances.
- Gift card redemption records.
- Third-party gift card vendor calls.

Points and gift cards make this more than a normal content workflow. If a user can manipulate point balances or redemption flows, the issue becomes fraud. If an attacker can read employee data, the issue becomes privacy. If a manager can recognize people outside their reporting line, the issue becomes authorization and business logic.

### High-Level Architecture

The application has five main areas:

- Web or mobile user interface.
- API gateway.
- Authentication and authorization service.
- Recognition and points services.
- Gift card redemption service.
- Application databases.
- Third-party gift card vendor API.

<center><img src="/assets/media/arcdiagram.png" alt="Architecture diagram of the employee recognition platform" width="500" height="600"></center>
<p><center>Figure: Architecture diagram of the employee recognition platform.</center></p>

Before writing threats, map the trust boundaries.

Important trust boundaries in this system include:

- Browser or mobile client to API gateway.
- API gateway to internal services.
- Internal services to databases.
- Redemption service to third-party vendor.
- Admin workflows to privileged APIs.
- CI/CD and runtime infrastructure to production services.

These boundaries are where assumptions break. They are also where threat modeling should spend time.

### STRIDE as a Thinking Tool

STRIDE is a threat classification model used by Microsoft to organize security questions. Microsoft describes the categories as spoofing, tampering, repudiation, information disclosure, denial of service, and elevation of privilege. Source: [Microsoft Threat Modeling Tool threats](https://learn.microsoft.com/en-us/azure/security/develop/threat-modeling-tool-threats).

For this application, STRIDE helps turn a vague question into concrete review prompts.

| STRIDE Category | Question to Ask |
| --- | --- |
| Spoofing | Can someone pretend to be another user, service, or vendor? |
| Tampering | Can someone change points, recognitions, requests, or vendor responses? |
| Repudiation | Can someone deny a sensitive action because evidence is missing? |
| Information disclosure | Can private employee, account, or gift card data leak? |
| Denial of service | Can a user or attacker disrupt login, points, or redemption? |
| Elevation of privilege | Can a normal user gain manager, admin, or service-level access? |

The value of STRIDE is not the acronym. The value is that it keeps the team from only thinking about the threat they are already comfortable with.

### What Matters Most

A threat model should not become a giant checklist that treats every risk equally.

For this application, the highest-risk areas are:

- Authentication and authorization.
- Manager-to-employee relationship enforcement.
- Points balance integrity.
- Gift card redemption integrity.
- Third-party vendor trust.
- Audit evidence for financial actions.
- Secrets and service credentials.

These areas matter because they connect directly to fraud, privacy, and privilege.

The goal is to understand how a real abuse case would work.

### Abuse Case: Fraudulent Recognition and Redemption

A practical abuse case might look like this:

1. An attacker gets access to a normal employee account.
2. The attacker discovers that the recognition API accepts a target employee ID and point amount.
3. The API checks that the caller is authenticated, but not that the caller is a manager of the target employee.
4. The attacker awards points to their own account or another controlled account.
5. The points service updates the balance.
6. The attacker redeems the points for a gift card.
7. Logs show a valid user session, but not enough business context to prove the recognition was unauthorized.

This is not an exotic attack. It is a common pattern: authentication exists, but object-level authorization is incomplete.

The important question is not "did the user log in?" The important question is "was this user allowed to perform this action on this resource?"

### Area 1: Authentication and Authorization

The authentication service verifies identity. The authorization model decides what the identity can do.

Common threats:

- Stolen credentials used to access employee accounts.
- Session tokens leaked through logs, URLs, browser storage, or insecure cookies.
- Weak token validation allowing forged or modified claims.
- A manager role granted without checking the current organizational hierarchy.
- Admin routes protected only in the UI but not on the API.

Controls:

- Enforce MFA for admins and sensitive roles.
- Use short-lived sessions and secure, HTTP-only cookies where appropriate.
- Validate tokens on every API request.
- Keep role and relationship checks server-side.
- Add object-level authorization tests for manager, employee, admin, and outsider cases.
- Log failed and successful sensitive actions with user, target resource, and decision reason.

The most important design rule: do not let the client decide authorization.

### Area 2: API Gateway

The API gateway is the front door. It can enforce authentication, rate limits, routing, request size limits, and some coarse policy. But it should not be the only place where authorization happens.

Common threats:

- Forged or replayed API requests.
- Overly broad CORS rules.
- Missing rate limits on login, recognition, or redemption endpoints.
- Direct access to internal services that should only be reachable through the gateway.
- Sensitive data returned in error messages.

Controls:

- Require authentication for protected routes.
- Restrict CORS to trusted origins.
- Apply rate limits to login, recognition submission, and redemption flows.
- Block direct public access to internal services.
- Normalize error responses so they do not reveal secrets or internal structure.
- Preserve request IDs across services for investigation.

The gateway is useful, but defense has to continue behind it.

### Area 3: Recognition and Points Services

The recognition service creates the business event. The points service turns that event into a balance. This is where integrity matters.

Common threats:

- A user awards points to themselves.
- A manager awards points outside their reporting line.
- A request changes the point value after approval.
- Race conditions allow double credit or double redemption.
- SQL injection or mass assignment changes point balances.
- A service account has write access to more tables than it needs.

Controls:

- Enforce manager-to-employee authorization in the backend.
- Use server-side point limits and approval rules.
- Store point changes as ledger entries, not only mutable balances.
- Make redemptions idempotent.
- Use database transactions for balance deduction and redemption creation.
- Use parameterized queries and strict input schemas.
- Restrict database write permissions by service.
- Alert on unusual point awards, high redemption velocity, and repeated failed authorization checks.

For financial-like flows, a ledger is usually safer than trusting a single current balance field.

### Area 4: Gift Card Redemption Service

The redemption service is high risk because points become real value.

Common threats:

- A user changes a gift card amount in the request.
- A user redeems the same points twice.
- Vendor API responses are trusted without verification.
- Gift card codes leak into logs, traces, tickets, or analytics.
- Vendor credentials are stored in code or exposed to too many services.
- A third-party outage leaves transactions in an inconsistent state.

Controls:

- Calculate redemption value server-side.
- Deduct points and create redemption records in one atomic flow.
- Use idempotency keys for vendor calls.
- Sign or otherwise verify sensitive vendor callbacks when supported.
- Store gift card codes encrypted and reveal them only to the right user.
- Redact gift card codes and vendor secrets from logs.
- Keep vendor credentials in a secrets manager.
- Build reconciliation jobs for failed, pending, and disputed redemptions.

This is one of the areas where threat modeling should involve product, finance, and support teams, not only engineering. They understand how fraud and disputes will show up in the real business.

### Area 5: Databases and Storage

The database stores the facts the business depends on. It is also where a small permission mistake can become a large data exposure.

Common threats:

- Stolen database credentials.
- Overly broad service account permissions.
- Exposed backups or object storage.
- Sensitive fields stored without encryption.
- Missing audit history for point changes.
- Production data copied into lower environments.

Controls:

- Use separate identities per service.
- Grant least privilege at the database level.
- Encrypt backups and sensitive fields.
- Keep production data out of development environments unless properly approved and protected.
- Audit changes to users, roles, point ledgers, gift cards, and vendor transactions.
- Rotate credentials and store them outside application code.

The database should not have to trust every application path equally.

### Prioritizing the Findings

After listing threats, rank them by business impact and likelihood.

For this application, the top findings would probably be:

1. Missing object-level authorization on recognition and redemption APIs.
2. Mutable point balances without a transaction ledger.
3. Gift card codes or vendor credentials exposed through logs or weak storage.
4. Vendor integration without idempotency or reconciliation.
5. Admin privileges without MFA and strong audit logging.
6. Internal services reachable outside expected network paths.

This is the part that turns threat modeling from a document into engineering work.

A good threat model should create clear actions:

- Add authorization checks.
- Add tests.
- Change service permissions.
- Improve logging.
- Redesign a risky flow.
- Add rate limits.
- Reduce vendor credential scope.
- Define monitoring alerts.

If the output is only a long list of theoretical risks, the team will not use it.

### What Good Looks Like

A strong threat model is practical and alive.

It should include:

- A current architecture diagram.
- Trust boundaries.
- Sensitive data types.
- Critical workflows.
- Abuse cases.
- Key threats.
- Existing controls.
- Gaps and owners.
- Decisions and accepted risks.

It should be revisited when the design changes:

- New API endpoints.
- New user roles.
- New vendor integrations.
- New data types.
- New admin features.
- New deployment architecture.

Threat modeling is not a ceremony. It is a way to keep security connected to design.

### Final Thought

The best threat models do not try to predict every possible attack.

They help teams ask better questions while the design can still change.

For the employee recognition platform, the critical questions are clear:

- Who can award points?
- Who can redeem points?
- Who can change balances?
- What proves the action was legitimate?
- What happens if the vendor call fails?
- What data leaks if one account or service is compromised?

Those questions are simple, but they expose the real risk.

That is the value of threat modeling. It moves security from abstract concern to concrete design choices.

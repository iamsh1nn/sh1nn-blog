---
title: "Authentication vs Authorization: From Internal Mechanics to Deep Testing"
author: sh1nn
date: 2026-07-19
tags:
  - appsec
  - access-control
  - idor
  - bola
  - api-security
  - pentesting
description: "A technical deep dive into authentication state, session and token lifecycles, authorization models, access-control enforcement, IDOR/BOLA, privilege escalation, and practical testing methodology."
draft: false
toc: true
---

# Authentication vs Authorization: From Internal Mechanics to Deep Testing

> This article is intended only for labs, systems you own, or targets you are explicitly authorized to test.

## Introduction

Authentication and authorization are often taught through two short definitions:

```text
Authentication = Who are you?
Authorization  = What are you allowed to do?
```

Those definitions are correct, but they are not enough for serious penetration testing. To test deeply, we need to understand the entire chain:

```text
Credential / proof
        ↓
Identity verification
        ↓
Authentication state
        ↓
Request carries identity
        ↓
Server resolves subject
        ↓
Authorization decision
        ↓
Policy enforcement
        ↓
Business action
```

A vulnerability may exist at any point in that chain. Authentication may be correct while authorization is wrong; the authorization policy may be correct while enforcement is missing; enforcement may exist on one endpoint and be absent on another. A token may be cryptographically valid but carry stale privilege state, and a user may have logged out or been downgraded while an older authentication state remains usable.

Instead of treating vulnerabilities as isolated tricks, I use a unified mental model:

```text
Identity + Authentication State + Role/Attributes/Relationships
+ Resource + Action + Context + Policy + Enforcement
= Security Decision
```

---

## TL;DR

- Authentication and authorization fail independently, often at the _boundary_ between them — not inside either one.
- Both should be modeled as **state machines** (session/token lifecycle, MFA stages) rather than one-time checks, because most real bugs are stale-state or skipped-transition bugs.
- Authorization is a function of `(Subject, Action, Resource, Context, Policy)` — IDOR/BOLA, tenant leaks, and privilege escalation are all instances of one dimension being silently dropped.
- Testing methodology: reconstruct the permission model first, build an authorization matrix, then use a **two-account differential** across user/role/tenant/state boundaries and across every access path (REST, GraphQL, batch, mobile API) — not just the primary endpoint.
- Part IV walks a concrete cross-tenant BOLA case end-to-end, including why swapping sequential IDs for UUIDs does not fix it.

**Contents:** Part I — Authentication (state, sessions, tokens, MFA) · Part II — Authorization (models, IDOR/BOLA, privilege escalation) · Part III — Testing methodology · Part IV — Case study · References

---

## PART I — AUTHENTICATION

### 1. What Authentication Actually Does

Authentication is the process by which a system decides which identity a request represents, and whether there is enough proof to trust that claim. An identity may be a human user, an administrator, an API client, a service account, a machine identity, or a third-party application. Proof may take the form of a password, an OTP, a hardware key, a client certificate, an OAuth assertion, a session cookie, a bearer token, or an API key.

Three concepts must be kept separate: **identity**, **credential**, and **authentication state**. For example:

```text
Identity:              alice@example.com
Credential:             Alice's password
Authentication state:  session=7f91c2...
```

The password is not Alice, and neither is the session — they are mechanisms used to prove or represent Alice's identity at different stages of the authentication lifecycle.

---

### 2. From Credential to Authenticated Request

A basic session-based flow:

```text
Client
  |
  | POST /login
  | email=alice@example.com
  | password=...
  v
Authentication endpoint
  |
  | verify password
  v
Identity store
  |
  | valid
  v
Create authentication state
  |
  | session_id = random value
  | session_id → user_id=123
  v
Set-Cookie: session=abc123
```

A later request carries that state:

```http
GET /account HTTP/2
Cookie: session=abc123
```

The server resolves `abc123` → lookup session → `user_id=123` → `current_user=Alice`. At this point the server knows `Subject = Alice`, but it still hasn't answered what Alice is allowed to do. That is authorization.

---

### 3. What a Session Really Represents

In a server-side session model, the client only stores a `session_id`; the server stores `session_id → authentication state`. For example:

```text
abc123
→ user_id=123
→ created_at=...
→ mfa_complete=true
→ auth_level=full
```

The key implication: whoever possesses a valid session identifier is often treated as the associated identity. That is why session hijacking matters — an attacker doesn't need the username, password, or MFA secret. If they obtain `Cookie: session=abc123` and the server still accepts it, they inherit the authenticated state.

```text
Credential proves identity
        ↓
Server creates authenticated state
        ↓
Session becomes reusable proof
```

Authentication security is therefore not just the login page — it is the full lifecycle of authentication state.

---

### 4. Session Lifecycle

A session moves through several stages:

```text
Created → Used → Privilege changes → Rotated → Expired → Revoked
```

The questions that matter most: does the session ID **rotate** at each trust-boundary transition (login, MFA, privilege elevation), and does it get **revoked** at each trust-boundary exit (logout, password change, account disable)? The full checklist is consolidated in [§35](#35-authentication-testing-methodology).

A common failure pattern: an attacker steals a session, the victim changes their password, and the old session remains valid for 30 days. The credential changed, but the stolen authentication state did not — that is a lifecycle failure, not a password-verification failure.

---

### 5. Session Fixation

Session fixation occurs when an attacker knows or controls the session identifier that a victim continues using after login: the attacker obtains session `S`, the victim authenticates using `S`, the server binds `S` to the victim's identity, and the attacker reuses `S` to inherit the victim's authentication.

The important transition to check is `Pre-auth session → (login) → Post-auth session`: did the session identifier rotate? Failure to rotate across that trust boundary deserves investigation.

---

### 6. Token-Based Authentication and JWT

With server-side sessions, the client holds an opaque identifier and the server holds the authentication state. With self-contained tokens such as JWT, the client holds signed claims and the server holds only the verification key and validation rules:

```json
{
  "sub": "123",
  "role": "member",
  "iat": 1760000000,
  "exp": 1760003600
}
```

The server should validate the signature, issuer, audience, expiration, not-before, expected algorithm, key trust, and any application-specific claims. After validation, `JWT valid → Subject = User 123` — but a valid JWT is not the same as permission to every resource. A JWT establishes identity and claims; authorization is still a separate step.

---

### 7. Authentication State vs Authorization State

Request:

```http
GET /api/invoices/900
Authorization: Bearer <valid-token>
```

Token: `{"sub": "123", "role": "member"}`. Authentication may establish `Subject = User 123, Role claim = member` — but authorization still needs to ask who owns Invoice 900, which tenant owns it, whether this member may read it, whether a relationship grants access, and what state the resource is in. Full flow:

```text
Token
  ↓ verify
Subject
  ↓
Load resource
  ↓
Resolve relationship/context
  ↓
Evaluate policy
  ↓
ALLOW / DENY
```

This boundary is fundamental.

---

### 8. Authentication Should Be Modeled as a State Machine

Example MFA flow:

```text
Unauthenticated
      ↓ correct password
Password Verified
      ↓ correct OTP
MFA Verified
      ↓
Fully Authenticated
```

If a sensitive endpoint checks only `password_verified == true` instead of `fully_authenticated == true`, MFA may be bypassed. Testing an authentication state machine means enumerating its states and transitions, then asking whether any state can be skipped, replayed, reused for another identity, or left alive past a security-context change (full checklist: [§35](#35-authentication-testing-methodology)). Many authentication bugs are really state-machine bugs.

---

### 9. Password Reset Is a Separate Authentication Protocol

Flow: `Request reset → Identify account → Issue reset proof → Deliver proof → Validate proof → Set new credential`. Each stage has its own trust boundary, and the reset token itself is worth treating like a second credential: unpredictable, account-bound, expiring, single-use, and not leakable through URLs, referrers, or logs.

Password reset is not a secondary feature — it is an alternate authentication route. If it is weaker than the primary login path, an attacker will choose the weaker route.

---

### 10. MFA Testing Means Testing the Flow, Not Only the OTP

Don't test only whether the OTP can be brute-forced. Model the flow instead: `Password verified → MFA pending → MFA complete`. The two questions that catch most real bugs: can the **MFA-pending** state reach a post-MFA endpoint directly, and is any recovery/reset path weaker than the primary MFA mechanism? (full checklist: [§35](#35-authentication-testing-methodology))

---

## PART II — AUTHORIZATION

### 11. Authorization Is a Policy Decision

Authorization can be modeled as `Decision = f(Subject, Action, Resource, Context, Policy)`. For example:

```text
Subject:  Alice
Action:   DELETE
Resource: Project 500
Context:  Organization 20
Policy:   Only project owner or organization admin can delete
```

The result is `ALLOW` or `DENY`. Broken access control exists whenever the expected decision does not match the actual decision.

---

### 12. Subject, Action, Resource, Context

**Subject** — who is acting: a user, admin, service account, API client, or anonymous principal.

**Action** — what operation: `READ, CREATE, UPDATE, DELETE, APPROVE, EXPORT, INVITE, REFUND, TRANSFER`.

**Resource** — what object: a user, invoice, project, file, organization, payment, or API key.

**Context** — what surrounding conditions matter: tenant, ownership, relationship, time, resource state, subscription plan, environment, authentication assurance.

Authorization failures often occur because the implementation ignores one of these dimensions.

---

### 13. RBAC

Role-Based Access Control maps `User → Role → Permission`. For example, `Alice → Admin → user.delete` and `Bob → Member → project.read`. But if "Member has `project.read`" — does that mean read _every_ project, or only projects in the same tenant? RBAC alone often cannot express ownership and relationship boundaries.

---

### 14. ABAC

Attribute-Based Access Control may evaluate a rule such as:

```text
ALLOW if user.department == document.department
AND user.clearance >= document.classification
```

Attributes can belong to the subject, resource, action, or environment. Failures typically come from stale attributes, client-controlled attributes, missing attributes, policy conflicts, or default-allow behavior. ABAC is expressive but can become inconsistent when policy logic is fragmented across the codebase.

---

### 15. Relationship-Based Authorization

In a SaaS environment: `Alice → (member_of) → Organization A → (owns) → Project X → (contains) → Document D`. Authorization may be derived as "Alice can read Document D because Alice is a member of Organization A, which owns Project X, which contains D." This is why `role=member` alone is not enough — the relationship between subject and resource is often what actually determines access.

---

### 16. Where Is Authorization Enforced?

Policy may be enforced at the API gateway, middleware, route guard, controller, service layer, ORM/query layer, or database row-level security. A common failure: `/admin/*` uses `require_admin()`, but a new endpoint `/api/users/{id}/disable` is added without that middleware. The result is that the UI is protected, Route A is protected, but Route B is unprotected — **inconsistent enforcement**.

Testing alternative endpoints, old APIs, mobile APIs, or GraphQL is not random guessing; the goal is to identify **authorization differentials between code paths**.

---

### 17. IDOR/BOLA: The Real Root Cause

Request:

```http
GET /api/orders/1001
Cookie: session=USER_A
```

Backend:

```python
order = db.get_order(request.id)
return order
```

Authentication may be completely correct — `session USER_A → User A` — but what's missing is:

```python
if not can_read(current_user, order):
    deny()
```

Root cause: the client controls the resource reference, the server loads the resource, and the server fails to verify the subject-resource relationship. That is the essence of IDOR/BOLA. It is not about the ID being numeric or guessable — a UUID does not replace authorization.

---

### 18. Horizontal Privilege Escalation

Boundary: `User A vs User B`. If A owns Invoice 100 and B owns Invoice 200, the expected results are `A → 100 = ALLOW` and `A → 200 = DENY`. Test with `Session A + Object A` and `Session A + Object B`; if Object B is returned, that's an object-level authorization failure. The key experimental idea: authentication context stays constant while the resource changes — that isolates the authorization decision.

---

### 19. Vertical Privilege Escalation

Boundary: `Member → Manager → Admin`. Request:

```http
POST /api/admin/users/42/disable
Cookie: session=MEMBER
```

Expected: admin only. If the backend checks only `authenticated == true` instead of `role == admin`, the root cause is an authentication check used where an authorization check was required.

---

### 20. Tenant Boundaries

```text
Organization A          Organization B
├── Alice               ├── Charlie
└── Bob                 └── David
```

Alice and Charlie may both have `role=member`, but `Alice → Org A resource` may be allowed while `Alice → Org B resource` must be denied. If the server checks only `if user.role == "member": allow()` and ignores the tenant relationship, cross-tenant access may occur. Correct policy usually needs an explicit check such as `user.tenant_id == resource.tenant_id`, or an equivalent relationship check.

---

### 21. Object-Level vs Function-Level Authorization

**Object-level:** the user may invoke the function, but not on this object — e.g. a user may view invoices, but only their own.

**Function-level:** the user should not be able to invoke the function at all — e.g. only admins may disable users.

These are different questions: can I invoke this function, and if yes, can I invoke it on this object?

---

## PART III — HOW I TEST ACCESS CONTROL

### 22. Do Not Start by Changing IDs — Reconstruct the Permission Model

Before tampering with requests, map out the identities, authentication states, roles, tenants, resources, relationships, actions, and lifecycle states involved. Example SaaS model:

```text
Organization                    Organization
├── Owner                       └── Projects
├── Admin                           └── Documents
└── Member
```

Expected policy:

```text
Member: read projects in own organization; edit own documents
Admin:  manage members; edit all documents in own organization
Owner:  delete organization
```

Without understanding expected behavior, it is easy to confuse intended behavior with a vulnerability, or to miss a real boundary.

---

### 23. Build an Authorization Matrix

| Subject  | Resource   | Action | Expected    |
| -------- | ---------- | ------ | ----------- |
| Member A | Project A  | Read   | Allow       |
| Member A | Project A  | Delete | Deny        |
| Member A | Project B  | Read   | Deny        |
| Admin A  | Project A  | Delete | Maybe/Allow |
| Admin A  | Org B User | Modify | Deny        |

Testing becomes a comparison of expected vs. actual, rather than random tampering.

---

### 24. Two-Account Methodology

At minimum you need Account A and Account B, preferably with role differences (Member, Admin, Owner). Flow:

```text
1. Capture legitimate request A
2. Identify authentication context
3. Identify object reference
4. Obtain equivalent object B
5. Keep authentication A
6. Replace only resource/context
7. Compare expected vs actual
```

Example: request `GET /api/projects/101` with `Cookie: session=A`, then `GET /api/projects/202` with the same `Cookie: session=A`. The experimental property is that authentication stays constant while the resource changes.

---

### 25. Cross-User, Cross-Role, Cross-Tenant, Cross-State

- **Cross-user:** User A → User B's object.
- **Cross-role:** Member → Admin function.
- **Cross-tenant:** Tenant A user → Tenant B resource.
- **Cross-state:** removed user → former resource; downgraded admin → old admin action.

Each boundary tests a different policy dimension.

---

### 26. Where Can Object and Context References Appear?

Object and context references can hide in more places than the obvious path parameter:

- **Path:** `/api/users/1001`
- **Query:** `?user_id=1001`
- **JSON body:** `{"owner_id": 1001}`
- **Header:** `X-Organization-ID: 10`
- **Cookie:** `active_org=10`
- **GraphQL variables:** `{"projectId": "1001"}`
- **Nested path:** `/orgs/10/projects/50/files/900`

For each, ask which client-controlled value selects the identity context, tenant, resource, or action.

---

### 27. Test Resource × Action

A resource supports many actions: `READ, UPDATE, DELETE, EXPORT, SHARE, APPROVE, TRANSFER`. It's common for `GET /api/documents/123` to correctly return `403`, while `POST /api/documents/123/export` returns `200` — authorization may be correct for READ and broken for EXPORT. So test resource × action, not resource alone.

---

### 28. Alternative Endpoints as Authorization Differentials

The same business action may exist through `/api/v1/users/42`, `/api/v2/users/42`, `/mobile/users/42`, `/graphql`, or `/internal-api/users/42` — different paths that may run different code:

```text
Same business action → Different path → Different enforcement → Potential differential
```

This is why old APIs, mobile APIs, and GraphQL are worth comparing against the primary endpoint.

---

### 29. HTTP Method Differentials

`GET`, `PATCH`, and `DELETE` on the same `/api/project/42` may route to entirely different code — `GET` through middleware A, `PATCH` through service B, `DELETE` through legacy controller C. Method testing is not random verb-switching; the question is whether a different method means a different code path with different policy enforcement.

---

### 30. Batch Endpoints

A single-object path such as `GET /api/documents/101` may enforce ownership correctly, while a batch path such as `POST /api/documents/batch` with `{"ids": [101, 202]}` may simply execute `SELECT * WHERE id IN (...)` and forget to check owner, tenant, or relationship. The pattern: security exists in the single-object path but disappears in the aggregation path.

---

### 31. Indirect Access Paths

A resource may be exposed through preview, download, export, search, notification, share link, attachment, audit log, or history — not just its primary endpoint. For example, `GET /documents/123` correctly denies, but `GET /documents/123/preview` allows. Therefore: authorization testing is not endpoint testing, it's **business-object access testing**.

---

### 32. Frontend Restrictions Are Not Security Enforcement

The frontend may hide a button, disable a field, redirect, or remove a menu item:

```javascript
if (user.role !== "admin") {
  hideDeleteButton();
}
```

The backend still needs its own check:

```python
if not can_delete(current_user, target):
    deny()
```

Frontend restrictions are UX. The attacker controls the requests directly, so trusted enforcement must exist server-side.

---

### 33. Mass Assignment Can Be an Authorization Problem

A normal update sends `{"name": "Alice"}`, but the server object also has `role` and `tenant_id`. If a client sends `{"name": "Alice", "role": "admin"}` and the framework blindly binds that field, the key question is whether this user is authorized to modify the `role` field at all. Sensitive properties to watch for: `role, permissions, owner_id, tenant_id, is_admin, status, plan, verified`.

---

### 34. Authorization Lifecycle and Stale State

Suppose a JWT contains `{"sub": "123", "role": "admin", "exp": "..."}`, but the user is downgraded so the database now says `role=member` while the token still says `role=admin`. The question: which source does authorization trust? If the server trusts stale claims, privilege revocation may be delayed. The same question applies to removed members, suspended users, transferred objects, revoked invitations, and changed tenant membership. Authorization is not static.

---

### 35. Authentication Testing Methodology

I model authentication as a state machine:

```text
Unauthenticated
   ↓ password
Password Verified
   ↓ MFA
Fully Authenticated
   ↓ sensitive action
Reauthentication Required
```

This is the consolidated checklist referenced throughout Part I, organized by what it targets:

| Target                  | Question                                                                                                                  |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Transition skipping     | Can I reach a later state without completing the transition (e.g. call a post-MFA endpoint while only password-verified)? |
| Proof reuse             | Can I replay proof, or reuse proof issued for one identity/session on another?                                            |
| Expiry                  | Can I use expired proof (session, token, reset link)?                                                                     |
| State/endpoint mismatch | Does any endpoint accept a weaker state than it should?                                                                   |
| Rotation                | Does the session/token rotate at login, MFA, and privilege elevation?                                                     |
| Revocation              | Does logout, password change, account disable, or privilege downgrade revoke the old state?                               |
| Secondary paths         | Is password reset or account recovery weaker than the primary login/MFA path?                                             |

---

### 36. Authorization Testing Methodology

My high-level flow:

```text
1. Identify subjects
2. Identify authentication contexts
3. Identify roles
4. Identify tenants
5. Identify resources
6. Identify relationships
7. Identify actions
8. Define expected policy
9. Capture legitimate requests
10. Cross boundaries
11. Test alternative enforcement paths
12. Test lifecycle/state changes
13. Validate impact minimally
```

---

## PART IV — TECHNICAL CASE STUDY

### 37. SaaS Project Management Application

Assume the following tenancy and policy:

```text
Organization A                       Organization B
├── Alice — Member                   ├── Charlie — Member
├── Bob — Admin                      └── Project 200
└── Project 100                          └── Document 900
    ├── Document 500
    └── Document 501
```

```text
Member:            read projects in own organization; edit documents they own
Admin:              manage members; edit any document in own organization
Cross-organization: always deny unless explicitly shared
```

---

### 38. Alice Authenticates

```http
POST /login
Content-Type: application/json

{
  "email": "alice@example.test",
  "password": "..."
}
```

Response: `Set-Cookie: session=A1B2C3; HttpOnly; Secure`. Server state: `A1B2C3 → user_id=10 → auth_level=full`.

A follow-up request confirms the resolution:

```http
GET /api/me
Cookie: session=A1B2C3
```

`session → Alice → Organization A → role Member`. Authentication is complete.

---

### 39. Correct Authorization Flow

Alice requests:

```http
GET /api/projects/100/documents/500
Cookie: session=A1B2C3
```

The server should: (1) authenticate the session → Alice; (2) load Document 500; (3) resolve the relationship — Document 500 belongs to Project 100, which belongs to Organization A; (4) resolve subject context — Alice belongs to Organization A; (5) evaluate policy — same tenant, member may read; (6) **ALLOW**.

---

### 40. Broken Authorization Flow

Alice changes the request:

```http
GET /api/projects/200/documents/900
Cookie: session=A1B2C3
```

Backend:

```python
document = get_document(document_id)

if current_user.is_authenticated:
    return document
```

Authentication is correct. Authorization is missing — the server never evaluates that Document 900 belongs to Project 200, which belongs to Organization B, while Alice belongs to Organization A. Expected: **DENY**. Actual: **ALLOW**. This is cross-tenant BOLA.

---

### 41. Why UUIDs Do Not Fix the Problem

Replacing `900` with `d9f1a6a4-...` does not change the policy. If the server still does:

```python
document = get_document(uuid)
return document
```

authorization is still missing — security is relying on identifier secrecy instead of explicit permission enforcement.

---

### 42. Correct Remediation

One approach is to scope the query directly:

```python
document = db.documents.find_one(
    id=document_id,
    organization_id=current_user.organization_id
)
```

Another is an explicit policy layer:

```python
document = get_document(document_id)

if not policy.can_read_document(current_user, document):
    deny()
```

The important property: every relevant access path must enforce equivalent policy — read, download, preview, export, search, batch, GraphQL, and the mobile API alike.

In practice, checking this manually across every path doesn't scale — the two-account differential from [§24](#24-two-account-methodology) is what I automate: a small script (or a Burp extension such as Autorize) replays a captured request set under a second identity and diffs the responses, so a regression in any single access path shows up immediately instead of requiring a full manual re-pass.

---

### 43. Final Mental Model

When I see `GET /api/resource/123` with `Cookie: session=...`, I do not ask only "can I change 123?" I ask:

```text
1. Which identity does this session/token represent?
2. Through which flow was that identity established?
3. What authentication state is active?
4. What exactly is Resource 123?
5. Who owns it and which tenant contains it?
6. What action is being requested?
7. What is the expected policy?
8. Where is that policy enforced?
9. Are there alternate code paths that bypass enforcement?
10. Can lifecycle changes make authentication or authorization state stale?
```

The core idea:

> **Authentication establishes the subject. Authorization decides what that subject may do. Access control is the consistent enforcement of that decision across every relevant code path and every relevant system state.**

A strong pentester does not merely look for parameters whose IDs can be changed. They reconstruct the identity model, the authentication state model, the permission model, the resource relationships, and the enforcement architecture — then look for places where those models stop matching each other. That is where authentication and authorization vulnerabilities usually emerge.

---

## References

- OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization, API5:2023 Broken Function Level Authorization
- MITRE CWE-639 — Authorization Bypass Through User-Controlled Key
- MITRE CWE-384 — Session Fixation
- OWASP Testing Guide — Testing for Broken Authentication and Session Management
- PortSwigger Web Security Academy — Access Control Vulnerabilities, Authentication

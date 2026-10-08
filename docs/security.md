\# Security Architecture



\## Overview



Security is implemented as a layered system rather than relying on a single authorization check.



The platform handles authenticated users, role-based administrative access, user-specific training data, private business documents, assessment results, and AI-powered processing.



The primary security boundaries are:



```text id="7o3fca"

&#x20;                   Client

&#x20;                     │

&#x20;                     ▼

&#x20;              Authentication

&#x20;                     │

&#x20;                     ▼

&#x20;             Application Layer

&#x20;                     │

&#x20;         ┌───────────┴───────────┐

&#x20;         │                       │

&#x20;         ▼                       ▼

&#x20;   Authorization             Validation

&#x20;         │                       │

&#x20;         └───────────┬───────────┘

&#x20;                     ▼

&#x20;                PostgreSQL

&#x20;                     │

&#x20;               Row-Level Security

&#x20;                     │

&#x20;         ┌───────────┴───────────┐

&#x20;         ▼                       ▼

&#x20;   Private Storage          Application Data

&#x20;                     │

&#x20;                     ▼

&#x20;             Trusted Server Jobs

&#x20;                     │

&#x20;                     ▼

&#x20;                AI Providers

```



The architecture follows a defense-in-depth approach so that a failure at one application boundary does not automatically result in unrestricted data access.



\---



\## 1. Authentication



The application uses Supabase Authentication for user identity and session management.



Authenticated sessions are used to determine the current application user before protected operations are performed.



Conceptually:



```text id="4r2q3v"

User

&#x20;│

&#x20;▼

Login

&#x20;│

&#x20;▼

Supabase Auth

&#x20;│

&#x20;▼

Authenticated Session

&#x20;│

&#x20;▼

Application Authorization

```



Authentication establishes \*\*who the user is\*\*.



Authorization determines \*\*what that user is allowed to access\*\*.



These concerns are intentionally kept separate.



\---



\## 2. Role-Based Access Control



The platform contains different application roles with different capabilities.



Examples include:



\* Learners

\* Managers

\* Administrators



The application uses role information to control access to protected workflows and administrative interfaces.



```text id="2eqfmg"

Authenticated User

&#x20;       │

&#x20;       ▼

&#x20;    User Role

&#x20;       │

&#x20;  ┌────┼────┐

&#x20;  ▼    ▼    ▼

Learner Manager Admin

```



Role checks are enforced server-side for protected operations.



The client interface may hide unavailable functionality, but UI visibility is not treated as a security boundary.



\---



\## 3. PostgreSQL Row-Level Security



PostgreSQL Row-Level Security (RLS) is the primary data-isolation boundary.



Application-level filtering alone is insufficient because an accidentally omitted query condition could expose records belonging to another user.



RLS provides an additional database-level authorization layer.



```text id="z1bdt8"

Application Query

&#x20;     │

&#x20;     ▼

&#x20;PostgreSQL

&#x20;     │

&#x20;     ▼

&#x20;  RLS Policy

&#x20;     │

&#x20;┌────┴────┐

&#x20;│         │

Allowed   Denied

&#x20;│         │

&#x20;▼         ▼

Data      No Access

```



The database therefore participates directly in authorization decisions.



\---



\## 4. User Data Isolation



User-owned records are protected so that one learner cannot access another learner's information.



This includes data such as:



\* Training records

\* Assessment attempts

\* Interview sessions

\* Progression data

\* Study information

\* Other user-specific application records



Security tests explicitly attempt cross-user access to verify the isolation boundary.



Example:



```text id="qj8j5d"

Learner A

&#x20;   │

&#x20;   ├── Own Data → Allowed

&#x20;   │

&#x20;   └── Learner B Data → Denied

```



This protects against both accidental query omissions and deliberate ID manipulation.



\---



\## 5. IDOR Protection



The application treats object identifiers supplied by a client as untrusted input.



Changing an identifier in a request must not grant access to another user's record.



For example:



```text id="r7z7y4"

Request

&#x20; │

&#x20; ├── record\_id = A

&#x20; │

&#x20; ▼

Authorization Check

&#x20; │

&#x20; ▼

RLS / Server Validation

&#x20; │

&#x20; ├── User owns A → Continue

&#x20; │

&#x20; └── User does not own A → Reject

```



Automated tests cover these cross-object access scenarios.



This is especially important for endpoints that operate on records referenced by URL parameters, request bodies, or form submissions.



\---



\## 6. Server-Side Authorization



Security-sensitive operations are performed on trusted server-side paths.



The browser is not treated as a trusted authority for:



\* User roles

\* Record ownership

\* Assessment progression

\* Administrative operations

\* Privileged database access

\* AI provider credentials



The server verifies authorization before performing protected operations.



\---



\## 7. Privileged Database Access



The Supabase service-role credential bypasses normal RLS protections and is therefore treated as a highly privileged secret.



Service-role access is restricted to trusted server-side operations such as:



\* Controlled provisioning

\* Background jobs

\* Server-side configuration operations

\* Signed upload workflows

\* Trusted scheduled processing



It is never exposed to browser clients.



Where possible, ordinary authenticated session clients are used so database authorization remains active.



\---



\## 8. Private Document Storage



Business documents are stored in private object storage.



The application does not rely on publicly accessible document URLs.



Controlled access is provided through server-authorized workflows and signed access mechanisms where appropriate.



```text id="f8a5py"

User

&#x20;│

&#x20;▼

Authorized Application Request

&#x20;│

&#x20;▼

Server Validation

&#x20;│

&#x20;▼

Signed / Controlled Access

&#x20;│

&#x20;▼

Private Document

```



This reduces the risk of sensitive documents being accessible simply by knowing their storage path.



\---



\## 9. RAG Security Boundary



The vector database is treated as a search system, not an authorization system.



Semantic similarity alone must never determine which information a user can access.



Before retrieved content reaches an AI model, retrieval is constrained by application context and relevant metadata.



```text id="7v3pqo"

User Query

&#x20;   │

&#x20;   ▼

Authorization Context

&#x20;   │

&#x20;   ▼

Record / Scope Filter

&#x20;   │

&#x20;   ▼

Vector Search

&#x20;   │

&#x20;   ▼

Relevant Allowed Passages

&#x20;   │

&#x20;   ▼

AI Model

```



This is an important distinction:



> \*\*RAG determines relevant information; authorization determines permitted information.\*\*



\---



\## 10. AI Provider Security



External AI providers are accessed from trusted server-side application code.



API credentials are stored as environment variables and are not embedded into client-side bundles.



```text id="xj6g9q"

Browser

&#x20;  │

&#x20;  │ No AI credentials

&#x20;  ▼

Next.js Server

&#x20;  │

&#x20;  │ Secure API credential

&#x20;  ▼

AI Provider

```



This prevents users from directly extracting provider credentials from browser code.



\---



\## 11. Structured AI Output Validation



Language model output is treated as untrusted external data.



When an AI workflow expects structured output, the response is validated against an application-defined schema before being persisted or passed into downstream workflows.



```text id="n3u7up"

AI Output

&#x20;   │

&#x20;   ▼

Parse

&#x20;   │

&#x20;   ▼

Schema Validation

&#x20;   │

&#x20;┌──┴───────┐

&#x20;│          │

Valid     Invalid

&#x20;│          │

&#x20;▼          ▼

Continue   Reject / Retry

```



This reduces the risk of malformed or unexpected model output corrupting structured application data.



\---



\## 12. Input Validation



Application inputs are validated before reaching business logic or external integrations.



Validation is particularly important for:



\* Form submissions

\* API parameters

\* IDs

\* Structured AI output

\* Document metadata

\* Assessment data

\* Configuration values



Schema validation is used where structured input contracts are required.



Input validation complements database constraints and authorization rather than replacing them.



\---



\## 13. Background Job Security



Background processing introduces a separate security boundary because scheduled workers may require elevated permissions.



Workers therefore operate only on trusted server-side paths.



The processing model includes:



\* Controlled job creation

\* Job ownership/context

\* Short-lived leases

\* Bounded retries

\* Server-side credentials

\* Explicit failure states



Scheduled processing endpoints are protected so they cannot be freely triggered as privileged operations by arbitrary clients.



\---



\## 14. Database-Enforced Business Rules



Some critical business rules are enforced at the database boundary rather than only inside application code.



This prevents alternate application paths from bypassing important data-integrity rules.



For example:



```text id="z5z4h3"

Application

&#x20;   │

&#x20;   ▼

Database Write

&#x20;   │

&#x20;   ▼

Database Rule / Trigger

&#x20;   │

&#x20;   ▼

Validated Persistent State

```



Application-level logic mirrors relevant rules for predictable behavior and user-facing calculations.



Integration tests verify that the application and database implementations remain consistent.



\---



\## 15. Security Testing



Security is tested as part of the automated test suite rather than relying only on manual review.



\### Authentication Tests



Verify:



\* Protected routes require authentication.

\* Unauthenticated users cannot access protected workflows.

\* Sessions are handled correctly.



\### Role Tests



Verify:



\* Users cannot access administrative workflows without the appropriate role.

\* Role-specific operations remain restricted.



\### RLS Tests



Verify:



\* Users can access their permitted records.

\* Users cannot access other users' records.

\* Unauthorized database operations are rejected.



\### IDOR Tests



Attempt to manipulate record identifiers and verify that authorization still prevents cross-user access.



\### Storage Tests



Verify that private document access follows the intended authorization model.



\### AI Failure Tests



Verify that invalid or failed external AI responses do not bypass application validation or corrupt persistent state.



\---



\## 16. Security Model



The overall model can be summarized as:



```text id="h0e7yw"

&#x20;                TRUST BOUNDARIES



&#x20;       ┌──────────────────────────┐

&#x20;       │          Browser         │

&#x20;       │      Untrusted Client    │

&#x20;       └────────────┬─────────────┘

&#x20;                    │

&#x20;             Authentication

&#x20;                    │

&#x20;                    ▼

&#x20;       ┌──────────────────────────┐

&#x20;       │      Next.js Server      │

&#x20;       │ Authorization + Domain   │

&#x20;       │ Validation + Workflows   │

&#x20;       └────────────┬─────────────┘

&#x20;                    │

&#x20;                    ▼

&#x20;       ┌──────────────────────────┐

&#x20;       │       PostgreSQL         │

&#x20;       │       RLS Boundary       │

&#x20;       └────────────┬─────────────┘

&#x20;                    │

&#x20;            ┌───────┴────────┐

&#x20;            ▼                ▼

&#x20;     Private Storage      Application Data

&#x20;            │

&#x20;            ▼

&#x20;      Trusted AI Jobs

&#x20;            │

&#x20;            ▼

&#x20;      External AI APIs

```



Each layer has a different responsibility.



No single layer is expected to provide complete protection on its own.



\---



\## Security Principles



The platform follows these principles:



1\. \*\*Authentication identifies the user; authorization determines access.\*\*

2\. \*\*The client is never treated as a trusted authority.\*\*

3\. \*\*RLS is the primary database isolation boundary.\*\*

4\. \*\*Semantic search is never used as an authorization mechanism.\*\*

5\. \*\*Privileged service credentials remain server-side.\*\*

6\. \*\*Sensitive documents remain in private storage.\*\*

7\. \*\*AI output is treated as untrusted input.\*\*

8\. \*\*Security-sensitive workflow decisions remain outside the LLM.\*\*

9\. \*\*Cross-user access is tested explicitly.\*\*

10\. \*\*Critical data-integrity rules are enforced at the strongest practical boundary.\*\*



\---



\## Summary



The security architecture combines application-level authorization with database-level enforcement and controlled access to external services.



Key mechanisms include:



\* Supabase Authentication

\* Role-based access control

\* PostgreSQL Row-Level Security

\* Cross-user / IDOR protection

\* Private document storage

\* Server-side privileged operations

\* Secure AI provider integration

\* Structured AI output validation

\* Input validation

\* Protected background jobs

\* Database-enforced business rules

\* Automated security and failure testing



The objective is to ensure that AI functionality remains inside clearly defined security and authorization boundaries rather than becoming an implicit source of access control.




\# Testing Strategy



\## Overview



The platform uses multiple testing layers to validate application logic, database behavior, authorization, AI integrations, and complete user workflows.



The goal is not only to verify that features work under normal conditions, but also to verify that the system behaves safely when:



\* Users attempt unauthorized access

\* External AI providers fail

\* Documents are unusually large

\* Background jobs are interrupted

\* AI responses do not match the expected schema

\* Multiple application layers interact



The testing strategy is organized into:



```text

&#x20;                Test Pyramid



&#x20;             ┌───────────────┐

&#x20;             │     E2E       │

&#x20;             │ User Flows    │

&#x20;             └───────┬───────┘

&#x20;                     │

&#x20;            ┌────────┴────────┐

&#x20;            │   Integration   │

&#x20;            │ DB / AI / RAG   │

&#x20;            └────────┬────────┘

&#x20;                     │

&#x20;             ┌───────┴───────┐

&#x20;             │      Unit      │

&#x20;             │ Domain / Core  │

&#x20;             └───────────────┘

```



Security and failure tests cut across all three layers.



\---



\## 1. Unit Testing



Unit tests validate deterministic application logic independently of external infrastructure.



The unit suite covers areas such as:



\* Domain rules

\* Authentication helpers

\* Validation

\* Assessment scoring

\* Interview state transitions

\* Progression logic

\* RAG utilities

\* Chunking

\* Metadata generation

\* Retry calculations

\* MCQ logic

\* Notification behavior

\* Document-processing helpers

\* Structured AI response validation



Unit tests use deterministic inputs and avoid unnecessary external dependencies.



This keeps feedback fast and makes failures easier to diagnose.



\---



\## 2. Integration Testing



Integration tests verify interactions between application modules and infrastructure.



Areas covered include:



\### Database



\* CRUD operations

\* Relationships

\* Transactions

\* Database functions

\* Triggers

\* Constraints

\* RLS policies



\### Authentication



\* Session handling

\* Role behavior

\* Protected operations



\### Document Processing



\* Document registration

\* Page persistence

\* Extraction workflows

\* Processing state

\* Resumable processing



\### RAG



\* Chunk persistence

\* Vector indexing

\* Retrieval

\* Retrieval consumers

\* Metadata filtering



\### AI Workflows



\* Question generation

\* Interview processing

\* Structured output validation

\* Provider failure handling



Integration testing provides confidence that independently tested modules also behave correctly when connected.



\---



\## 3. End-to-End Testing



Playwright is used for browser-level end-to-end testing.



E2E tests exercise complete user workflows rather than isolated functions.



Examples include:



\### Administrative Workflows



```text

Login

&#x20; │

&#x20; ▼

Admin Dashboard

&#x20; │

&#x20; ▼

Manage Records

&#x20; │

&#x20; ▼

Create / Update Data

&#x20; │

&#x20; ▼

Verify Result

```



\### Learner Workflows



```text

Login

&#x20; │

&#x20; ▼

Learner Dashboard

&#x20; │

&#x20; ├── Study

&#x20; ├── Quiz

&#x20; ├── Exam

&#x20; ├── Interview

&#x20; └── Progression

```



\### AI Workflows



```text

Input

&#x20; │

&#x20; ▼

Application Workflow

&#x20; │

&#x20; ▼

AI Processing

&#x20; │

&#x20; ▼

Validated Result

&#x20; │

&#x20; ▼

Persisted Application State

```



E2E tests help detect issues that would not be visible through unit or integration tests alone.



\---



\## 4. Authorization Testing



Authorization is treated as a first-class testing concern.



The platform includes tests for:



\* Role restrictions

\* Record ownership

\* Row-Level Security

\* Cross-user access

\* IDOR scenarios

\* Protected routes

\* Administrative operations



A representative security test is:



```text

User A

&#x20; │

&#x20; ├── Request User A Data → Allowed

&#x20; │

&#x20; └── Request User B Data → Denied

```



These tests are important because a successful frontend interaction does not prove that the underlying data boundary is secure.



\---



\## 5. RLS Testing



PostgreSQL Row-Level Security policies are tested directly through integration tests.



The tests verify both positive and negative cases.



\### Positive



A user can access records they are authorized to access.



\### Negative



A user cannot access records belonging to another user.



This approach helps catch regressions where a database policy is accidentally made too permissive or a new table is introduced without the required isolation.



\---



\## 6. IDOR Testing



The application explicitly tests identifier manipulation.



For example:



```text

Normal Request

&#x20;    │

&#x20;    ▼

/resource/OWN\_ID

&#x20;    │

&#x20;    ▼

Allowed



Manipulated Request

&#x20;    │

&#x20;    ▼

/resource/OTHER\_ID

&#x20;    │

&#x20;    ▼

Rejected

```



These tests ensure that knowing or guessing another object's identifier does not grant access to it.



\---



\## 7. AI Provider Failure Testing



External AI services are treated as unreliable dependencies.



The test suite therefore includes simulated provider failures.



Examples include:



\* API errors

\* Invalid responses

\* Missing fields

\* Unexpected structured output

\* Timeouts

\* Provider unavailability



The application is expected to handle these failures without silently writing invalid application state.



\---



\## 8. Fake Providers



External integrations are abstracted so that deterministic fake providers can be used during testing.



Conceptually:



```text

&#x20;                AI Interface

&#x20;                    │

&#x20;         ┌──────────┴──────────┐

&#x20;         ▼                     ▼

&#x20;Production Provider       Fake Provider

&#x20;         │                     │

&#x20;         ▼                     ▼

&#x20;   External API          Deterministic Tests

```



This provides several advantages:



\* Tests do not require production API credentials.

\* Tests remain deterministic.

\* Provider failures can be simulated.

\* Test execution is faster.

\* Business logic remains independent of provider implementation details.



The same abstraction principle is applied to other external integrations where practical.



\---



\## 9. Structured Output Validation



AI-generated structured data is validated before it enters application workflows.



Testing covers:



```text

Valid AI Output

&#x20;     │

&#x20;     ▼

Schema Validation

&#x20;     │

&#x20;     ▼

Accepted

```



and:



```text

Invalid AI Output

&#x20;     │

&#x20;     ▼

Schema Validation

&#x20;     │

&#x20;     ▼

Rejected / Retry

```



This is particularly important for AI-generated assessment questions and interview evaluation data.



\---



\## 10. Document Processing Tests



The document pipeline is tested against both normal and difficult inputs.



Test scenarios include:



\* Normal documents

\* Large documents

\* Oversized uploads

\* Multi-page processing

\* Partial processing

\* Failed processing units

\* Retry behavior

\* Resumable processing

\* Invalid extracted content

\* Vector indexing failures



The objective is to ensure that document size or temporary external failures do not force the entire pipeline to restart.



\---



\## 11. Background Job Testing



Background workflows are tested for reliability characteristics.



The test suite covers scenarios such as:



```text

Pending Job

&#x20;   │

&#x20;   ▼

Claim

&#x20;   │

&#x20;   ▼

Process

&#x20;   │

&#x20;┌──┴───────┐

&#x20;▼          ▼

Success    Failure

&#x20;│          │

&#x20;▼          ▼

Complete   Retry

```



Tests verify that:



\* Jobs can be claimed.

\* Expired work can be recovered.

\* Failed jobs can retry.

\* Maximum attempts are respected.

\* Successful work reaches a completed state.

\* Duplicate processing is minimized through idempotent behavior.



\---



\## 12. Database Integrity Testing



Database behavior is tested beyond basic CRUD operations.



The integration suite covers:



\* Foreign-key relationships

\* Constraints

\* Triggers

\* Derived business rules

\* Eligibility calculations

\* Role policies

\* RLS behavior

\* Transactional workflows



Critical business rules are tested at both the application and database levels where the rule exists in both layers.



\---



\## 13. Regression Testing



New functionality should not break existing workflows.



Regression coverage includes major areas such as:



\* Authentication

\* Role-based access

\* Administrative management

\* Learner progression

\* Assessments

\* Interviews

\* Document processing

\* AI question generation

\* RAG consumers

\* Notifications

\* Appointments

\* Name-card generation



The test suite can therefore act as a safety net while individual modules continue to evolve.



\---



\## 14. Testing AI Without Testing the Model



The objective is not to test whether a language model is universally "correct."



Instead, the application tests the boundaries around the model.



```text

&#x20;             AI Model

&#x20;                │

&#x20;      ┌─────────┴─────────┐

&#x20;      │                   │

&#x20;  Input Contract      Output Contract

&#x20;      │                   │

&#x20;      ▼                   ▼

&#x20;   Validation          Validation

&#x20;      │                   │

&#x20;      └─────────┬─────────┘

&#x20;                ▼

&#x20;         Application Logic

```



Testing focuses on questions such as:



\* Was the correct context supplied?

\* Was the correct user/record scope applied?

\* Was the output structurally valid?

\* What happens when the provider fails?

\* Does invalid output get rejected?

\* Does the workflow remain consistent?



This treats the AI model as an external probabilistic dependency inside a deterministic application architecture.



\---



\## 15. Test Configuration



The project separates testing environments and configurations where appropriate.



The repository includes dedicated configuration for:



\* Unit tests

\* Integration tests

\* End-to-end tests

\* Local database testing



This allows the appropriate level of infrastructure to be used for each test type.



\---



\## 16. CI Validation



The project includes automated checks suitable for continuous integration.



Typical validation includes:



```text

Install Dependencies

&#x20;      │

&#x20;      ▼

Lint

&#x20;      │

&#x20;      ▼

Typecheck

&#x20;      │

&#x20;      ▼

Unit Tests

&#x20;      │

&#x20;      ▼

Integration Tests

&#x20;      │

&#x20;      ▼

E2E Tests

```



The exact CI execution environment may vary, but the repository is structured so that correctness checks can be automated rather than relying entirely on manual verification.



\---



\## 17. Local Development and Reproducibility



The project includes a Docker-backed local Supabase environment.



This allows database-dependent testing to be performed against a reproducible local stack rather than requiring direct access to production infrastructure.



A simplified workflow is:



```text

Clone Repository

&#x20;     │

&#x20;     ▼

Install Dependencies

&#x20;     │

&#x20;     ▼

Start Local Services

&#x20;     │

&#x20;     ▼

Initialize Database

&#x20;     │

&#x20;     ▼

Run Tests

```



External AI integrations can be replaced with test providers where live provider access is unnecessary.



\---



\## Testing Principles



The project follows these testing principles:



1\. \*\*Test deterministic business logic at the unit level.\*\*

2\. \*\*Test infrastructure interactions through integration tests.\*\*

3\. \*\*Test complete user journeys through E2E tests.\*\*

4\. \*\*Treat authorization as a testable system boundary.\*\*

5\. \*\*Test both positive and negative security cases.\*\*

6\. \*\*Simulate external AI failures instead of assuming providers are always available.\*\*

7\. \*\*Validate AI-generated structured output before persistence.\*\*

8\. \*\*Test large and failure-prone document workflows explicitly.\*\*

9\. \*\*Use reproducible local infrastructure for database-dependent tests.\*\*

10\. \*\*Test the boundaries around AI rather than pretending model behavior is deterministic.\*\*



\---



\## Summary



The testing architecture reflects the fact that the platform is both a conventional full-stack application and an AI-enabled system.



Coverage spans:



\* TypeScript domain logic

\* PostgreSQL

\* Supabase

\* Authentication

\* RLS

\* IDOR protection

\* AI integrations

\* RAG

\* Document processing

\* Background jobs

\* Structured validation

\* Administrative workflows

\* Learner workflows

\* Browser-level user journeys



The result is a test strategy focused not only on feature correctness, but also on \*\*security, reliability, failure recovery, and safe integration of probabilistic AI components into deterministic application workflows\*\*.




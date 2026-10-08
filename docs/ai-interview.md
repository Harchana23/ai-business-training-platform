\# AI Readiness Interview



\## Overview



The platform includes a conversational AI assessment that evaluates a learner's readiness based on business-specific knowledge and predefined assessment requirements.



The system combines:



\* Conversational AI

\* RAG-based knowledge retrieval

\* Session state management

\* Structured answer evaluation

\* Business-specific assessment criteria

\* Persistent interview sessions

\* Final readiness evaluation



The interview is intentionally designed as a \*\*controlled conversational workflow\*\* rather than a fully autonomous agent.



The application controls the assessment flow, while the language model handles natural-language interaction and knowledge-grounded evaluation.



\---



\## Architecture



```text

&#x20;                   LEARNER

&#x20;                      │

&#x20;                      ▼

&#x20;             Interview Interface

&#x20;                      │

&#x20;                      ▼

&#x20;             Next.js Server Layer

&#x20;                      │

&#x20;            ┌─────────┴─────────┐

&#x20;            │                   │

&#x20;            ▼                   ▼

&#x20;       Session State          RAG Search

&#x20;            │                   │

&#x20;            │                   ▼

&#x20;            │              Relevant Context

&#x20;            │                   │

&#x20;            └─────────┬─────────┘

&#x20;                      │

&#x20;                      ▼

&#x20;               AI Interview Layer

&#x20;                      │

&#x20;                      ▼

&#x20;                Anthropic API

&#x20;                      │

&#x20;                      ▼

&#x20;             Structured Evaluation

&#x20;                      │

&#x20;                      ▼

&#x20;             Persisted Interview Data

&#x20;                      │

&#x20;                      ▼

&#x20;             Final Readiness Result

```



\---



\## 1. Controlled Conversational Flow



The interview follows a predefined assessment sequence based on business requirements.



The application determines:



\* Which stage of the interview the learner is currently in

\* Which question should be presented

\* Whether an answer has been submitted

\* When the assessment is complete

\* When the final readiness evaluation should be generated



The language model does \*\*not\*\* independently determine the next assessment question.



This separation provides predictable assessment behavior while still allowing the learner to interact naturally with the AI.



```text

Assessment Definition

&#x20;       │

&#x20;       ▼

Predefined Question Flow

&#x20;       │

&#x20;       ▼

Learner Answer

&#x20;       │

&#x20;       ▼

AI Evaluation

&#x20;       │

&#x20;       ▼

Persist Result

&#x20;       │

&#x20;       ▼

Next Defined Question

```



\---



\## 2. Session State



Each interview maintains persistent session state.



The state allows the system to recover the learner's position and maintain context throughout the assessment.



Conceptually:



```text

Interview Session

&#x20;     │

&#x20;     ├── learner

&#x20;     ├── current stage

&#x20;     ├── conversation history

&#x20;     ├── submitted answers

&#x20;     ├── evaluation data

&#x20;     └── completion state

```



The server remains responsible for validating the current session before accepting new responses.



This prevents the client interface from becoming the source of truth for interview progression.



\---



\## 3. RAG Integration



The interview can retrieve relevant business knowledge before generating or evaluating responses.



```text

Learner Answer

&#x20;     │

&#x20;     ▼

Determine Relevant Context

&#x20;     │

&#x20;     ▼

RAG Retrieval

&#x20;     │

&#x20;     ▼

Relevant Business Passages

&#x20;     │

&#x20;     ▼

AI Evaluation / Response

```



The retrieved context provides the AI with information specific to the business or training material being assessed.



This reduces reliance on the model's general knowledge and allows the assessment to evaluate responses against the relevant source material.



The RAG layer is described separately in:



`docs/rag-pipeline.md`



\---



\## 4. Grounded Evaluation



The AI evaluation layer receives structured assessment context together with relevant retrieved knowledge.



A simplified flow is:



```text

Interview State

&#x20;     │

&#x20;     ├── Current Question

&#x20;     ├── Learner Answer

&#x20;     ├── Assessment Criteria

&#x20;     └── Retrieved Knowledge

&#x20;               │

&#x20;               ▼

&#x20;        Evaluation Prompt

&#x20;               │

&#x20;               ▼

&#x20;          AI Provider

&#x20;               │

&#x20;               ▼

&#x20;      Structured Evaluation

```



The evaluation can consider factors such as:



\* Relevance of the response

\* Understanding of the subject

\* Alignment with expected business knowledge

\* Completeness of the answer

\* Evidence contained in the response



The exact evaluation criteria are application-defined rather than left entirely to open-ended model judgment.



\---



\## 5. Structured AI Output



AI responses are converted into structured application data before being used by downstream workflows.



Schema validation is used to ensure that model output conforms to the expected structure.



```text

AI Response

&#x20;   │

&#x20;   ▼

Parse Structured Output

&#x20;   │

&#x20;   ▼

Schema Validation

&#x20;   │

&#x20;┌──┴──────────┐

&#x20;│             │

Valid        Invalid

&#x20;│             │

&#x20;▼             ▼

Persist      Reject / Retry

```



This prevents malformed model output from directly corrupting application state.



Structured validation is also used elsewhere in the platform, including AI-generated question workflows.



\---



\## 6. Conversation and Application Responsibilities



The system deliberately separates responsibilities between deterministic application logic and probabilistic AI behavior.



\### Application controls



\* Authentication

\* Authorization

\* Interview ownership

\* Session state

\* Assessment sequence

\* Question progression

\* Persistence

\* Completion rules

\* Final workflow state

\* Validation



\### AI handles



\* Natural-language interaction

\* Understanding learner responses

\* Knowledge-grounded interpretation

\* Answer evaluation

\* Conversational response generation



This boundary is important because the AI should not be responsible for security-sensitive or workflow-critical decisions that can be enforced deterministically.



\---



\## 7. Interview Lifecycle



A typical interview follows this lifecycle:



```text

Start Interview

&#x20;     │

&#x20;     ▼

Create / Resume Session

&#x20;     │

&#x20;     ▼

Load Assessment State

&#x20;     │

&#x20;     ▼

Present Defined Question

&#x20;     │

&#x20;     ▼

Learner Responds

&#x20;     │

&#x20;     ▼

Retrieve Relevant Knowledge

&#x20;     │

&#x20;     ▼

Evaluate Response

&#x20;     │

&#x20;     ▼

Persist Conversation + Evaluation

&#x20;     │

&#x20;     ▼

Advance Interview State

&#x20;     │

&#x20;     ├───────────────┐

&#x20;     │               │

&#x20;  More Questions   Complete

&#x20;     │               │

&#x20;     ▼               ▼

Next Question    Final Evaluation

&#x20;                     │

&#x20;                     ▼

&#x20;               Readiness Result

```



\---



\## 8. Resume and State Recovery



Interview state is persisted rather than existing only in browser memory.



This allows the application to distinguish between:



\* A new interview

\* An interview currently in progress

\* A resumed interview

\* A completed interview



Persisted state also allows server-side validation of the learner's current position.



The browser therefore acts as an interface rather than the authoritative source of assessment progression.



\---



\## 9. Final Readiness Evaluation



After the defined assessment flow is completed, the system produces a final readiness result based on the accumulated interview data and evaluation information.



```text

Individual Evaluations

&#x20;       │

&#x20;       ▼

Aggregate Assessment Context

&#x20;       │

&#x20;       ▼

Final AI Evaluation

&#x20;       │

&#x20;       ▼

Structured Result

&#x20;       │

&#x20;       ▼

Persisted Readiness State

```



The result becomes part of the broader learner progression workflow.



The platform can then use this state alongside other independently derived conditions rather than treating the AI output as the only source of progression logic.



\---



\## 10. Why the Interview Is Not an Autonomous Agent



Although the system uses conversational AI, it is intentionally \*\*not implemented as a fully autonomous agent\*\*.



An autonomous agent might independently:



\* Choose its own objectives

\* Dynamically plan actions

\* Select tools

\* Determine the next task

\* Modify its strategy during execution



This interview instead uses deterministic application orchestration.



```text

Application Workflow

&#x20;       │

&#x20;       ├── Defines assessment sequence

&#x20;       ├── Maintains session state

&#x20;       ├── Controls permissions

&#x20;       └── Determines completion

&#x20;               │

&#x20;               ▼

&#x20;          AI Component

&#x20;               │

&#x20;       ├── Understands answers

&#x20;       ├── Retrieves context

&#x20;       └── Evaluates responses

```



This architecture makes the assessment easier to test, reason about, and constrain.



It also prevents an LLM from accidentally bypassing application-level workflow rules.



\---



\## 11. Failure Handling



AI services are external dependencies and can fail.



The application therefore treats provider failures as expected operational conditions rather than assuming every model request succeeds.



The integration layer supports provider abstraction and test doubles.



This allows automated tests to simulate scenarios such as:



\* Provider errors

\* Invalid AI output

\* Timeout conditions

\* Unexpected response formats

\* Retrieval failures



The application can then verify that failures do not silently corrupt interview state.



\---



\## 12. Security



Interview data is protected using the same authorization model as the rest of the platform.



Important boundaries include:



\### Authentication



Only authenticated users can access their interview sessions.



\### Authorization



Users can only access interviews associated with their permitted records.



\### Database-Level Isolation



PostgreSQL Row-Level Security provides an additional authorization boundary.



\### Server-Side AI Credentials



AI provider credentials remain on the server and are never exposed to the browser.



\### Controlled Retrieval



RAG retrieval is constrained to the appropriate business context before information is supplied to the model.



\---



\## 13. Testing Strategy



The interview system is covered by multiple testing layers.



\### Unit Tests



Core interview logic is tested independently, including:



\* State transitions

\* Interview progression

\* Evaluation parsing

\* Validation

\* Completion logic



\### Integration Tests



Integration tests verify:



\* Interview persistence

\* AI integration adapters

\* RAG consumers

\* Session handling

\* Database interactions



\### End-to-End Tests



Browser-level tests cover the learner's complete interview workflow.



\### Failure Tests



AI provider failures and malformed model responses are explicitly tested to ensure the application fails safely.



\### Authorization Tests



Cross-user access attempts are tested to ensure one learner cannot access another learner's interview data.



\---



\## Engineering Principles



The interview architecture follows several principles:



1\. \*\*Deterministic workflow, probabilistic AI.\*\*

2\. \*\*The application owns state; the model does not.\*\*

3\. \*\*RAG provides evidence; it does not provide authorization.\*\*

4\. \*\*AI output is validated before entering application workflows.\*\*

5\. \*\*External AI providers are isolated behind integration adapters.\*\*

6\. \*\*Security-sensitive decisions remain outside the language model.\*\*

7\. \*\*Persistent state enables recovery and reliable testing.\*\*

8\. \*\*Conversational UX does not require autonomous agent behavior.\*\*



\---



\## Summary



The AI readiness interview demonstrates how conversational AI can be integrated into a production-oriented full-stack workflow without delegating core application control to an LLM.



The design combines:



\* Next.js server-side application logic

\* Persistent interview sessions

\* Predefined assessment orchestration

\* RAG-based business knowledge retrieval

\* Anthropic-powered conversational interaction

\* Structured AI evaluation

\* Schema validation

\* PostgreSQL persistence

\* Row-Level Security

\* Failure isolation

\* Automated unit, integration, E2E, and authorization testing



The result is a controlled conversational assessment system that combines the flexibility of an LLM with deterministic application-level workflow and security boundaries.




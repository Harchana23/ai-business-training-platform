\# Document Processing Architecture



\## Overview



The platform processes business documents and transforms their contents into structured, searchable knowledge for downstream training and AI workflows.



The document-processing system was designed around an important constraint: \*\*large documents cannot reliably be processed as a single serverless request\*\*.



Instead, processing is incremental, resumable, and persisted at each stage.



```text

Upload

&#x20; │

&#x20; ▼

Document Registration

&#x20; │

&#x20; ▼

Page Detection

&#x20; │

&#x20; ▼

Processing Queue

&#x20; │

&#x20; ▼

Page-Level Extraction

&#x20; │

&#x20; ▼

Persisted Transcripts

&#x20; │

&#x20; ▼

Chunking

&#x20; │

&#x20; ▼

Vector Indexing

&#x20; │

&#x20; ▼

Knowledge Available

```



\---



\## 1. Upload Architecture



Documents are uploaded directly to private object storage using controlled access rather than passing large files through the application server.



```text

Browser

&#x20;  │

&#x20;  │ Signed Upload

&#x20;  ▼

Private Object Storage

&#x20;  │

&#x20;  ▼

Document Metadata

&#x20;  │

&#x20;  ▼

PostgreSQL

```



The database stores metadata and processing state separately from the binary document.



This allows application workflows to operate on document records without repeatedly transferring the original file.



\---



\## 2. Why Page-Level Processing?



A single large document-processing request creates several problems in a serverless environment:



\* Execution time limits

\* Request payload limits

\* Memory consumption

\* AI provider latency

\* Partial failures

\* Difficulty recovering from a failed request



Instead, the system divides processing into bounded units.



```text

Large PDF

&#x20;   │

&#x20;   ▼

┌──────────────┐

│ Pages 1–N    │

├──────────────┤

│ Pages N–M    │

├──────────────┤

│ Pages M–X    │

├──────────────┤

│     ...      │

└──────────────┘

```



Each unit can be processed and persisted independently.



If one unit fails, the entire document does not need to be processed again.



\---



\## 3. Processing State



Each document moves through explicit processing stages.



A simplified lifecycle is:



```text

Registered

&#x20;   │

&#x20;   ▼

Queued

&#x20;   │

&#x20;   ▼

Processing

&#x20;   │

&#x20;   ▼

Extracted

&#x20;   │

&#x20;   ▼

Indexed

&#x20;   │

&#x20;   ▼

Ready

```



Failure paths are also represented explicitly:



```text

Processing

&#x20;   │

&#x20;   ▼

Failure

&#x20;   │

&#x20;   ▼

Retry

&#x20;   │

&#x20;   ├──────► Success

&#x20;   │

&#x20;   └──────► Failed

```



Persisting processing state allows background workers to determine what remains to be completed.



\---



\## 4. Durable Background Jobs



Long-running document processing is implemented as a durable background workflow.



Instead of keeping a browser request open:



```text

User Request

&#x20;   │

&#x20;   ▼

Create Job

&#x20;   │

&#x20;   ▼

Return Quickly

```



A scheduled worker later processes the job:



```text

Scheduled Worker

&#x20;      │

&#x20;      ▼

Find Pending Work

&#x20;      │

&#x20;      ▼

Claim Job

&#x20;      │

&#x20;      ▼

Process Bounded Unit

&#x20;      │

&#x20;      ▼

Persist Result

```



This keeps the interactive request separate from expensive document-processing work.



\---



\## 5. Job Leases



Workers use short-lived leases when claiming jobs.



The lease prevents multiple workers from processing the same job concurrently.



Conceptually:



```text

&#x20;                Job Queue

&#x20;                    │

&#x20;         ┌──────────┴──────────┐

&#x20;         ▼                     ▼

&#x20;      Worker A              Worker B

&#x20;         │                     │

&#x20;      Claim Job             Job Unavailable

&#x20;         │

&#x20;      Lease Acquired

&#x20;         │

&#x20;         ▼

&#x20;      Processing

```



If a worker terminates unexpectedly and its lease expires, the job can become eligible for processing again.



This provides recovery from interrupted background execution.



\---



\## 6. Retry and Backoff



External AI and infrastructure services can fail temporarily.



The processing system therefore supports bounded retries.



```text

Attempt 1

&#x20;  │

&#x20;  └── Failure

&#x20;       │

&#x20;       ▼

&#x20;     Backoff

&#x20;       │

&#x20;       ▼

Attempt 2

&#x20;  │

&#x20;  └── Failure

&#x20;       │

&#x20;       ▼

&#x20;     Backoff

&#x20;       │

&#x20;       ▼

Attempt N

&#x20;  │

&#x20;  ├── Success → Complete

&#x20;  │

&#x20;  └── Failure → Failed

```



Retry information is persisted rather than kept only in application memory.



This makes failures observable and allows processing to resume safely.



\---



\## 7. Idempotent Processing



Document-processing operations should be safe to retry.



Deterministic identifiers are used for processed content and indexed chunks so that repeated processing does not unnecessarily create duplicate records.



```text

Same Source Page

&#x20;      │

&#x20;      ▼

Deterministic Processing Identity

&#x20;      │

&#x20;      ▼

Same Logical Output

```



This is particularly important when a worker loses its connection after successfully completing an external operation but before recording the result locally.



\---



\## 8. AI Extraction Pipeline



The extraction layer uses AI to transform document content into structured information.



A simplified pipeline is:



```text

Document Page

&#x20;    │

&#x20;    ▼

Text / Image Extraction

&#x20;    │

&#x20;    ▼

AI Processing

&#x20;    │

&#x20;    ├── Classification

&#x20;    ├── Transcription

&#x20;    └── Structured Extraction

&#x20;            │

&#x20;            ▼

&#x20;       Schema Validation

&#x20;            │

&#x20;            ▼

&#x20;      Normalized Data

```



The application does not treat raw model output as trusted application data.



Structured responses are validated before they are used by downstream workflows.



\---



\## 9. Provider Abstraction



AI integrations are isolated behind application-level adapters.



```text

&#x20;                   Application

&#x20;                        │

&#x20;                        ▼

&#x20;               Extraction Interface

&#x20;                        │

&#x20;             ┌──────────┴──────────┐

&#x20;             ▼                     ▼

&#x20;       Anthropic Adapter       Test Adapter

&#x20;             │                     │

&#x20;             ▼                     ▼

&#x20;       External AI API        Deterministic Tests

```



This design prevents provider-specific SDK behavior from spreading throughout the application.



It also makes automated testing possible without requiring live AI API calls.



\---



\## 10. Persisting Intermediate Results



Intermediate document-processing results are persisted rather than keeping the entire pipeline in memory.



For example:



```text

Document

&#x20;  │

&#x20;  ▼

Pages

&#x20;  │

&#x20;  ▼

Transcripts

&#x20;  │

&#x20;  ▼

Chunks

&#x20;  │

&#x20;  ▼

Vectors

```



Each stage has a durable representation.



This allows later stages to be retried or rebuilt without repeating every previous operation.



It also makes the processing pipeline easier to inspect and debug.



\---



\## 11. Chunking and Indexing



After extraction, normalized text is divided into retrieval-friendly chunks.



```text

Persisted Transcript

&#x20;       │

&#x20;       ▼

&#x20;    Chunking

&#x20;       │

&#x20;       ▼

&#x20;Metadata Attachment

&#x20;       │

&#x20;       ▼

&#x20;    Embedding

&#x20;       │

&#x20;       ▼

&#x20;   Vector Upsert

&#x20;       │

&#x20;       ▼

&#x20;    Pinecone

```



Each indexed chunk retains enough metadata to identify its source document and page.



The vector index therefore acts as a searchable representation of persisted application data rather than the primary source of truth.



\---



\## 12. Resumability



The pipeline is designed so that processing can resume from the last successfully persisted stage.



For example:



```text

Pages 1–20     ✓

Pages 21–40    ✓

Pages 41–60    ✗

Pages 61–80    Pending

```



A retry does not need to restart the entire document.



Instead:



```text

Resume

&#x20; │

&#x20; ▼

Process Failed / Pending Work

&#x20; │

&#x20; ▼

Continue Pipeline

```



This is particularly useful for large documents where reprocessing the entire source would be expensive and time-consuming.



\---



\## 13. Handling Serverless Limits



The system was designed around bounded execution rather than assuming unlimited compute.



The processing strategy is:



\* Split large work into smaller units.

\* Persist intermediate results.

\* Use background jobs for expensive work.

\* Keep individual executions bounded.

\* Retry failed units independently.

\* Avoid sending large documents through a single application request.



This architecture allows the application to remain compatible with a serverless deployment model while supporting substantially larger processing workloads.



\---



\## 14. Data Flow



The complete document lifecycle can be represented as:



```text

&#x20;                ┌──────────────────┐

&#x20;                │      Browser     │

&#x20;                └────────┬─────────┘

&#x20;                         │

&#x20;                    Signed Upload

&#x20;                         │

&#x20;                         ▼

&#x20;                ┌──────────────────┐

&#x20;                │ Private Storage │

&#x20;                └────────┬─────────┘

&#x20;                         │

&#x20;                         ▼

&#x20;                ┌──────────────────┐

&#x20;                │    PostgreSQL    │

&#x20;                │ Document Record  │

&#x20;                └────────┬─────────┘

&#x20;                         │

&#x20;                         ▼

&#x20;                   Background Job

&#x20;                         │

&#x20;                         ▼

&#x20;                ┌──────────────────┐

&#x20;                │ Page Processing  │

&#x20;                └────────┬─────────┘

&#x20;                         │

&#x20;                         ▼

&#x20;                ┌──────────────────┐

&#x20;                │    Transcripts   │

&#x20;                └────────┬─────────┘

&#x20;                         │

&#x20;                         ▼

&#x20;                     Chunking

&#x20;                         │

&#x20;                         ▼

&#x20;                ┌──────────────────┐

&#x20;                │     Pinecone     │

&#x20;                │   Vector Index   │

&#x20;                └──────────────────┘

```



\---



\## 15. Security Boundaries



Document processing handles potentially sensitive business information.



The architecture therefore separates storage, authorization, processing, and retrieval responsibilities.



\### Private Storage



Original documents are stored in private object storage.



\### Database Authorization



PostgreSQL Row-Level Security protects application records.



\### Server-Side Processing



AI provider calls are performed by trusted server-side code.



\### Retrieval Isolation



Indexed content contains metadata allowing retrieval to be constrained to the appropriate record.



\### Credential Protection



External service credentials are never exposed to browser clients.



\---



\## 16. Testing



The document-processing pipeline is tested at multiple levels.



\### Unit Tests



Core processing logic is tested independently, including:



\* Chunking

\* Normalization

\* Processing state

\* Retry calculations

\* Validation

\* Metadata generation



\### Integration Tests



Integration coverage includes:



\* Document registration

\* Upload workflows

\* Page processing

\* Database persistence

\* Vector indexing

\* Resumable processing



\### Failure Tests



The system explicitly tests:



\* AI provider failures

\* Invalid AI responses

\* Processing interruptions

\* Large documents

\* Oversized inputs

\* Retry behavior

\* Partial processing



\### End-to-End Tests



Browser-level tests verify complete document workflows from upload through downstream application usage.



\---



\## Design Principles



The document-processing system follows these principles:



1\. \*\*Large work is divided into bounded units.\*\*

2\. \*\*Processing state is durable.\*\*

3\. \*\*Jobs can be retried safely.\*\*

4\. \*\*Workers use leases to prevent duplicate concurrent processing.\*\*

5\. \*\*Intermediate results are persisted.\*\*

6\. \*\*External AI providers are abstracted behind adapters.\*\*

7\. \*\*Model output is validated before becoming application data.\*\*

8\. \*\*The vector index is a retrieval layer, not the system of record.\*\*

9\. \*\*Security boundaries are enforced independently of AI behavior.\*\*

10\. \*\*Serverless constraints are treated as architectural constraints, not runtime surprises.\*\*



\---



\## Summary



The document-processing architecture demonstrates how an AI-heavy workflow can be engineered around real infrastructure constraints.



Instead of relying on a single long-running request, the platform uses:



\* Private object storage

\* PostgreSQL document metadata

\* Page-level processing

\* Durable background jobs

\* Job leases

\* Retry and backoff

\* Deterministic identifiers

\* AI extraction

\* Structured validation

\* Incremental persistence

\* Pinecone vector indexing

\* Automated failure testing



The result is a resumable document pipeline capable of transforming large business documents into structured and searchable knowledge while remaining compatible with a serverless application architecture.




\# RAG Pipeline



\## Overview



The platform uses Retrieval-Augmented Generation (RAG) to ground AI features in business and training documents rather than relying solely on the language model's general knowledge.



The pipeline is designed around four goals:



\* Keep retrieved information scoped to the correct business record.

\* Process large documents within serverless execution limits.

\* Make retrieval deterministic and testable.

\* Provide grounded AI responses with traceable source references.



The RAG system is used by multiple application features, including administrative Q\&A, AI-generated assessment questions, learner study content, and the conversational readiness assessment.



\---



\## Architecture



```text

&#x20;                DOCUMENT INGESTION

&#x20;                        │

&#x20;                        ▼

&#x20;               ┌─────────────────┐

&#x20;               │  PDF / Document │

&#x20;               └────────┬────────┘

&#x20;                        │

&#x20;                        ▼

&#x20;                Page-level split

&#x20;                        │

&#x20;                        ▼

&#x20;               Text extraction

&#x20;                        │

&#x20;                        ▼

&#x20;                Text normalization

&#x20;                        │

&#x20;                        ▼

&#x20;                   Chunking

&#x20;                        │

&#x20;                        ▼

&#x20;                  Embeddings

&#x20;                        │

&#x20;                        ▼

&#x20;               ┌─────────────────┐

&#x20;               │     Pinecone    │

&#x20;               │  Vector Index   │

&#x20;               └────────┬────────┘

&#x20;                        │

&#x20;                        │

&#x20;             ───────────┼───────────

&#x20;                        │

&#x20;                   QUERY PATH

&#x20;                        │

&#x20;                        ▼

&#x20;                    User Query

&#x20;                        │

&#x20;                        ▼

&#x20;                Query embedding

&#x20;                        │

&#x20;                        ▼

&#x20;             Metadata / record filter

&#x20;                        │

&#x20;                        ▼

&#x20;                Vector retrieval

&#x20;                        │

&#x20;                        ▼

&#x20;                   Reranking

&#x20;                        │

&#x20;                        ▼

&#x20;               Top relevant passages

&#x20;                        │

&#x20;                        ▼

&#x20;             Grounded LLM generation

&#x20;                        │

&#x20;                        ▼

&#x20;                Response + sources

```



\---



\## 1. Document Ingestion



Documents are uploaded through the application and stored in private object storage.



The database maintains metadata describing each document and its processing state.



A simplified representation is:



```text

Document

&#x20;  │

&#x20;  ├── metadata

&#x20;  ├── processing status

&#x20;  └── pages

&#x20;        │

&#x20;        ├── page 1

&#x20;        ├── page 2

&#x20;        ├── page 3

&#x20;        └── ...

```



The application does not send an entire large document through a single serverless request.



Instead, documents are processed incrementally.



This allows the system to recover from individual processing failures without restarting the entire document.



\---



\## 2. Page-Level Processing



Large documents are divided into manageable page ranges before extraction.



This design addresses serverless execution and payload constraints.



```text

Large Document

&#x20;     │

&#x20;     ▼

┌───────────────┐

│ Page Range 1  │

├───────────────┤

│ Page Range 2  │

├───────────────┤

│ Page Range 3  │

├───────────────┤

│      ...      │

└───────────────┘

&#x20;     │

&#x20;     ▼

Independent processing

```



Each processing unit can succeed or fail independently.



The resulting page transcripts are persisted in PostgreSQL and become the source of truth for subsequent processing.



\---



\## 3. Text Extraction and Normalization



Extracted document content is normalized before indexing.



The extraction layer is separated from the RAG layer through integration adapters.



This provides several advantages:



\* AI providers can be replaced without rewriting domain logic.

\* Fake providers can be used during automated tests.

\* Provider-specific response formats remain isolated.

\* Extraction failures can be retried independently.



The resulting normalized text is stored with its document and page metadata.



\---



\## 4. Chunking



Page content is divided into smaller chunks before embedding.



The chunking strategy aims to balance:



\* Retrieval precision

\* Context completeness

\* Embedding efficiency

\* LLM context size



Chunks retain their relationship to the original document and page.



A simplified structure is:



```text

Document

&#x20;  │

&#x20;  ├── Page

&#x20;  │     ├── Chunk A

&#x20;  │     ├── Chunk B

&#x20;  │     └── Chunk C

&#x20;  │

&#x20;  └── Page

&#x20;        ├── Chunk D

&#x20;        └── Chunk E

```



Chunk identifiers are deterministic so that the same source content can be indexed and updated predictably.



\---



\## 5. Vector Indexing



Chunk embeddings are stored in Pinecone.



Each vector contains metadata required to constrain retrieval.



Conceptually:



```text

Vector

&#x20;├── embedding

&#x20;└── metadata

&#x20;      ├── record identifier

&#x20;      ├── document identifier

&#x20;      ├── document type

&#x20;      └── page information

```



The record identifier is particularly important.



It allows retrieval to be restricted to the business record currently being processed or queried.



This prevents semantically similar information belonging to another record from entering the context.



\---



\## 6. Retrieval



The retrieval process follows several stages.



```text

Query

&#x20; │

&#x20; ▼

Embedding

&#x20; │

&#x20; ▼

Vector Search

&#x20; │

&#x20; ▼

Metadata Filtering

&#x20; │

&#x20; ▼

Candidate Passages

&#x20; │

&#x20; ▼

Reranking

&#x20; │

&#x20; ▼

Final Context

```



\### Metadata Filtering



Retrieval is constrained using application-level metadata filters.



For user- or record-specific workflows, the relevant record identifier is applied as a hard filter.



This is an important security and correctness boundary.



Semantic similarity alone must never determine which user's information can be retrieved.



\---



\## 7. Reranking



Initial vector retrieval produces a candidate set of passages.



A reranking stage then evaluates those candidates against the query to improve relevance.



Conceptually:



```text

Vector Search

&#x20;    │

&#x20;    ▼

Candidate Set

&#x20;    │

&#x20;    ▼

Reranker

&#x20;    │

&#x20;    ▼

Highest-Relevance Passages

```



Only the most relevant passages are passed into the final LLM prompt.



This reduces unnecessary context and improves the signal-to-noise ratio of the generated response.



\---



\## 8. Grounded Generation



The selected passages are passed to the language model as contextual evidence.



The generation layer is instructed to answer using the supplied evidence rather than inventing information that is not present in the retrieved material.



```text

Retrieved Passages

&#x20;       │

&#x20;       ▼

┌─────────────────────┐

│ Grounded LLM Prompt │

└──────────┬──────────┘

&#x20;          │

&#x20;          ▼

&#x20;      LLM Response

&#x20;          │

&#x20;          ▼

&#x20;  Source References

```



The application keeps the RAG implementation behind an integration boundary so different AI providers can be substituted or mocked during testing.



\---



\## 9. RAG Consumers



The same retrieval infrastructure supports several product workflows.



\### Administrative Q\&A



Administrators can ask questions about indexed business documents.



The system retrieves relevant passages and generates an answer grounded in those passages.



\### AI Question Generation



The question-generation pipeline retrieves relevant business knowledge before asking the model to generate assessment questions.



The generated output is then validated against a structured schema before it enters the question bank.



```text

Business Knowledge

&#x20;      │

&#x20;      ▼

&#x20;     RAG

&#x20;      │

&#x20;      ▼

Relevant Context

&#x20;      │

&#x20;      ▼

Question Generation

&#x20;      │

&#x20;      ▼

Schema Validation

&#x20;      │

&#x20;      ▼

Question Bank

```



\### Learner Study Content



Relevant document passages can be retrieved to support learner study material and evidence-based content.



\### Conversational Readiness Assessment



The readiness assessment uses RAG to provide the conversational AI with relevant business knowledge while maintaining the predefined assessment flow and session state.



RAG provides the knowledge context; it does not independently decide the assessment sequence.



\---



\## 10. Reliability and Failure Handling



Document processing is designed as a resumable background workflow rather than a single long-running request.



```text

&#x20;            Queue

&#x20;              │

&#x20;              ▼

&#x20;        Claim + Lease

&#x20;              │

&#x20;              ▼

&#x20;          Process

&#x20;         /       \\

&#x20;    Success     Failure

&#x20;       │           │

&#x20;       ▼           ▼

&#x20;   Complete     Retry

&#x20;                   │

&#x20;                   ▼

&#x20;              Backoff

&#x20;                   │

&#x20;                   ▼

&#x20;             Max Attempts

&#x20;                   │

&#x20;                   ▼

&#x20;                Failed

```



The processing system uses:



\* Durable job records

\* Short-lived leases

\* Retry attempts

\* Backoff

\* Deterministic identifiers

\* Idempotent processing where appropriate

\* Persisted intermediate results



This allows processing to resume after temporary provider failures or serverless execution limits.



\---



\## 11. Serverless Constraints



The application runs on a serverless deployment model.



Long-running document processing therefore cannot depend on a single request remaining alive until the entire document is processed.



The system instead uses:



```text

Upload

&#x20; │

&#x20; ▼

Persist Job

&#x20; │

&#x20; ▼

Scheduled Worker

&#x20; │

&#x20; ├── Process a bounded unit

&#x20; │

&#x20; ├── Persist result

&#x20; │

&#x20; └── Continue later

```



This approach keeps individual executions bounded while allowing large documents to be processed over multiple jobs.



\---



\## 12. Data Consistency



PostgreSQL remains the system of record.



The vector index is treated as a retrieval layer rather than the authoritative source of application data.



```text

&#x20;            PostgreSQL

&#x20;         System of Record

&#x20;               │

&#x20;               │

&#x20;               ▼

&#x20;         Indexed Chunks

&#x20;               │

&#x20;               ▼

&#x20;            Pinecone

&#x20;       Retrieval / Search

```



This separation makes it possible to rebuild or re-index vectors from persisted document data without making the vector database the primary application database.



\---



\## 13. Security Considerations



The RAG system follows a defense-in-depth approach.



\### Record Isolation



Retrieval is restricted to the relevant business record.



\### Database Authorization



PostgreSQL Row-Level Security protects user-owned application data.



\### Private Documents



Source documents are stored in private object storage rather than publicly accessible URLs.



\### Server-Side AI Access



AI provider credentials are kept on the server and are not exposed to browser clients.



\### Controlled Privileged Access



Service-role credentials are restricted to trusted server-side workflows such as background processing and controlled administrative operations.



\---



\## 14. Testing Strategy



The RAG system is tested at multiple levels.



\### Unit Tests



Core RAG logic is tested independently, including:



\* Chunking behavior

\* Metadata construction

\* Retrieval parameters

\* Ranking logic

\* Context construction

\* Validation behavior



\### Integration Tests



Integration tests cover:



\* Document indexing

\* Vector integration

\* Retrieval consumers

\* Database interactions

\* Failure and retry behavior



\### Security Tests



Authorization tests verify that a user cannot retrieve information belonging to another user or record.



\### Failure Tests



External AI and vector providers are tested using fake providers and failure scenarios so the application can verify behavior without requiring production credentials.



\---



\## Design Principles



The RAG implementation follows several principles:



1\. \*\*Retrieval is constrained before generation.\*\*

2\. \*\*Semantic similarity is not an authorization mechanism.\*\*

3\. \*\*PostgreSQL remains the system of record.\*\*

4\. \*\*Large documents are processed incrementally.\*\*

5\. \*\*Intermediate results are persisted for resumability.\*\*

6\. \*\*External providers are isolated behind adapters.\*\*

7\. \*\*Generated output is validated before entering structured workflows.\*\*

8\. \*\*RAG provides evidence; application logic controls workflow and permissions.\*\*



\---



\## Summary



The RAG subsystem is implemented as part of a larger full-stack application rather than as an isolated chatbot.



Its architecture combines:



\* Next.js server-side application logic

\* PostgreSQL and Supabase

\* Private document storage

\* Incremental document processing

\* AI-based extraction

\* Embeddings

\* Pinecone vector search

\* Metadata-constrained retrieval

\* Reranking

\* Anthropic-powered generation

\* Structured output validation

\* Background jobs and retries

\* Automated security and integration testing



The resulting design provides a reusable retrieval layer that can support multiple AI workflows while keeping authorization, data integrity, and business logic outside the language model.




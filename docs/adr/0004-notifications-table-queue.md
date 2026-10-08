# ADR 0004: Database-Backed Job Queue

**Status:** Accepted

## Context

Several operations need to happen asynchronously after a user action.

Examples include sending notifications and other background processing tasks. These operations must be:

* Retryable when an external provider fails
* Idempotent so the same operation is not executed twice
* Durable across application restarts
* Decoupled from the user's primary request

Introducing a separate message-queue service would add infrastructure and operational complexity for the current workload.

## Decision

Use PostgreSQL as a lightweight durable job queue.

When a transactional operation creates a background task, the application inserts a job record into the database as part of the same transaction.

A scheduled worker then:

1. Fetches pending jobs.
2. Claims a job using a short-lived lease.
3. Processes the task.
4. Marks successful jobs as completed.
5. Retries failed jobs with backoff.
6. Makes failed jobs observable for investigation or reprocessing.

Jobs use an idempotency key to prevent duplicate processing.

## Reliability Model

```text
Application Transaction
        │
        ├── Persist business result
        │
        └── Create durable job
                │
                ▼
          Scheduled Worker
                │
          Claim + Lease
                │
        ┌───────┴────────┐
        │                │
     Success           Failure
        │                │
        ▼                ▼
    Completed       Retry + Backoff
                         │
                         ▼
                  Max Attempts
                         │
                         ▼
                    Failed State
```

## Consequences

### Benefits

* No additional queue infrastructure is required.
* Business state and job creation can be committed atomically.
* Failed jobs remain observable.
* Retry state is persisted.
* Idempotency protects against duplicate execution.
* The same pattern can support additional asynchronous workflows.

### Trade-offs

A database-backed queue is appropriate for moderate workloads but is not intended to replace a dedicated distributed messaging system at very high throughput.

If workload requirements increase significantly, the job abstraction can be migrated to a dedicated queue or worker platform without changing the application's domain-level workflow.

## Related Engineering Concerns

The implementation must also account for:

* Concurrent workers attempting to claim the same job
* Expired leases
* Maximum retry attempts
* Exponential/backoff delays
* Idempotent external API calls
* Secure authentication of scheduled workers
* Failure visibility and operational recovery

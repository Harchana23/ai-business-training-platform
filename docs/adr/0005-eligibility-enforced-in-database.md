# ADR 0005: Database-Enforced Business Rules

**Status:** Accepted

## Context

Some business rules determine whether a user or record becomes eligible for the next stage of a workflow.

These rules must remain correct even when data is modified through different application paths. Relying only on frontend or server-side calculations could allow inconsistent state when a record is updated outside the expected workflow.

The system also needs to preserve historical eligibility decisions rather than recalculating past decisions whenever a policy changes.

## Decision

Critical eligibility rules are enforced at the PostgreSQL database boundary.

When relevant source data is created or changed:

1. A database trigger invokes the eligibility calculation.
2. The calculated result is stored as an immutable snapshot.
3. The application domain layer contains the same rule for display and application-level reasoning.
4. Integration tests verify that the application and database implementations produce identical results, including boundary cases.

Policy values are configurable rather than hard-coded into individual workflows.

## Data Flow

```text
Source Data
    │
    ▼
PostgreSQL Trigger
    │
    ▼
Eligibility Calculation
    │
    ▼
Eligibility Snapshot
    │
    ├──────────────► Audit / Historical Record
    │
    ▼
Application Domain Layer
    │
    ▼
Current Workflow State
```

## Consequences

### Benefits

* Critical business rules cannot be bypassed by an alternative application path.
* Historical eligibility decisions remain available for auditing.
* Application and database calculations are validated against each other.
* Policy changes can be introduced without rewriting historical snapshots.
* Boundary conditions are covered by integration tests.

### Trade-offs

The same rule exists in both the database and application domain layer.

This introduces duplication, but the duplication is intentional: the database protects data integrity while the application layer provides predictable domain behavior and presentation logic.

A parity test reduces the risk of the two implementations diverging.

## Principle

> **Enforce data-integrity rules at the strongest boundary that must remain trustworthy.**

For rules that determine durable business state, database-level enforcement provides an additional layer of protection beyond application-level validation.

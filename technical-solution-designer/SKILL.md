---
name: technical-solution-designer
description: Use when Codex needs to produce or review an engineering-grade technical solution design from a PRD, product discussion, existing codebase, browser behavior, database records, or legacy implementation. Trigger for architecture/design docs, backend方案, approval/workflow/payment/card/fund lifecycle designs, service boundary analysis, state machines, permissions/RBAC, database schema, API contracts, sequence diagrams, migration plans, non-functional requirements, or when a user asks to “参考现有系统设计新流程”.
---

# Technical Solution Designer

## Purpose

Create implementation-ready technical方案 from product intent and existing system behavior. Prefer evidence from the PRD, UI, code, and database over assumptions. The output should help engineers implement safely, not merely describe a product idea.

## Core Workflow

1. **Read the source of truth**
   - Read the PRD and extract explicit requirements, implied requirements, unresolved ambiguities, and diagrams.
   - Inspect current UI flows when relevant, especially comparable existing products.
   - Inspect existing code, APIs, workflow definitions, database tables, permissions, and status transitions before proposing new patterns.
   - Separate confirmed facts from inference.

2. **Map existing capabilities first**
   - Identify what already exists: business interfaces, internal services, providers/adapters, workflow engines, RBAC helpers, tables, logs, async jobs, notifications, and error codes.
   - Draw current business flow, interaction flow, and code/data flow before designing the new one.
   - Reuse existing service boundaries and helper APIs unless they are demonstrably wrong for the new feature.

3. **Clarify domain semantics early**
   - Pin down money/limit semantics, ownership, actor identity, idempotency keys, lifecycle terminal states, and retry behavior.
   - For approval or payment-like features, explicitly define whether approval success means “ready for manual execution” or “automatically applied”.
   - Do not let labels from one product leak into another if the business semantics differ. For example, avoid `ready_to_pay` for a flow with no manual pay step.

4. **Design around state machines**
   - Define workflow-engine states separately from business-request states.
   - Map every workflow callback or timeout to a business state.
   - Preserve meaningful terminal states such as `rejected`, `cancelled`, `expired`, and `failed` when they represent different facts.
   - Prefer existing state vocabulary, but do not force poor semantics such as `paid` for non-payment completion.

5. **Set service boundaries**
   - Business service owns product APIs, business state, permissions, request details, and provider calls.
   - Workflow service owns generic definitions, instances, tasks, logs, auto approval, timeout, and callbacks.
   - RBAC/user service owns permission, role, member resolution.
   - Provider adapters own external calls and provider-local persistence already established by the codebase.
   - Avoid exposing workflow-private config directly to frontend or unrelated business modules.

6. **Design permissions explicitly**
   - List new and reused permission points.
   - Explain who can submit, approve, manage rules, view lists, cancel, retry, and execute.
   - Prefer centralized helpers such as `hasXPermission` instead of scattering permission checks across handlers.
   - State whether frontend permission display uses current-user permission APIs or role-editing APIs.

7. **Make data changes auditable**
   - Define request tables, status logs, relationships, indexes, uniqueness constraints, and status update order.
   - Use logs for business state transitions when future debugging, retries, provider failures, or fund/card reconciliation matter.
   - JSON metadata may vary by action, but must be bounded: keep common fields in table columns and context-specific fields in metadata; do not store sensitive card data or full provider responses.

8. **Handle concurrency and idempotency as first-class requirements**
   - Add business checks and database constraints for duplicate active requests.
   - Use locks or compare-and-set transitions for automatic execution.
   - State which terminal state makes repeated execution idempotently successful.
   - Include retry and compensation behavior for provider or balance failures.

9. **Produce diagrams that connect layers**
   - Include user interaction flow, business implementation flow, code/service sequence flow, database update order, and ER diagram when data changes are involved.
   - Diagrams should show decisions, ownership boundaries, and failure branches, not only happy paths.

10. **Close with non-functional requirements and open questions**
    - Cover idempotency, audit, observability, notifications, permissions, rollout, migration/backfill, failure handling, and metrics.
    - Keep open questions concrete and implementation-blocking; mark resolved decisions as such.

## Output Shape

Use this structure unless the user asks for a different format:

1. PRD analysis: background, actors, confirmed semantics, unresolved points.
2. Existing system analysis: current flows, service boundaries, tables, permissions, reusable code.
3. Overall solution: ownership by service, lifecycle, state mapping, permissions, and diagrams.
4. Module designs:
   - Business APIs and DTOs.
   - Permission/RBAC design.
   - Workflow design and callback behavior.
   - Provider/external adapter reuse.
   - Database schema, ER diagram, indexes, and update-order flow.
5. Non-functional requirements: idempotency, concurrency, audit logs, monitoring, notifications, migration, rollout, risks.
6. Implementation checklist and open questions.

## Design Principles To Preserve

- Prefer extending existing patterns over inventing a parallel framework.
- Keep workflow generic; keep product state and product execution in the product service.
- Do not conflate visibility with action authorization. A list item may be visible while execution still requires a separate permission check.
- Treat approval result and business execution result as different facts.
- Record enough audit context to answer: who requested, who approved, who executed, what changed, when, why it failed, and what external/fund/card records prove it.
- Add database constraints for invariants that must survive concurrency.
- Make executor/actor identity explicit when automatic execution reuses APIs that normally require a user.
- Avoid unbounded JSON metadata. Define action-specific recommended keys.
- State what is intentionally out of scope for the current phase.

## Review Checklist

Before finalizing a方案, verify:

- The PRD semantics are reflected and contradictions are called out.
- Existing code/table/API behavior has been checked, not guessed.
- The state machine distinguishes `rejected`, `cancelled`, `expired`, `failed`, processing, and success correctly.
- Every workflow callback has a business-state outcome.
- Permissions cover submit, approve, configure, view, cancel, retry, and execute.
- The frontend permission source is named.
- API boundaries match comparable existing modules.
- Tables include primary keys, indexes, uniqueness constraints, timestamps, and JSON usage rules.
- Data update order is clear for submit, approval, auto approval, execution success, execution failure, rejection, cancellation, and expiration.
- Idempotency and duplicate active request protection are explicit.
- Audit logs and metadata fields support future debugging without storing sensitive data.
- Diagrams are complete enough for another engineer to implement from them.

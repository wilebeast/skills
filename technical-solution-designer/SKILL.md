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
   - Treat existing products' real business use cases as design inputs, not merely as code to reuse. For each comparable product, identify its actor, outcome, preconditions, normal and exceptional paths, state meaning, funds/limit effects, user-visible result, and operational recovery path. Use this baseline together with the new PRD before deriving any shared abstraction.
   - Reuse existing service boundaries and helper APIs unless they are demonstrably wrong for the new feature.

3. **Derive the design from use cases**
   - Start with business use cases in product language: actor, desired outcome, trigger, preconditions, success result, and user-visible failure result.
   - For each business use case, enumerate the primary path and meaningful variations: rejection, validation failure, provider failure, timeout, duplicate request, retry, cancellation, compensation, and concurrent execution.
   - Derive the facts, state transitions, invariants, permission decisions, and fund/limit effects required by those paths before designing APIs or tables.
   - Decompose each business use case into application-orchestration use cases. These own the ordered steps that coordinate business state, authorization, ledger/fund actions, asynchronous work, idempotency, and external calls; they are not merely a mechanical list of HTTP handlers.
   - Extract a shared application use case only when two or more business use cases share the same business meaning and change boundary, or when a confirmed variation point requires it. Do not generalize from a single endpoint or speculative future channel.
   - Define provider/channel capabilities from the application use cases: stable provider-neutral commands, queries, and outcomes such as `OpenCard`, `QueryRequestResult`, or `UpdateLimit`. Keep provider URI, field mapping, authentication, and provider-code parsing inside the Adapter implementation.
   - For card, payment, workflow, or other multi-product domains, compare equivalent use cases horizontally across every product before finalizing an abstraction. A product is not a reason to create a different layered model for the same business meaning.
     Build the comparison from business outcome through application orchestration, channel capability, Adapter, state, event, persistence, retry, and observability. State whether a difference is a true semantic difference or an explicit variation point; do not hide it in product-named services, tables, DTOs, or state machines.
   - For the same business semantic, require model homomorphism across products: the same layer owns the same kind of fact, uses the same contract shape and lifecycle vocabulary, and exposes differences through declared capabilities, strategies, policy/configuration, or Adapter mappings. If this is impossible, document why the business semantics are genuinely different and define a separate use case rather than silently diverging the layered model.
   - Record the derivation explicitly in a use-case matrix:

     | Layer | Question answered | Required output |
     | --- | --- | --- |
     | Business | What outcome does the actor need and what rules apply? | Product use case, rules, states, UX result |
     | Application / logic | What coordinated steps safely realize that outcome? | Orchestration use case, ownership, transactions, events, retries |
     | Channel capability | What external ability is required, independent of a vendor URI? | Command/query contract and normalized outcome |
     | Provider Adapter | How does this vendor implement that ability? | API mapping, DTO conversion, auth, error and webhook normalization |

   - A useful derivation chain is: `business goal -> paths and variations -> facts/invariants/state machine -> orchestration steps -> shared application use cases -> channel capability contract -> provider API mapping -> data/events/jobs`.
   - Add a cross-product expansion matrix whenever the scope contains, or is expected to contain, multiple card products:

     | Layer | Common business semantic | Product-independent model | Explicit variation point | Product-specific implementation evidence |
     | --- | --- | --- | --- | --- |
     | Business | What outcome is identical for all products? | Shared use case, rules, state meaning, UX contract | Eligibility, fee, limit, or capability differences | Product policy/configuration |
     | Application / logic | What orchestration is common? | Shared command, state transition, idempotency, retry/compensation boundary | Product policy or capability branch | Product resolver/strategy |
     | Channel capability | What provider-neutral ability is needed? | Stable command/query and normalized result/event | Capability support or command mode | Capability implementation |
     | Provider Adapter | How is the capability realized? | Adapter boundary and normalized DTO | URI, credentials, field mapping, provider codes | Provider-specific API mapping |
     | Data / event / job | What fact is persisted or replayed? | Shared entity/event/operation model and audit fields | Product-scoped metadata only | Adapter payload/context |

     Reject a design where equivalent products require different application services, state-machine shapes, event categories, or primary persistence models solely because their provider APIs differ.

4. **Clarify domain semantics early**
   - Pin down money/limit semantics, ownership, actor identity, idempotency keys, lifecycle terminal states, and retry behavior.
   - For approval or payment-like features, explicitly define whether approval success means “ready for manual execution” or “automatically applied”.
   - Do not let labels from one product leak into another if the business semantics differ. For example, avoid `ready_to_pay` for a flow with no manual pay step.

5. **Design around state machines**
   - Define workflow-engine states separately from business-request states.
   - Map every workflow callback or timeout to a business state.
   - Preserve meaningful terminal states such as `rejected`, `cancelled`, `expired`, and `failed` when they represent different facts.
   - Prefer existing state vocabulary, but do not force poor semantics such as `paid` for non-payment completion.

6. **Set service boundaries**
   - Business service owns product APIs, business state, permissions, request details, and provider calls.
   - Workflow service owns generic definitions, instances, tasks, logs, auto approval, timeout, and callbacks.
   - RBAC/user service owns permission, role, member resolution.
   - Provider adapters own external calls and provider-local persistence already established by the codebase.
   - Avoid exposing workflow-private config directly to frontend or unrelated business modules.

7. **Design permissions explicitly**
   - List new and reused permission points.
   - Explain who can submit, approve, manage rules, view lists, cancel, retry, and execute.
   - Prefer centralized helpers such as `hasXPermission` instead of scattering permission checks across handlers.
   - State whether frontend permission display uses current-user permission APIs or role-editing APIs.

8. **Make data changes auditable**
   - Define request tables, status logs, relationships, indexes, uniqueness constraints, and status update order.
   - Use logs for business state transitions when future debugging, retries, provider failures, or fund/card reconciliation matter.
   - JSON metadata may vary by action, but must be bounded: keep common fields in table columns and context-specific fields in metadata; do not store sensitive card data or full provider responses.

9. **Handle concurrency and idempotency as first-class requirements**
   - Add business checks and database constraints for duplicate active requests.
   - Use locks or compare-and-set transitions for automatic execution.
   - State which terminal state makes repeated execution idempotently successful.
   - Include retry and compensation behavior for provider or balance failures.

10. **Produce diagrams that connect layers**
   - Include user interaction flow, business implementation flow, code/service sequence flow, database update order, and ER diagram when data changes are involved.
   - Diagrams should show decisions, ownership boundaries, and failure branches, not only happy paths.

11. **Close with non-functional requirements and open questions**
    - Cover idempotency, audit, observability, notifications, permissions, rollout, migration/backfill, failure handling, and metrics.
    - Keep open questions concrete and implementation-blocking; mark resolved decisions as such.

## Output Shape

Use this structure unless the user asks for a different format:

1. PRD analysis: background, actors, confirmed semantics, unresolved points.
2. Use-case derivation: business use cases and variations; cross-product expansion matrix; application-orchestration use cases; channel capability contracts; provider API mapping.
3. Existing system analysis: current flows, service boundaries, tables, permissions, reusable code.
4. Overall solution: ownership by service, lifecycle, state mapping, permissions, and diagrams.
5. Module designs:
   - Business APIs and DTOs.
   - Permission/RBAC design.
   - Workflow design and callback behavior.
   - Provider/external adapter reuse.
   - Database schema, ER diagram, indexes, and update-order flow.
6. Non-functional requirements: idempotency, concurrency, audit logs, monitoring, notifications, migration, rollout, risks.
7. Implementation checklist and open questions.

## Design Principles To Preserve

- Prefer extending existing patterns over inventing a parallel framework.
- Let business use cases and their failure/concurrency variants drive layer boundaries; do not start from provider HTTP endpoints or a preferred abstraction.
- Keep business use cases, application-orchestration use cases, channel capability contracts, and provider API mappings distinct. A provider API is evidence for an Adapter, not automatically a business or shared application interface.
- Generalize only on demonstrated common semantics or a confirmed variation point; preserve product-specific behavior when its rules, states, funds, or ownership differ.
- For equivalent cross-product use cases, preserve one layered model end-to-end. Product variation must be visible as an explicit capability, policy, strategy, configuration, or Adapter mapping—not as a parallel product-specific service, state machine, event model, or core table.
- Perform the horizontal comparison before adding a product-specific abstraction. If a proposed difference reaches more than one layer, either collapse it into a declared variation point or prove it is a distinct business semantic and name it as a separate use case.
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
- Existing comparable products' actual business use cases—not only their APIs, tables, or implementation code—have been enumerated as inputs to the new design.
- Each proposed module/interface/table can be traced back to a named business use case, variation, invariant, or confirmed provider capability.
- Business, application-orchestration, channel capability, and provider Adapter use cases are separated; no provider URI has been promoted directly into a business abstraction without a business-semantic reason.
- Equivalent use cases have been compared horizontally across current and planned card products at business, application, capability, Adapter, state/event, data, retry, and observability layers.
- The design uses one layered model for equivalent product semantics; every product difference is explicitly classified as a policy/configuration, capability, strategy, or Adapter variation. Any intentionally separate model is justified by a documented semantic difference.
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

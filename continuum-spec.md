# Continuum Engineering Specification

## 1. Document Purpose

This document defines the first engineering specification for **Continuum**, a workflow-resilience layer for essential digital services operating under poor, unstable, intermittent, or temporarily unavailable internet connectivity.

The goal is to lock the MVP behavior before implementation so the frontend, backend, transaction engine, local storage, network simulator, and prototype demo follow the same model.

## 2. Product Identity

**Project Name:** Continuum  
**Sector:** Technology for Social Good  
**Problem Statement:** How can essential digital services stay useful under poor internet connectivity?

### Core Concept

Continuum introduces a **Minimum Viable Transaction (MVT)** model.

When the available connection is insufficient to complete an entire digital workflow, Continuum identifies and preserves the smallest safe and dependency-valid part of the user's intended task, then progressively completes deferred information when connectivity improves.

> **Core principle:** Preserve user intent and valid task progress, not only stored data.

## 3. MVP Objective

The MVP must prove four things:

1. A digital workflow can be represented as mandatory, dependent, and deferrable components.
2. Continuum can create a smaller valid transaction from a larger full transaction.
3. The core transaction can be acknowledged without requiring the full workflow to complete.
4. Deferred components can resume automatically when connectivity improves.

The MVP is not intended to prove production-grade performance or compatibility with real government systems.

## 4. Demo Application

The initial demonstration application will be a fictional **Citizen Benefit Application Portal**.

It should resemble a real digital public-service workflow without impersonating or integrating with any real government portal.

### User Flow

1. Start application
2. Enter personal details
3. Complete eligibility details
4. Upload supporting documents
5. Review application
6. Submit
7. Track completion status

### Example Data Fields

#### Core data
- applicant_id
- scheme_id
- full_name
- eligibility_status
- created_at

#### Deferred data
- profile_photo
- identity_proof
- bank_proof
- supporting_document

## 5. Transaction State Machine

```text
DRAFT
  ↓
SAVED_LOCAL
  ↓
INTENT_SECURED
  ↓
MVT_READY
  ↓
MVT_SENT
  ↓
SERVER_ACKNOWLEDGED
  ↓
CORE_COMPLETE
  ↓
DEFERRED_PENDING
  ↓
FULLY_COMPLETE
```

Failure/recovery states:

```text
VALIDATION_FAILED
RETRY_PENDING
SYNC_FAILED
```

### State Semantics

- **SAVED_LOCAL:** The device has safely stored the user's current data.
- **INTENT_SECURED:** A unique intent record has been created locally.
- **SERVER_ACKNOWLEDGED:** The server has received and acknowledged the MVT.
- **CORE_COMPLETE:** The service-defined minimum valid transaction has completed.
- **FULLY_COMPLETE:** All required deferred items have been transmitted and confirmed.

A local save must never be presented as server acknowledgement.

## 6. Minimum Viable Transaction

### Definition

> The smallest safe, dependency-valid subset of a workflow that preserves the user's core intent and reaches a valid checkpoint under current connectivity constraints.

### Illustrative Example

| Component | Approx. Size |
|---|---:|
| Personal details | 3 KB |
| Eligibility details | 2 KB |
| Photo | 700 KB |
| ID proof | 400 KB |
| Bank proof | 300 KB |
| Supporting PDF | 2 MB |
| **Total** | **≈ 3.4 MB** |

Illustrative MVT:

- Applicant ID
- Scheme ID
- Eligibility
- Timestamp
- Intent ID

Illustrative MVT size: **≈ 5–10 KB**.

These values are for demonstration and are not measured production benchmarks.

## 7. Workflow Definition

V1 will use developer-defined workflow configuration.

```yaml
workflow: benefit_application

checkpoint:
  name: core_application_registered

required:
  - applicant_id
  - scheme_id
  - full_name
  - eligibility_status
  - created_at

deferred:
  - profile_photo
  - identity_proof
  - bank_proof
  - supporting_document
atomic:
  - create_application_intent
```

### Workflow Compiler Responsibilities

- load workflow definitions
- identify mandatory fields
- identify deferrable fields
- identify dependencies
- identify atomic actions
- identify safe checkpoints
- expose the compiled workflow to the MVT Engine

Automatic AI-based workflow generation is not required for the MVP.

## 8. MVT Engine

### Inputs

```text
Compiled Workflow
+
Current User Transaction
+
Payload Metadata
+
Connectivity State
+
Safety / Dependency Rules
```

### Output Example

```json
{
  "can_execute": true,
  "checkpoint": "core_application_registered",
  "core_fields": [
    "applicant_id",
    "scheme_id",
    "full_name",
    "eligibility_status",
    "created_at"
  ],
  "deferred_fields": [
    "profile_photo",
    "identity_proof",
    "bank_proof",
    "supporting_document"
  ]
}
```

### MVP Algorithm

```text
1. Load workflow definition.
2. Validate mandatory dependencies.
3. Build the minimum checkpoint payload.
4. Estimate serialized core payload size.
5. Compare it with the current connectivity budget.
6. If the core payload is feasible:
      create MVT
      mark non-core data as deferred
   Else:
      preserve locally
      wait for a usable connectivity window
```

## 9. Connectivity Engine

### Purpose

Estimate whether a transaction is likely to complete under current conditions.

### Input Signals

- browser online/offline status
- recent request latency
- recent request success rate
- timeout frequency
- estimated throughput
- simulator mode

### Connectivity Classes

```text
GOOD
LIMITED
POOR
INTERMITTENT
OFFLINE
```

Example:

```json
{
  "state": "POOR",
  "estimated_bandwidth_kbps": 20,
  "stability_score": 0.42,
  "can_attempt_mvt": true,
  "can_attempt_full_transaction": false
}
```

The MVP should not rely on a single browser bandwidth estimate. Simulated conditions and observed request behavior should be combined.

## 10. Intent Ledger

Required fields:

```text
intent_id
application_id
workflow_id
current_state
core_payload_hash
server_ack_id
retry_count
created_at
updated_at
last_attempt_at
```

Requirements:

- intent ID generated before sending the MVT
- unique across logical transactions
- same intent ID reused for retries
- retries must never create duplicate logical submissions

## 11. Idempotency and Duplicate Protection

Server behavior:

```text
IF intent_id already exists:
    return existing transaction state
ELSE:
    create new transaction
```

Expected result if the user clicks Submit five times during poor connectivity:

```text
5 HTTP attempts
1 logical transaction
1 application intent
```

## 12. Local Storage

The web MVP will use **IndexedDB**.

Suggested stores:

### applications
- application_id
- form_data
- workflow_id
- updated_at

### intents
- intent_id
- application_id
- current_state
- payload_hash
- server_ack_id

### pending_operations
- operation_id
- intent_id
- operation_type
- payload_reference
- priority
- status
- retry_count
- created_at

### attachments
- attachment_id
- intent_id
- type
- local_reference
- size
- status

## 13. Progressive Completion Engine

After core acknowledgement:

```text
Photo                PENDING
Identity Proof       PENDING
Bank Proof           PENDING
Supporting Document  PENDING
```

When connectivity improves:

```text
Identity Proof       COMPLETE
Bank Proof           COMPLETE
Photo                COMPLETE
Supporting Document  COMPLETE
```

Final state:

```text
FULLY_COMPLETE
```

### Retry Rules

- retry only failed pending operations
- use bounded exponential backoff
- do not restart acknowledged core transactions
- preserve retry state across refreshes
- stop retrying permanently invalid operations

## 14. Client SDK Responsibilities

The Continuum client layer should handle:

- workflow loading
- local persistence
- intent creation
- network-state monitoring
- MVT creation
- queue management
- retries
- progressive completion
- UI status updates

Initially this can be implemented as internal TypeScript modules rather than a published SDK.

## 15. Backend Responsibilities

The backend must provide:

- workflow metadata
- application intent creation
- idempotent MVT acceptance
- intent lookup
- attachment upload
- progressive completion tracking
- final completion state
- simulator support for controlled demonstration

## 16. Initial API Contracts

### Create / acknowledge MVT

```http
POST /api/intents
```

Request:

```json
{
  "intent_id": "CNT-A829X",
  "workflow_id": "benefit_application",
  "application_id": "APP-1042",
  "core_payload": {
    "applicant_id": "USR-123",
    "scheme_id": "SCH-101",
    "full_name": "Demo User",
    "eligibility_status": true,
    "created_at": "2026-10-03T12:00:00Z"
  }
}
```

Response:

```json
{
  "intent_id": "CNT-A829X",
  "ack_id": "ACK-49211",
  "state": "CORE_COMPLETE",
  "deferred_items_pending": 4
}
```

### Get Intent State

```http
GET /api/intents/{intent_id}
```

### Upload Deferred Attachment

```http
POST /api/intents/{intent_id}/attachments
```

### Final Completion

```http
POST /api/intents/{intent_id}/complete
```

The server must return `FULLY_COMPLETE` only after all required deferred operations are satisfied.

## 17. Database Schema

### users

```text
id
name
phone
created_at
```

### applications

```text
id
user_id
scheme_id
workflow_id
status
created_at
updated_at
```

### intents

```text
id
application_id
workflow_id
state
core_payload_hash
server_ack_id
retry_count
created_at
updated_at
```

### attachments

```text
id
intent_id
attachment_type
storage_path
size_bytes
status
created_at
updated_at
```

### sync_operations

```text
id
intent_id
operation_type
priority
status
retry_count
last_error
created_at
updated_at
```

## 18. Proposed Technology Stack

### Frontend
- Next.js
- TypeScript
- Tailwind CSS

### Offline Storage
- IndexedDB

### Backend
- FastAPI
- Python

### Database
MVP: SQLite  
Later: PostgreSQL

### Testing
Frontend:
- Vitest or Jest
- Playwright

Backend:
- Pytest

## 19. Continuum Lab / Network Simulator

Presets:

```text
FAST
3G
2G
20 KBPS
INTERMITTENT
OFFLINE
```

Simulator controls, where technically feasible:

- artificial delay
- request timeout
- upload throttling
- controlled failures
- intermittent availability

Visible developer panel:

```text
Current network
Current transaction state
Full transaction size
MVT size
Pending operations
Completed operations
Retry count
```

The simulator is for controlled demonstration and must not be presented as real-world benchmark data.

## 20. Demo Modes

### Conventional Mode

```text
Full transaction required
↓
Poor network
↓
Timeout / failure
↓
User must retry
```

### Continuum Mode

```text
Workflow analyzed
↓
MVT generated
↓
Core transaction acknowledged
↓
Deferred operations queued
↓
Connectivity improves
↓
Progressive completion
```

Both modes should use the same simulated conditions.

## 21. UI Requirements

Example status screen:

```text
Application Status

✓ Saved on device
✓ Core application registered

○ Photo waiting for upload
○ Identity proof waiting for upload
○ Bank proof waiting for upload

You may safely close this page.
Pending uploads will resume when connectivity improves.
```

Required labels:

- Saved Locally
- Intent Secured
- Server Acknowledged
- Core Complete
- Documents Pending
- Fully Complete

Avoid “submitted successfully” until the service-defined completion state is actually reached.

## 22. Security Requirements

- generate non-guessable intent IDs
- hash critical payloads for integrity tracking
- validate file types
- validate file sizes
- sanitize filenames
- perform server-side validation
- enforce idempotency
- keep secrets out of frontend source
- do not claim encryption unless it is implemented

## 23. Edge Cases

### Browser refresh
Restore local application, queue, and transaction state.

### Repeated Submit
Reuse the same intent. Do not create duplicate logical transactions.

### Network disappears before MVT send
Remain `INTENT_SECURED`; retry later.

### MVT reaches server but acknowledgement response is lost
Retry with same intent ID; server returns existing acknowledgement.

### One attachment fails
Retry only that attachment.

### User closes application
Persist state and reconstruct it on next launch.

### Core validation fails
Enter `VALIDATION_FAILED`; never show `CORE_COMPLETE`.

### Workflow cannot legally be split
Mark the operation atomic and keep it locally preserved until sufficient connectivity exists.

## 24. Acceptance Criteria

- **AC-01:** Partially completed form survives refresh.
- **AC-02:** Exactly one intent ID is generated for one logical submission.
- **AC-03:** Repeated retries do not create duplicate server transactions.
- **AC-04:** MVT contains only required checkpoint data plus required metadata.
- **AC-05:** Weak-network mode preserves the transaction when the full request cannot complete.
- **AC-06:** UI shows server acknowledgement only after a real response.
- **AC-07:** Deferred attachments remain queued after core acknowledgement.
- **AC-08:** Pending uploads resume automatically when connectivity improves.
- **AC-09:** `FULLY_COMPLETE` occurs only after all required deferred work completes.
- **AC-10:** Browser restart restores transaction state from local and server state.

## 25. MVP Test Scenarios

### Stable Connection
Full workflow completes normally.

### Weak but Usable Connection
MVT succeeds; large deferred items remain queued.

### Intermittent Connection
MVT retries safely without duplicate creation.

### Fully Offline
Application is saved locally; no server acknowledgement is shown.

### Recovery
Network returns; pending operations resume and eventually reach `FULLY_COMPLETE`.

## 26. Development Phases

### Phase 1 — Base Demo Service
- Next.js frontend
- FastAPI backend
- application flow
- SQLite persistence
- file uploads

### Phase 2 — Local Resilience
- IndexedDB
- local persistence
- connectivity detection
- local intent creation

### Phase 3 — Continuum Core
- workflow definitions
- Workflow Compiler
- MVT Engine
- deferred classification

### Phase 4 — Intent Ledger
- stable intent IDs
- idempotent backend
- state machine
- acknowledgement handling

### Phase 5 — Progressive Completion
- persistent upload queue
- retry engine
- reconnect detection
- automatic resume

### Phase 6 — Continuum Lab
- simulator presets
- transaction metrics
- conventional vs Continuum mode

### Phase 7 — Polishing
- status dashboard
- error handling
- end-to-end testing
- demo script
- deployment

## 27. Future Research Direction

Represent workflows as graphs:

```text
G = (V, E)
```

Each node may contain:

```text
payload_size
mandatory
dependency
atomicity
checkpoint
```

Given estimated connectivity budget `B`, future versions can search for subset `S` such that:

```text
sum(payload_size(S)) <= B
```

while satisfying dependency and safety constraints.

Goal:

> Find the smallest dependency-complete safe checkpoint that preserves the user's intended task.

This graph optimization is a future enhancement and is not required for the first MVP.

## 28. Explicit MVP Non-Goals

The MVP will not include:

- real Aadhaar integration
- real government-service integration
- payments or UPI
- medical diagnosis
- mesh networking
- peer-to-peer relays
- Android/iOS native apps
- automatic AI workflow generation
- predictive ML bandwidth scheduling
- unsupported claims of production-scale resilience

## 29. Definition of MVP Done

The MVP is complete when a reviewer can:

1. open the demo service
2. fill an application
3. add large attachments
4. switch to a weak simulated connection
5. observe conventional mode fail
6. enable Continuum mode
7. observe an MVT being created
8. receive a real backend acknowledgement for the core transaction
9. see deferred files queued
10. restore good connectivity
11. observe queued files resume
12. reach `FULLY_COMPLETE`
13. refresh/reopen without losing transaction state
14. repeat Submit without creating a duplicate transaction

## 30. Product Vision

```text
Essential Digital Service
          ↓
     Continuum SDK
          ↓
    Workflow Contract
          ↓
       MVT Runtime
          ↓
 Reliable User Progress
```

Future API concept:

```ts
continuum.defineWorkflow({
  name: "benefit-application",
  checkpoint: "core-registered",
  required: ["applicantId", "schemeId", "eligibility"],
  deferred: ["photo", "identityProof", "supportingDocument"]
});
```

Continuum would manage local persistence, intent identity, safe retries, MVT execution, acknowledgement state, and progressive completion.

## 31. Final Engineering Principle

> **Continuum must never pretend that a transaction is complete when it is only stored locally. It should preserve user intent honestly, advance the workflow only through valid checkpoints, and progressively complete the remaining work when connectivity permits.**

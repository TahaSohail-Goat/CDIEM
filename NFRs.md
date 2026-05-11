# Non-Functional Requirements (NFRs) — CDIEM

This document consolidates the non-functional requirements for the CDIEM system.

## 1. Security

- Passwords must never be stored in plaintext; they must be stored as SHA-256 hashes.
- Evidence integrity must be protected by storing an original hash at upload and verifying it by hash recalculation/comparison.
- Database access must prevent SQL injection by using parameterized queries (`PreparedStatement`).
- Evidence file storage must prevent unsafe file handling through sanitized filenames, unique timestamp/UUID naming, and controlled storage directories.
- Role-based access control (RBAC) must restrict module access and actions according to user role (`OFFICER`, `ANALYST`, `SUPERVISOR`) across UI and business logic layers.

## 2. Auditability and Traceability

- Every critical workflow action must be written to an immutable audit log.
- The system must preserve chain-of-custody history for investigation records and evidence actions.
- Supervisory decisions (closure approvals/rejections and escalated case reviews) must be persisted in dedicated records.
- Audit logging must be transactionally coupled with state-changing operations so committed actions are always traceable.

## 3. Data Integrity

- Multi-step business operations must execute atomically with explicit transaction handling (`setAutoCommit(false)`, `commit`, `rollback`).
- Referential integrity must be enforced through database foreign key constraints.
- Invalid case-state transitions must be rejected through domain-level validation guards.
- User input must be validated before persistence (e.g., required fields, valid enum/state values, role checks).

## 4. Reliability and Consistency

- The system must maintain consistent data under partial failures by rolling back unsuccessful transactional workflows.
- Business workflows must avoid partial commits that leave related records out of sync.

## 5. Maintainability

- The architecture must remain layered (`Controller -> Service -> Repository -> Database`) to isolate responsibilities and reduce coupling.
- Business rules must remain centralized in service/domain layers rather than duplicated in UI code.
- Persistence contracts should remain interface-driven to allow implementation changes with minimal impact.

## 6. Testability

- Service and repository logic should be testable in isolation via interface-based design and dependency inversion.
- Core business logic should be structured to allow validation without requiring UI interaction.

## 7. Portability and Deployability

- Database configuration must support environment-based deployment (system properties, environment variables, or properties file) without code changes.
- Database schema setup must be reproducible through ordered migration scripts.

## 8. Usability (Operational)

- The UI should enforce role-appropriate visibility and actions to reduce misuse and user error.
- Screen navigation should avoid stale state by loading fresh screen instances for workflow-critical modules.

## Notes

- These NFRs are extracted from project documentation, including `README.md`, `Architecture Rationale.md`, and `RubricMapping.md`.
- This list includes both explicitly stated NFRs and architecture-level quality attributes that are clearly implied by the implementation and rationale.

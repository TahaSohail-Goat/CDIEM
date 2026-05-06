# Architecture Rationale — CDIEM

> **CDIEM** — Crime & Digital Investigation Evidence Management System
> A JavaFX desktop application backed by Microsoft SQL Server.

---

## 1. Overview

CDIEM is a role-gated desktop application for managing criminal investigation cases, digital evidence, audit trails, and supervisory approvals. The system supports three user roles — **Investigating Officer (OFFICER)**, **Digital Forensic Analyst (ANALYST)**, and **Supervisory Authority (SUPERVISOR)** — each with a distinct set of permitted modules and workflow actions.

The architecture is deliberately **layered and use-case-driven**, chosen to reflect the strict role boundaries, transactional integrity requirements, and auditability obligations that the domain imposes.

---

## 2. Core Architectural Style: Layered Architecture (N-Tier)

The codebase is organised into four well-defined, vertically-stacked layers:

```
┌──────────────────────────────────────────────────┐
│  Presentation Layer  (JavaFX FXML + Controllers) │
├──────────────────────────────────────────────────┤
│  Service Layer       (Business Logic / Services) │
├──────────────────────────────────────────────────┤
│  Repository Layer    (Data Access / JDBC)        │
├──────────────────────────────────────────────────┤
│  Database            (Microsoft SQL Server)      │
└──────────────────────────────────────────────────┘
```

Each layer **only ever calls the layer directly beneath it**. Controllers call Services; Services call Repositories; Repositories call the database. This separation was chosen because:

- **Domain complexity is isolated.** Business rules (e.g., "a FROZEN case cannot be modified," "only a SUPERVISOR can approve closure") live exclusively in the service and model layers. The UI never enforces logic — it only renders results.
- **Testability.** Services and repositories can be unit-tested with mock collaborators without touching the UI or a live database.
- **Maintainability.** A change to the SQL schema only requires changes in one repository implementation; the service and controller layers are unaffected.

---

## 3. Package Structure

```
com.project
├── CaseManagementApplication.java   ← JavaFX entry point
├── CaseManagementLauncher.java       ← Module-safe launcher shim
├── controller/                       ← Presentation layer
├── service/                          ← Business logic layer
├── repository/                       ← Data access layer
├── model/                            ← Domain entities & enums
└── util/                             ← Cross-cutting utilities
```

### `model/` — Domain Objects
Plain Java objects (POJOs) and enums that represent the core domain:

| Class / Enum | Role |
|---|---|
| `Case` | Central domain entity with lifecycle state and SLA data |
| `Evidence` | Digital evidence attached to a case |
| `User` | System user with an assigned role |
| `UserRole` | Enum: `OFFICER`, `ANALYST`, `SUPERVISOR` |
| `CaseState` | Enum encoding the case lifecycle state machine |
| `SeverityLevel` | Enum: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` |
| `PriorityState` | Enum: `STANDARD`, `PRIORITY`, `ESCALATED`, `UNDER_ACTIVE_REVIEW` |
| `AuditLog` | Immutable audit record for the chain-of-custody log |
| `NotificationRecord` | In-system notification message |

**Why plain objects?** No ORM framework (like Hibernate) was introduced. Plain JDBC with explicit SQL keeps the queries fully visible, predictable, and efficient — crucial for a system where evidence integrity and transactional correctness are non-negotiable.

---

## 4. Presentation Layer — JavaFX MVC with FXML

**Technology:** JavaFX 17 + FXML + per-screen CSS stylesheets.

Each screen in the application is composed of three files:

| File | Purpose |
|---|---|
| `*.fxml` | Declarative XML layout (the View) |
| `*.css` | Screen-specific styles + `theme.css` global theme |
| `*Controller.java` | Event handler + data binding logic (the Controller) |

**`AppNavigator`** — a final utility class — centralises all screen transitions. It:

1. Loads the FXML file into memory.
2. Retrieves the controller instance from the `FXMLLoader`.
3. Injects the authenticated `User` object into the controller via `setCurrentUser(User)`.
4. Sets the scene, stylesheet, and window dimensions on the primary `Stage`.

**Why this approach?**

- **Single responsibility.** Every controller handles exactly one screen. No controller ever directly creates another screen — it delegates to `AppNavigator`, keeping navigation logic in one place.
- **User context propagation.** Instead of a global session singleton, the current `User` object is passed explicitly into each controller. This makes it immediately visible which user triggered a workflow, and simplifies role-access checks.
- **Stateless screens.** Each screen is freshly loaded on navigation (no caching). This avoids stale UI state, which is important in an investigation management context where data changes continuously.

### Role-Based UI Filtering

`DashboardController` dynamically shows or hides module cards based on the logged-in user's role using the `hasAccess(UserRole, String)` switch expression. This enforces UI-level role boundaries alongside the service-level enforcement — defence in depth.

---

## 5. Service Layer — Business Logic

The service layer contains all business rules. Services are instantiated by controllers (or by other services) using constructor injection of their required repositories.

### Key Services

| Service | Responsibility |
|---|---|
| `AuthService` | Credential validation (SHA-256 password hashing via `PasswordUtil`) |
| `SignUpService` | New user registration with uniqueness validation |
| `CaseService` | Orchestrates case registration, severity updates, reassignment, and deletion |
| `CaseStateTransitionService` | Manages transitions between `CaseState` values |
| `CaseSubmissionService` | Submits a case for supervisor review |
| `CaseClosureService` | Handles supervisor approval/rejection of case closure |
| `EvidenceService` | Evidence upload, integrity verification, and tamper marking |
| `EscalatedCaseReviewService` | Supervisor review of SLA-breached escalated cases |
| `NotificationService` | Creates in-system notifications for workflow events |
| `AuditLogService` | Writes and reads immutable audit records |
| `ChainOfCustodyLog` | Composes human-readable audit messages for all workflow events |
| `HashService` | SHA-256 hash generation and comparison for evidence integrity |
| `SecureFileStorage` | Copies evidence files into a managed storage directory |
| `SLAManager` | Maps `SeverityLevel` → SLA hours and `PriorityState` |
| `ReportService` | Builds, aggregates, and exports (CSV / PDF) summary reports |

### Why Fine-Grained Services?

Each service is narrowly scoped to a single business capability. This reflects the **Single Responsibility Principle** and maps directly to the system's Use Cases. For example, `CaseClosureService` exists solely to manage the supervisor's approve/reject workflow, and nothing else touches that logic.

### Transactional Integrity

Services that perform multi-step database mutations explicitly manage transactions:

```java
// Pattern used throughout the service layer
connection.setAutoCommit(false);
try {
    // Step 1: Validate and read
    // Step 2: Persist changes
    // Step 3: Write audit log
    // Step 4: Dispatch notifications
    connection.commit();
} catch (Exception e) {
    connection.rollback();
    throw ...;
}
```

This ensures that a partial failure (e.g., the audit log fails to write) cannot leave the database in an inconsistent state. ACID compliance is enforced at the application level because no ORM transaction manager was introduced.

---

## 6. Repository Layer — Repository Pattern with Interface/Implementation Split

Every entity has a **Java interface** and a **JDBC implementation**:

```
CaseRepository (interface)
    └── CaseRepositoryImpl (PreparedStatement-based SQL)

EvidenceRepository (interface)
    └── EvidenceRepositoryImpl

UserRepository (interface)
    └── UserRepositoryImpl

AuditRepository (interface)
    └── AuditRepositoryImpl

NotificationRepository (interface)
    └── NotificationRepositoryImpl

CaseClosureDecisionRepository (interface)
    └── CaseClosureDecisionRepositoryImpl

EscalatedCaseReviewRepository (interface)
    └── EscalatedCaseReviewRepositoryImpl
```

**Why interface + implementation?**

1. **Dependency inversion.** Services depend on the `CaseRepository` interface, not on `CaseRepositoryImpl`. This makes it straightforward to substitute a mock repository in unit tests without touching the service code.
2. **Explicit contracts.** The interface documents exactly what data operations a given domain entity supports — it serves as a readable specification of the persistence boundary.
3. **Future portability.** If the database were ever replaced, only the `*Impl` classes would need rewriting.

**Connection passing:** All repository methods accept a `java.sql.Connection` parameter. This is a deliberate design choice — the **caller** (the service) owns the connection and controls the transaction boundary. Repositories are stateless and do not manage their own connections.

---

## 7. Utility Layer

| Utility | Role |
|---|---|
| `DBConnection` | Opens SQL Server connections; resolves credentials from `config/db.properties`, environment variables, or JVM system properties (priority order) |
| `AppNavigator` | Centralised screen navigation and `Stage` management |
| `PasswordUtil` | SHA-256 hash of plaintext passwords for login |
| `IdGenerator` | Generates unique string IDs for certain entities |

`DBConnection` uses a **priority-based credential resolution** strategy:
```
System properties → Environment variables → config/db.properties file
```
This makes the application deployable in different environments (CI, staging, production) without code changes.

---

## 8. Architectural Patterns Used

### 8.1 Template Method Pattern — `AbstractCaseWorkflowController`

All case-workflow controllers (`CaseRegistrationController`, `ReassignmentController`, `SeverityUpdateController`, `CaseDeletionController`, etc.) extend `AbstractCaseWorkflowController`. The abstract class provides:

- Shared validation helpers (`validateCaseRegistration`, `requireUser`, `requireCase`)
- Role-check helpers (`requireUserWithRole`, `validateAssignedOfficer`)
- Transaction helpers (`openConnection`, `rollbackQuietly`, `wrapException`)
- Shared repository references (`caseRepository`, `userRepository`)

This avoids duplicating validation and error-handling code across 27+ controller files, and guarantees that all case workflows enforce the same pre-conditions.

### 8.2 State Pattern — `Case` Domain Object

The `Case` model encodes its own lifecycle state machine. Transition guards are methods on the object itself:

```java
case.validateActiveState();          // Rejects CLOSED / FROZEN
case.validateEvidenceUploadState();  // Only CASE_CREATED / EVIDENCE_UPLOADED / CASE_REASSIGNED
case.validateForensicReviewState();  // Only FORENSIC_REVIEW
case.validateSupervisorReviewState(); // Only SUPERVISOR_REVIEW
case.validateFreezableState();       // Only evidence/review workflow states
```

Each service calls the appropriate guard before persisting a transition. This puts state rules inside the domain object that owns that state, rather than scattering `if` statements across services.

The valid lifecycle is:
```
CASE_CREATED → EVIDENCE_UPLOADED → FORENSIC_REVIEW → SUPERVISOR_REVIEW → CLOSED
                                         ↕ FROZEN ↕
                   CASE_REASSIGNED ←→ (any active state)
```

### 8.3 Repository Pattern

Already described in §6. All database interactions are abstracted behind interface contracts.

### 8.4 Facade Pattern — Screen-Level Services

`CaseService`, `EvidenceService`, and `CaseStateTransitionService` act as **facades** for their respective module screens. Rather than a screen's controller wiring up 5-6 repositories directly, it calls one service that delegates to the appropriate sub-controller or repository. For example, `CaseService` internally wires `CaseRegistrationController`, `SeverityUpdateController`, `ReassignmentController`, and `CaseDeletionController`, then exposes four clean methods: `registerCase`, `updateSeverityLevel`, `reassignOfficer`, `deleteCase`.

### 8.5 Singleton Pattern — `AuditLogService`

`AuditLogService` exposes a `getInstance()` static method returning a shared default instance used by workflow helpers that do not require custom repository injection. Constructor injection is still supported for testing, making this a *testable* singleton.

### 8.6 Strategy / Value Object Pattern — `SLAManager`

`SLAManager` is a stateless strategy object. Given a `SeverityLevel`, it deterministically returns:
- An SLA threshold in hours (`LOW`=120h, `MEDIUM`=72h, `HIGH`=24h, `CRITICAL`=8h)
- The corresponding `PriorityState`

This keeps the mapping rules in exactly one place, used by both case registration and severity updates.

### 8.7 Chain of Responsibility / Decorator — `ChainOfCustodyLog`

`ChainOfCustodyLog` wraps `AuditLogService` and composes structured, human-readable audit messages for each type of workflow event (severity change, reassignment, evidence upload, tamper marking, freeze, reopen, etc.). This decouples the *format* of audit entries from the *mechanics* of persisting them.

---

## 9. Database Architecture

**DBMS:** Microsoft SQL Server (via `mssql-jdbc 13.4.0`).

Schema is managed through **sequential numbered migration scripts** in `/database/`:

```
001_manage_case_schema.sql              ← Core tables: Users, Cases, AuditLogs, Notifications
002_manage_case_module1_migration.sql   ← Seed data and initial officer assignments
003_auth_and_dashboard_structure.sql    ← Username, Email, PasswordHash columns on Users
004_cleanup_legacy_seed_data.sql        ← Data corrections
005–013_*.sql                           ← Incremental schema additions (Evidence, closure
                                           decisions, escalated reviews, state transitions, etc.)
```

**Why numbered migrations?**
The system evolved over six development phases. Numbered scripts create an explicit, ordered history of every schema change. Any developer (or CI pipeline) can apply them in sequence to reach the current schema from a blank database.

**Core tables:**

| Table | Purpose |
|---|---|
| `Users` | User accounts with role (`OFFICER`, `ANALYST`, `SUPERVISOR`) |
| `Cases` | Case records with status, severity, SLA, and officer assignment |
| `Evidence` | Digital evidence files with hash and integrity status |
| `AuditLogs` | Append-only chain-of-custody log |
| `Notifications` | In-system messages linking sender, recipient, and case |
| `CaseClosureDecisions` | Supervisor approve/reject records for case closure |
| `EscalatedCaseReviews` | Supervisor review records for SLA-breached escalated cases |

All foreign key constraints are enforced at the database level in addition to the Java-layer validation, creating a second barrier against data corruption.

---

## 10. Security Design

| Concern | Mechanism |
|---|---|
| **Authentication** | SHA-256 hash stored in `PasswordHash`; login accepts username or email |
| **Role-based access control** | Enforced at UI (dashboard card visibility), service (role checks before mutations), and database (role column constraint) |
| **Evidence integrity** | SHA-256 hash stored at upload time; `HashService` recalculates on-demand for tamper detection |
| **File storage** | Evidence files copied to `storage/evidence/case-{id}/` with timestamped + UUID filenames to prevent collisions and path traversal |
| **Audit trail** | Every workflow event is written to `AuditLogs` within the same transaction as the state change — the log cannot be missing if the action committed |
| **SQL injection** | All queries use `PreparedStatement` with parameter binding; no string concatenation of user inputs |

---

## 11. Why These Choices Were Made

| Decision | Rationale |
|---|---|
| **JavaFX (desktop, not web)** | Investigation workflow tools are typically used on secure, managed workstations, not browsers. A desktop app avoids web server infrastructure and simplifies access control. |
| **Plain JDBC (no ORM)** | Evidence integrity and transactional correctness demand predictable, explicit SQL. ORM magic (lazy loading, dirty tracking) would obscure what is happening at the database level in a system where auditability is critical. |
| **Microsoft SQL Server** | Enterprise-grade RDBMS with strong ACID guarantees, row-level locking, and `DATETIME2` precision — suitable for a legal evidence chain. |
| **Repository interface split** | Enables testing services with mock repositories without spinning up a database. Keeps the persistence contract explicit and separate from implementation details. |
| **Connection passed as parameter** | Services own transaction boundaries. If repositories managed their own connections, multi-step workflows (save case + write audit log + send notification) could not be wrapped in a single atomic transaction. |
| **`AbstractCaseWorkflowController`** | 27 workflow controllers share common validation and error-handling logic. Without the abstract base class, the same 5-6 helper methods would be copy-pasted across every controller — a maintenance liability. |
| **State guards on the `Case` model** | Business rules about valid state transitions belong to the domain object that owns that state. Putting guards in the `Case` model means a controller can never forget to check a precondition — the model enforces it. |
| **Numbered SQL migration scripts** | Provide a reproducible, auditable path from empty database to current schema — required for a system that must demonstrate traceability of design decisions. |
| **`AppNavigator` as navigation hub** | Centralises screen creation in one class. Controllers never `new`-up screens directly. This prevents controllers from becoming tightly coupled to one another and makes navigation logic easy to locate and change. |

---

## 12. Summary Diagram

```
                    ┌─────────────────────────────────┐
                    │       JavaFX FXML Views          │
                    │  (*.fxml + *.css per screen)     │
                    └────────────┬────────────────────-┘
                                 │ AppNavigator (navigation)
                    ┌────────────▼────────────────────-┐
                    │     UI Controllers (27)          │
                    │  extends AbstractCaseWorkflow-   │
                    │  Controller (for case modules)   │
                    └────────────┬─────────────────────┘
                                 │ calls
           ┌─────────────────────▼──────────────────────────────┐
           │                  Services (36)                      │
           │  AuthService | CaseService | EvidenceService        │
           │  NotificationService | AuditLogService              │
           │  ChainOfCustodyLog | SLAManager | ReportService ... │
           └──────┬─────────────────────────────────────────────┘
                  │ uses (Connection parameter passed down)
           ┌──────▼────────────────────────────────────────┐
           │           Repositories (7 interfaces)         │
           │  CaseRepository | EvidenceRepository          │
           │  UserRepository | AuditRepository             │
           │  NotificationRepository | ...                 │
           └──────┬────────────────────────────────────────┘
                  │ PreparedStatement / JDBC
           ┌──────▼──────────────────────┐
           │  Microsoft SQL Server       │
           │  (CDIEM database)           │
           └─────────────────────────────┘
```

---

*Document prepared for CDIEM — Software Design and Architecture, Semester 4, Spring 2026.*

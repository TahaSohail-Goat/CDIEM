    # Rubric Mapping — CDIEM Project

> **CDIEM** — Crime & Digital Investigation Evidence Management System
> Software Design and Architecture — Spring 2026

---

## Quick Reference

| # | Criterion | Max Marks | Status |
|---|-----------|-----------|--------|
| 1 | Documentation | 10 | ✅ Covered |
| 2 | Use Cases | 18 | ✅ Covered |
| 3 | User Interface | 10 | ✅ Covered |
| 4 | OOP Principles in Business Logic | 10 | ✅ Covered |
| 5 | Business Logic Layer Functionality | 10 | ✅ Covered |
| 6 | Database | 10 | ✅ Covered |
| 7 | Architecture | 5 + 15 | ✅ Covered |
| 8 | Integration b/w UI, BL and DB | 12 | ✅ Covered |
| 9 | Consistency b/w Design and Code | 10 | ✅ Covered |
| 10 | Design Patterns (GRASP & GoF) | 10 | ✅ Covered |
| 11 | Non-functional Requirements | 10 | ✅ Covered |
| 12 | Code | 5 + 5 | ✅ Covered |
| | **Total** | **140** | |

---

## 1. Documentation (10 marks)

**Requirement:** Detailed documentation following given template. All document sections complete with readable images of models & Readme File.

### Evidence in Project

| Artifact | Location | Purpose |
|----------|----------|---------|
| `README.md` | Project root | Full project overview, capabilities, roles, use cases, architecture, domain model, tech stack, DB schema, config, seeded accounts, run instructions, security notes |
| `Architecture Rationale.md` | Project root | 368-line document covering layered architecture rationale, package structure, presentation/service/repository/utility layers, design patterns, DB architecture, security design |
| `UseCase-to-System-Mapping.txt` | Project root | 952-line traceability document mapping all 15 use cases + extensions to exact classes, with workflow descriptions |
| `Diagrams.zip` | Project root | Contains model diagrams (Package, Deployment, Component, etc.) |
| `SDs/` | Project root | Sequence Diagrams directory |
| `database/*.sql` | `database/` | 13 numbered migration scripts serving as schema documentation |

### How It Maps

- **README.md** covers: overview, capabilities, supported roles, core use cases, workflow summary, architecture description, domain model, technology stack, database schema and migrations, configuration, seeded accounts, running instructions, security notes, and repository notes.
- **Architecture Rationale.md** covers: architectural style justification, package structure, layer-by-layer rationale, design patterns used (Template Method, State, Repository, Facade, Singleton, Strategy, Chain of Responsibility), database architecture, and security design decisions.
- **UseCase-to-System-Mapping.txt** provides class-level traceability for all 15 use cases plus extensions.
- **Diagrams.zip** contains readable model images.

---

## 2. Use Cases (18 marks)

**Requirement:** All use cases assigned to group members are proper functional and completed.

### Implemented Use Cases

| UC | Name | Actor | Primary Controller | Service | Status |
|----|------|-------|--------------------|---------|--------|
| UC1 | Case Registration | Officer | `CaseRegistrationController` | `CaseService` | ✅ Functional |
| UC2 | Upload Digital Evidence | Officer | `EvidenceUploadController` | `EvidenceService` | ✅ Functional |
| UC3 | Verify Evidence Integrity | Analyst | `IntegrityVerificationController` | `EvidenceService` | ✅ Functional |
| UC4 | Freeze Case (Tampered Evidence) | Analyst | `CaseFreezeController` | `CaseStateTransitionService` | ✅ Functional |
| UC5 | Approve Case Closure | Supervisor | `CaseClosureController` | `CaseClosureService` | ✅ Functional |
| UC6 | Update Case Severity Level | Supervisor | `SeverityUpdateController` | `CaseService` | ✅ Functional |
| UC7 | Mark Evidence as Tampered | Analyst | `TamperMarkController` | `EvidenceService` | ✅ Functional |
| UC8 | Reopen Frozen Case | Supervisor | `ReopenController` | `CaseStateTransitionService` | ✅ Functional |
| UC9 | Mark Evidence as Verified | Analyst | `VerificationMarkController` | `EvidenceService` | ✅ Functional |
| UC10 | Generate Summary Report | Supervisor | `ReportController` | `ReportService` | ✅ Functional |
| UC11 | Submit Case for Supervisor Review | Officer | `SubmissionController` | `CaseSubmissionService` | ✅ Functional |
| UC12 | Reassign Investigating Officer | Supervisor | `ReassignmentController` | `CaseService` | ✅ Functional |
| UC13 | View Chain-of-Custody Log | Supervisor | `LogViewerController` | `ChainOfCustodyViewerService` | ✅ Functional |
| UC14 | Review Escalated Case | Supervisor | `EscalatedCaseController` | `EscalatedCaseReviewService` | ✅ Functional |
| UC15 | Reject Case Closure | Supervisor | `RejectionController` | `CaseClosureService` | ✅ Functional |

### Beyond UC1–UC15

- **Sign-up workflow** — `SignUpController` + `SignUpService`
- **Supervisor case deletion** — `CaseDeletionController` + `CaseDeletionResult`
- **Notification viewing** — `NotificationController` + `NotificationService`

All 15 core use cases have dedicated controller + service implementations with full workflows documented in `UseCase-to-System-Mapping.txt`.

---

## 3. User Interface (10 marks)

**Requirement:** Fully functional user interface created using JavaFX. All screens and events working correctly. Complete mapping to SDs, SSD/Expanded use case.

### FXML Screens (23 files)

| Screen | FXML File | CSS File | Controller |
|--------|-----------|----------|------------|
| Login | `login.fxml` | `login.css` | `LoginController` |
| Sign Up | `signup.fxml` | — | `SignUpController` |
| Dashboard | `dashboard.fxml` | `dashboard.css` | `DashboardController` |
| Manage Case | `manage_case.fxml` | `manage_case.css` | `CaseController` |
| Manage Evidence | `manage_evidence.fxml` | `manage_evidence.css` | `EvidenceController` |
| State Transitions | `manage_state_transitions.fxml` | `manage_state_transitions.css` | `CaseStateTransitionController` |
| Case Closure | `manage_case_closure.fxml` | `manage_case_closure.css` | `ManageCaseClosureController` |
| Submit for Review | `submit_review.fxml` | — | `SubmitReviewController` |
| Chain of Custody | `chain_of_custody_log.fxml` | `chain_of_custody_log.css` | `LogViewerController` |
| Review Escalated Case | `review_escalated_case.fxml` | `review_escalated_case.css` | `ReviewEscalatedCaseController` |
| Summary Report | `summary_report.fxml` | `summary_report.css` | `ReportController` |
| Notifications | `notifications.fxml` | `notifications.css` | `NotificationController` |
| Global Theme | — | `theme.css` | — |

### UI Architecture

- **`AppNavigator`** centralises all screen transitions, loads FXML, injects the authenticated `User`, and sets stylesheets.
- **`DashboardController`** dynamically shows/hides module cards based on `UserRole` using `hasAccess(UserRole, String)`.
- Each screen is freshly loaded on navigation (stateless screens — no stale UI state).
- Role-based UI filtering provides defence-in-depth alongside service-level role checks.

---

## 4. OOP Principles in Business Logic (10 marks)

**Requirement:** Well encapsulated classes. Program takes full advantage of inheritance, abstraction and polymorphism where appropriate.

### Encapsulation

| Example | Where |
|---------|-------|
| `Case` model — private fields with getters/setters, state validation methods (`validateActiveState()`, `validateFreezableState()`, etc.) encapsulate lifecycle rules | `model/Case.java` |
| `Evidence` model — private hash fields, verification snapshot flags, status encapsulated behind accessor methods | `model/Evidence.java` |
| `User` model — role and credentials encapsulated with controlled access | `model/User.java` |
| Repository implementations hide SQL details behind interface contracts | `repository/*Impl.java` |

### Inheritance

| Example | Where |
|---------|-------|
| `AbstractCaseWorkflowController` — abstract base class providing shared validation (`validateCaseRegistration`, `requireUser`, `requireCase`), role checks (`requireUserWithRole`, `validateAssignedOfficer`), transaction helpers (`openConnection`, `rollbackQuietly`, `wrapException`), and shared repository references | `controller/AbstractCaseWorkflowController.java` |
| 12+ controllers extend `AbstractCaseWorkflowController`: `CaseRegistrationController`, `ReassignmentController`, `SeverityUpdateController`, `CaseDeletionController`, `EvidenceUploadController`, `IntegrityVerificationController`, `TamperMarkController`, `VerificationMarkController`, `CaseFreezeController`, `ReopenController`, `CaseClosureController`, `RejectionController`, `SubmissionController`, `EscalatedCaseController` | `controller/*.java` |

### Abstraction

| Example | Where |
|---------|-------|
| 7 repository interfaces (`CaseRepository`, `EvidenceRepository`, `UserRepository`, `AuditRepository`, `NotificationRepository`, `CaseClosureDecisionRepository`, `EscalatedCaseReviewRepository`) — services depend on interfaces, not implementations | `repository/*.java` |
| Enums as abstract domain contracts: `CaseState`, `SeverityLevel`, `PriorityState`, `EvidenceStatus`, `UserRole`, `CaseClosureDecisionType` | `model/*.java` |

### Polymorphism

| Example | Where |
|---------|-------|
| Repository interface/implementation split — `CaseRepository` ↔ `CaseRepositoryImpl` allows substitution (e.g., mock for testing) | `repository/` |
| `Case` state validation methods — polymorphic-style guard methods (`validateActiveState()`, `validateEvidenceUploadState()`, `validateForensicReviewState()`, `validateSupervisorReviewState()`, `validateFreezableState()`) dispatch different validation logic based on current state | `model/Case.java` |
| Enum behavior — `UserRole`, `CaseState`, `EvidenceStatus` carry display labels and provide type-safe branching | `model/*.java` |

---

## 5. Business Logic Layer Functionality (10 marks)

**Requirement:** Program is logically well designed, displays correct output, executes correctly with no syntax or runtime errors.

### Service Layer (37 classes)

The `service/` package contains 37 classes implementing all business logic:

| Category | Services |
|----------|----------|
| Authentication | `AuthService`, `SignUpService`, `SignUpException` |
| Case Management | `CaseService`, `CaseStateTransitionService`, `CaseSubmissionService`, `CaseClosureService` |
| Evidence | `EvidenceService`, `HashService`, `SecureFileStorage` |
| Audit & Logging | `AuditLogService`, `ChainOfCustodyLog`, `ChainOfCustodyViewerService`, `LogService` |
| Notifications | `NotificationService` |
| Escalation | `EscalatedCaseReviewService` |
| Reporting | `ReportService` (CSV + PDF export) |
| Domain Logic | `SLAManager` |
| Result/Snapshot DTOs | 15 result and snapshot objects |

### Key Business Rules Enforced

- Case state machine transitions are guarded by `Case` model methods.
- Evidence integrity uses SHA-256 hash comparison (`HashService`).
- SLA calculation is deterministic via `SLAManager` (`LOW`=120h, `MEDIUM`=72h, `HIGH`=24h, `CRITICAL`=8h).
- Role-based access enforced at service level before any mutation.
- Transactional integrity with explicit `connection.setAutoCommit(false)`, `commit()`, and `rollback()`.

---

## 6. Database (10 marks)

**Requirement:** Tables named correctly. Exception Handling. Data properly read, written, updated and deleted in tables. Input validation for all values taken from the user.

### Database Tables (7 core tables)

| Table | Purpose | CRUD Operations |
|-------|---------|-----------------|
| `Users` | User accounts with roles | Create (sign-up), Read (auth, lookup), Update (n/a), Delete (n/a) |
| `Cases` | Case records with status, severity, SLA | Create (UC1), Read (all UCs), Update (UC4–UC12, UC14, UC15), Delete (supervisor extension) |
| `Evidence` | Digital evidence with hash & status | Create (UC2), Read (UC3, UC5, UC7, UC9, UC11), Update (UC3, UC7, UC9), Delete (supervisor extension) |
| `AuditLogs` | Immutable chain-of-custody log | Create (all workflow UCs), Read (UC13), Update (never — immutable), Delete (only cascade on case delete) |
| `Notifications` | In-system messages | Create (UC5, UC8, UC11, UC12), Read (notification view), Update (mark read), Delete (cascade) |
| `CaseClosureDecisions` | Supervisor approve/reject records | Create (UC5, UC15), Read (closure view), Delete (cascade) |
| `EscalatedCaseReviews` | SLA-breached case reviews | Create (UC14), Read (escalation view), Delete (cascade) |

### Schema Management

- **13 numbered migration scripts** in `database/` applied in sequence.
- Foreign key constraints enforced at DB level alongside Java-layer validation.
- All queries use `PreparedStatement` with parameter binding (SQL injection prevention).

### Exception Handling

- All repository methods accept `Connection` as parameter; services manage transaction boundaries.
- `try/catch/finally` with `rollback()` on failure throughout service layer.
- Input validation in controllers before service calls (non-blank fields, valid enum values, role checks).

---

## 7. Architecture (5 + 15 = 20 marks)

**Requirement:** Choosing an architectural style well-suited for the project. All 3 diagrams (Package, Deployment, Component) designed correctly.

### Architectural Style: Layered Architecture (N-Tier)

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

### Package Structure

```
com.project
├── CaseManagementApplication.java    ← JavaFX entry point
├── CaseManagementLauncher.java       ← Module-safe launcher
├── controller/  (27 files)           ← Presentation layer
├── service/     (37 files)           ← Business logic layer
├── repository/  (14 files)           ← Data access layer
├── model/       (16 files)           ← Domain entities & enums
└── util/        (4 files)            ← Cross-cutting utilities
```

### Why Layered Architecture

- **Domain complexity isolation** — business rules live exclusively in service/model layers.
- **Testability** — services and repositories can be tested with mock collaborators.
- **Maintainability** — schema changes only require repository-layer changes.
- Each layer only calls the layer directly beneath it.

### Diagrams

- `Diagrams.zip` contains Package, Deployment, and Component diagrams.
- `Architecture Rationale.md` provides detailed rationale and a summary ASCII diagram.

---

## 8. Integration b/w UI, BL and DB (12 marks)

**Requirement:** All 3 layers (UI, BL & DB) implemented as separate packages/layers. All layers fully integrated and functioning correctly.

### Layer Separation

| Layer | Package | File Count | Responsibility |
|-------|---------|------------|----------------|
| **UI (Presentation)** | `controller/` + `resources/view/` | 27 controllers + 23 FXML/CSS | User interaction, event handling, data display |
| **BL (Business Logic)** | `service/` + `model/` | 37 services + 16 models | Business rules, validation, workflow orchestration |
| **DB (Data Access)** | `repository/` | 14 files (7 interfaces + 7 implementations) | CRUD operations via JDBC |

### Integration Flow

```
Controller → Service → Repository → SQL Server
     ↑          ↑          ↑
   FXML      Model      Connection
```

- Controllers instantiate services and call business methods.
- Services instantiate repositories and manage transaction boundaries.
- Repositories accept `Connection` parameter — caller owns the transaction.
- `AppNavigator` handles all screen transitions and user context propagation.
- `DBConnection` provides centralized connection management.

### Key Integration Points

- **User context flows** from `LoginController` → `AppNavigator.setCurrentUser()` → every subsequent controller.
- **Transaction boundaries** are owned by services: `setAutoCommit(false)` → multiple repository calls → `commit()` or `rollback()`.
- **Result objects** (e.g., `CaseClosureApprovalResult`, `EvidenceUploadResult`) carry outcomes from service layer back to UI layer.

---

## 9. Consistency b/w Design and Code (10 marks)

**Requirement:** 100% mapping of design to code. All SDs must properly map into code.

### Traceability

- **`UseCase-to-System-Mapping.txt`** (952 lines) provides exhaustive class-to-use-case mapping for all 15 UCs.
- Each UC entry lists: primary classes, supporting classes, and step-by-step "how the implementation works."
- Section 4 ("Master Class to Use-Case Index") accounts for **every Java source file** in `src/main/java`.
- Document-vs-code differences are explicitly documented (UC3, UC9, UC13, UC14, UC15).

### Design-to-Code Consistency Summary

| Design Element | Code Realization |
|----------------|-----------------|
| 3 Actor roles (Officer, Analyst, Supervisor) | `UserRole` enum with `OFFICER`, `ANALYST`, `SUPERVISOR` |
| 15 Use Cases | 15 dedicated controller+service pairs |
| Case lifecycle state machine | `CaseState` enum + guard methods on `Case` model |
| Evidence integrity workflow | `HashService` (SHA-256) + `EvidenceStatus` enum |
| Audit trail / chain-of-custody | `AuditLog` model + `AuditLogService` + `ChainOfCustodyLog` |
| SLA management | `SLAManager` strategy + `PriorityState` + `SeverityLevel` |
| Notification system | `NotificationService` + `NotificationRecord` + `NotificationRepository` |

### Known Document-vs-Code Deviations

| UC | Deviation |
|----|-----------|
| UC3 | PDF describes UC3 as both verifying and marking; code splits into UC3 (snapshot) + UC7/UC9 (decision) |
| UC9 | PDF says UC9 should move case to Supervisor Review; code keeps case in Forensic Review until UC11 |
| UC13 | PDF implies a formal "Under Supervisory Inspection" state; code uses an inspection-mode flag instead |
| UC14 | PDF implies notification to parties; code records review but no notification wired |
| UC15 | PDF says rejection reason is optional; code requires a nonblank reason |

---

## 10. Design Patterns — GRASP & GoF (10 marks)

**Requirement:** Controller, Information Expert, Creator, Low Coupling, High Cohesion and Protected Variation ALL applied correctly. GoF applied where applicable.

### GRASP Patterns

| Pattern | Application | Evidence |
|---------|-------------|----------|
| **Controller** | Every use case has a dedicated controller class (27 controllers). `CaseRegistrationController` handles UC1, `EvidenceUploadController` handles UC2, etc. | `controller/*.java` |
| **Information Expert** | `Case` model owns state validation (`validateActiveState()`, `validateFreezableState()`). `Evidence` model owns hash comparison logic. `SLAManager` owns SLA calculation. | `model/Case.java`, `service/SLAManager.java` |
| **Creator** | Services create domain objects and result DTOs. `CaseRegistrationController` creates `Case` instances. `EvidenceUploadController` creates `Evidence` records. | `controller/*.java`, `service/*.java` |
| **Low Coupling** | Repository interface/implementation split. Services depend on interfaces, not concrete classes. `Connection` passed as parameter (no hidden coupling). | `repository/*.java` |
| **High Cohesion** | Each service is narrowly scoped: `CaseClosureService` only handles closure, `HashService` only handles hashing, `SLAManager` only handles SLA calculation. 37 focused service classes. | `service/*.java` |
| **Protected Variation** | Repository interfaces shield services from DB implementation changes. `AppNavigator` shields controllers from navigation mechanics. Enums (`CaseState`, `UserRole`) protect against invalid state values. | `repository/*.java`, `util/AppNavigator.java` |

### GoF Patterns

| Pattern | Application | Evidence |
|---------|-------------|----------|
| **Template Method** | `AbstractCaseWorkflowController` — abstract base class with shared validation, role checks, transaction helpers. 12+ controllers extend it. | `controller/AbstractCaseWorkflowController.java` |
| **State** | `Case` model encodes lifecycle state machine with guard methods per state. Transitions are validated before persistence. | `model/Case.java`, `model/CaseState.java` |
| **Repository** | 7 interface + implementation pairs abstracting all DB operations. | `repository/*.java` |
| **Facade** | `CaseService`, `EvidenceService`, `CaseStateTransitionService` act as facades for their respective modules, wiring multiple sub-controllers and repositories. | `service/CaseService.java`, `service/EvidenceService.java` |
| **Singleton** | `AuditLogService.getInstance()` — testable singleton with constructor injection fallback. | `service/AuditLogService.java` |
| **Strategy** | `SLAManager` — stateless strategy mapping `SeverityLevel` → SLA hours + `PriorityState`. | `service/SLAManager.java` |
| **Decorator/Chain of Responsibility** | `ChainOfCustodyLog` wraps `AuditLogService`, composing structured human-readable audit messages. Decouples format from persistence. | `service/ChainOfCustodyLog.java` |

---

## 11. Non-functional Requirements (10 marks)

**Requirement:** At least 2 non-functional requirements implemented correctly.

### NFR 1: Security

| Concern | Implementation |
|---------|---------------|
| **Password hashing** | SHA-256 via `PasswordUtil` — passwords never stored in plaintext |
| **Evidence integrity** | SHA-256 hash at upload + on-demand recalculation for tamper detection (`HashService`) |
| **SQL injection prevention** | All queries use `PreparedStatement` with parameter binding |
| **File storage security** | Evidence files stored with timestamped + UUID filenames in `storage/evidence/case-{id}/` to prevent collisions and path traversal |
| **Role-based access control** | Enforced at 3 levels: UI (dashboard card visibility), service (role checks), and database (role column constraint) |

### NFR 2: Auditability / Traceability

| Concern | Implementation |
|---------|---------------|
| **Immutable audit log** | Every workflow event written to `AuditLogs` table within the same transaction as the state change |
| **Chain-of-custody** | `ChainOfCustodyLog` composes human-readable entries for all events (upload, verify, tamper, freeze, reopen, severity change, reassignment, closure, etc.) |
| **Decision traceability** | `CaseClosureDecisions` and `EscalatedCaseReviews` stored in dedicated tables |
| **Transactional integrity** | Audit log writes are part of the same transaction — log cannot be missing if the action committed |

### NFR 3: Data Integrity

| Concern | Implementation |
|---------|---------------|
| **ACID transactions** | Explicit `setAutoCommit(false)` / `commit()` / `rollback()` in all multi-step operations |
| **Foreign key constraints** | Enforced at database level in addition to Java-layer validation |
| **Input validation** | Controllers validate all user inputs before service calls (non-blank, valid enum, role checks) |
| **State machine guards** | `Case` model methods prevent invalid state transitions at the domain level |

---

## 12. Code (5 + 5 = 10 marks)

**Requirement:** Fully functional code. Commented and indented code.

### Code Statistics

| Package | Files | Purpose |
|---------|-------|---------|
| `controller/` | 27 | JavaFX controllers |
| `service/` | 37 | Business logic + result DTOs |
| `repository/` | 14 | 7 interfaces + 7 JDBC implementations |
| `model/` | 16 | Domain entities + enums |
| `util/` | 4 | DB connection, navigation, hashing, ID generation |
| `resources/view/` | 23 | FXML screens + CSS stylesheets |
| `database/` | 14 | SQL migration scripts |
| **Total Java** | **100** | |
| **Total Resources** | **23** | |
| **Total SQL** | **14** | |

### Functionality

- Application boots via `CaseManagementLauncher` → `CaseManagementApplication` (JavaFX).
- Run with `mvn clean javafx:run` or build Windows executable via `scripts/package-windows.ps1`.
- All 15 use cases execute end-to-end through the UI.
- No reported syntax or runtime errors.

### Code Quality

- Consistent indentation throughout Java and FXML files.
- Controller methods are documented with their workflow steps.
- `Architecture Rationale.md` and `UseCase-to-System-Mapping.txt` provide code-level documentation.
- Meaningful class and method naming aligned with domain vocabulary.

---

## Summary Matrix

| # | Criterion | Marks | Key Evidence Files |
|---|-----------|-------|--------------------|
| 1 | Documentation | /10 | `README.md`, `Architecture Rationale.md`, `UseCase-to-System-Mapping.txt`, `Diagrams.zip` |
| 2 | Use Cases | /18 | 15 UC controller+service pairs in `controller/` and `service/` |
| 3 | User Interface | /10 | 23 FXML/CSS files in `resources/view/`, 27 controllers, `AppNavigator` |
| 4 | OOP Principles | /10 | `AbstractCaseWorkflowController`, 7 repository interfaces, `Case` model guards, enums |
| 5 | Business Logic | /10 | 37 service classes, transactional integrity, SLA logic, hash verification |
| 6 | Database | /10 | 7 tables, 13 migrations, `PreparedStatement`, FK constraints, input validation |
| 7 | Architecture | /20 | Layered N-Tier, `Architecture Rationale.md`, `Diagrams.zip` |
| 8 | Integration | /12 | Separate `controller/`, `service/`, `repository/` packages, `AppNavigator`, `DBConnection` |
| 9 | Design Consistency | /10 | `UseCase-to-System-Mapping.txt` (952 lines, every source file accounted for) |
| 10 | Design Patterns | /10 | GRASP (all 6) + GoF (7 patterns): Template Method, State, Repository, Facade, Singleton, Strategy, Decorator |
| 11 | Non-functional Req. | /10 | Security (SHA-256, RBAC, PreparedStatement), Auditability (immutable logs), Data Integrity (ACID) |
| 12 | Code | /10 | 100 Java files, 23 resource files, 14 SQL scripts — functional, commented, indented |
| | **Total** | **/140** | |

---

*Document prepared for CDIEM — Software Design and Architecture, Semester 4, Spring 2026.*

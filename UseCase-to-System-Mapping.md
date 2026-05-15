# Use Case to System Mapping

## 1. How to Read This File

- **Primary classes** are the classes that directly execute the use case.
- **Supporting classes** are repositories, models, result objects, enums, and utilities that make the use case work.
- **Shared classes** are used before or across many use cases, especially for login, navigation, database access, and actor lookup.
- **Extension classes** are current-code features that exist in the implementation but are not named as UC1 to UC15 in the source PDF.
- Where the PDF behavior and the current code differ, the difference is stated explicitly in a **Document vs current code** note.

---

## 2. Global Shared Classes Used Across Many Use Cases

- **CaseManagementApplication** – Boots the JavaFX application and hands control to AppNavigator.
- **AppNavigator** – Loads the scenes and moves the user between login, dashboard, and the workflow modules.
- **LoginController + AuthService** – Implement the authentication precondition that appears in many use cases before a user can access a module.
- **DashboardController** – Serves as the role-based entry point to the modules that implement UC1 to UC15.
- **DBConnection** – Opens the SQL Server connections used by controllers, services, and repositories throughout the workflow.
- **UserRepository + UserRepositoryImpl** – Load users by ID, authenticate users, and provide actor lookup for role checks and notifications.
- **User + UserRole** – Represent the actors named in the PDF and enforce role-based access checks across the workflow.
- **Case** – Central domain model for the overall investigation workflow. It holds case state, severity, priority, ownership, and SLA data, and provides validation/transition methods used throughout the system.
- **CaseState** – Shared state enum used across the full case lifecycle from creation to closure, reopening, freezing, review, and escalation handling.

---

## 2A. Current-Code Extension Outside the PDF UC1 to UC15 Set

Supervisor-only case deletion is now implemented inside the Manage Case module, but it is not a separately numbered use case in the PDF.

### Primary classes
- **CaseController** – Exposes the supervisor-only delete action in the Manage Case UI.
- **CaseService** – Service facade that now also wires the delete workflow.
- **CaseDeletionController** – Direct implementation of deletion. It removes dependent records, deletes the case, records a non-case-bound audit entry, notifies the related officer, and attempts secure evidence-file cleanup.

### Supporting classes
- **CaseDeletionResult** – Returns deletion outcome details to the UI.
- **CaseRepository + CaseRepositoryImpl** – Delete the case record itself.
- **EvidenceRepository + EvidenceRepositoryImpl** – Load and remove evidence records before case deletion.
- **CaseClosureDecisionRepository + CaseClosureDecisionRepositoryImpl + EscalatedCaseReviewRepository + EscalatedCaseReviewRepositoryImpl** – Remove dependent supervisor-review records before case deletion.
- **NotificationService + NotificationRepository + NotificationRepositoryImpl + NotificationRecord** – Remove case-linked notifications and create the post-delete user notification.
- **AuditLogService + AuditRepository + AuditRepositoryImpl + AuditLog** – Remove case-linked audit records and retain the accountability record for the delete action itself.
- **SecureFileStorage** – Attempts to remove stored evidence files after the database delete commits.

### How the implementation works
1. The supervisor triggers delete from the Manage Case module.
2. CaseDeletionController loads the case, assigned user, and evidence file paths.
3. It removes dependent rows that still reference the case.
4. It deletes the case record itself.
5. It creates a general notification for the assigned officer and a non-case-bound audit entry for accountability.
6. After commit, it attempts to delete the stored evidence files.

---

## 3. Detailed Mapping: Use Case by Use Case

### UC1 – Case Registration
**Actor:** Investigating Officer

#### Primary classes
- DashboardController
- CaseController
- CaseRegistrationController
- CaseService

#### Supporting classes
- AbstractCaseWorkflowController
- CaseRepository + CaseRepositoryImpl
- Case
- SeverityLevel + PriorityState + CaseState
- SLAManager
- IdGenerator
- AuditLogService + AuditRepository + AuditRepositoryImpl + AuditLog

#### How the implementation works
1. The officer authenticates through LoginController/AuthService.
2. DashboardController opens the case module.
3. CaseRegistrationController validates the form using the shared helper methods from AbstractCaseWorkflowController.
4. SLAManager computes SLA due time and priority.
5. CaseRepositoryImpl stores the Case.
6. IdGenerator provides the generated display identifier.
7. AuditLogService records the registration action.
8. The UI returns confirmation to the officer.

---

### UC2 – Upload Digital Evidence
**Actor:** Investigating Officer

#### Primary classes
- DashboardController
- EvidenceController
- EvidenceUploadController
- EvidenceService

#### Supporting classes
- AbstractCaseWorkflowController
- CaseRepository + CaseRepositoryImpl
- EvidenceRepository + EvidenceRepositoryImpl
- Evidence
- SecureFileStorage
- HashService
- ChainOfCustodyLog + AuditRepository + AuditLog
- EvidenceUploadResult + EvidenceSnapshot
- EvidenceStatus + CaseState

#### How the implementation works
1. The officer opens the evidence module for a valid case.
2. EvidenceUploadController checks the officer role and case readiness.
3. SecureFileStorage stores the file.
4. HashService computes the SHA-256 hash.
5. EvidenceRepositoryImpl stores the Evidence record.
6. ChainOfCustodyLog records the upload with actor, file details, and hash information.
7. CaseRepositoryImpl updates the case state to "Evidence Uploaded".

---

### UC3 – Verify Evidence Integrity
**Actor:** Digital Forensic Analyst

#### Primary classes
- EvidenceController
- IntegrityVerificationController
- EvidenceService

#### Supporting classes
- AbstractCaseWorkflowController
- EvidenceRepository + EvidenceRepositoryImpl
- CaseRepository + CaseRepositoryImpl
- Evidence
- Case
- HashService
- ChainOfCustodyLog + AuditRepository + AuditLog
- IntegrityVerificationResult + EvidenceSnapshot
- EvidenceStatus + CaseState

#### How the implementation works
1. The analyst opens a case with uploaded evidence.
2. IntegrityVerificationController loads the stored evidence record.
3. HashService recalculates the SHA-256 hash from the stored file.
4. The controller compares original and recalculated hash values.
5. EvidenceRepositoryImpl stores the verification snapshot.
6. ChainOfCustodyLog records the verification action.

#### Document vs current code
- The PDF describes UC3 as the step that both verifies integrity and directly marks the evidence as "Verified" or "Tampered".
- The current code splits that behavior into two stages:
  1. UC3 performs the comparison and stores the verification snapshot.
  2. UC7 or UC9 then finalizes the evidence status.

---

### UC4 – Freeze Case Due to Tampered Evidence
**Actor:** Digital Forensic Analyst

#### Primary classes
- CaseStateTransitionController
- CaseFreezeController
- CaseStateTransitionService

#### Supporting classes
- AbstractCaseWorkflowController
- CaseRepository + CaseRepositoryImpl
- Case
- ChainOfCustodyLog + AuditRepository + AuditLog
- CaseFreezeResult

#### How the implementation works
1. The analyst reviews the case after tamper detection.
2. CaseFreezeController validates the analyst role and case state.
3. Case.triggerFreezeWorkflow() is used to move the case to "Frozen".
4. CaseRepositoryImpl persists the frozen state.
5. ChainOfCustodyLog records the freeze action.

#### Document vs current code
- The current implementation supports both:
  1. A dedicated manual freeze action through CaseFreezeController.
  2. An automatic freeze path triggered by UC7 through TamperMarkController.

---

### UC5 – Approve Case Closure
**Actor:** Supervisory Authority

#### Primary classes
- ManageCaseClosureController
- CaseClosureController
- CaseClosureService

#### Supporting classes
- AbstractCaseWorkflowController
- CaseRepository + CaseRepositoryImpl
- EvidenceRepository + EvidenceRepositoryImpl
- CaseClosureDecisionRepository + CaseClosureDecisionRepositoryImpl
- CaseClosureDecision + CaseClosureDecisionType
- CaseClosureSnapshot + CaseClosureApprovalResult
- Case
- AuditLogService + AuditRepository + AuditLog
- NotificationService + NotificationRepository + NotificationRecord

#### How the implementation works
1. The supervisor opens the closure module for a case in the proper review state.
2. CaseClosureController validates the role and evidence readiness.
3. Case.closeCase() moves the domain object to "Closed".
4. CaseRepositoryImpl stores the new state.
5. CaseClosureDecisionRepositoryImpl stores the approval record.
6. AuditLogService records the action.
7. NotificationService informs the investigating officer.

---

### UC6 – Update Case Severity Level
**Actor:** Supervisory Authority

#### Primary classes
- CaseController
- SeverityUpdateController
- CaseService

#### Supporting classes
- AbstractCaseWorkflowController
- CaseRepository + CaseRepositoryImpl
- Case
- SeverityLevel + PriorityState
- SLAManager
- ChainOfCustodyLog + AuditRepository + AuditLog
- CaseSeverityUpdateResult

#### How the implementation works
1. The supervisor opens an eligible case.
2. SeverityUpdateController validates authority and input.
3. SLAManager recalculates SLA and priority from the new severity.
4. CaseRepositoryImpl saves the updated severity profile.
5. ChainOfCustodyLog records the change as an immutable log entry.

---

### UC7 – Mark Evidence as Tampered
**Actor:** Digital Forensic Analyst

#### Primary classes
- EvidenceController
- TamperMarkController
- EvidenceService

#### Supporting classes
- AbstractCaseWorkflowController
- EvidenceRepository + EvidenceRepositoryImpl
- CaseRepository + CaseRepositoryImpl
- Evidence
- Case
- ChainOfCustodyLog + AuditRepository + AuditLog
- EvidenceDecisionResult + EvidenceSnapshot
- EvidenceStatus + CaseState

#### How the implementation works
1. UC3 must already have stored a verification snapshot.
2. TamperMarkController confirms that original and recalculated hashes do not match.
3. EvidenceRepositoryImpl marks the Evidence as "Tampered".
4. Case.triggerFreezeWorkflow() moves the case to "Frozen".
5. CaseRepositoryImpl stores the frozen state.
6. ChainOfCustodyLog records the decision.

#### Document vs current code
- The PDF frames UC7 as part of the hash recalculation activity.
- The current code expects hash recalculation to have already happened in UC3, and UC7 only finalizes the tampered outcome.

---

### UC8 – Reopen Frozen Case
**Actor:** Supervisory Authority

#### Primary classes
- CaseStateTransitionController
- ReopenController
- CaseStateTransitionService

#### Supporting classes
- AbstractCaseWorkflowController
- CaseRepository + CaseRepositoryImpl
- UserRepository + UserRepositoryImpl
- Case
- ChainOfCustodyLog + AuditRepository + AuditLog
- NotificationService + NotificationRepository + NotificationRecord
- CaseReopenResult + CaseReopenNotificationResult

#### How the implementation works
1. The supervisor opens a frozen case.
2. ReopenController validates authority and the reopen reason.
3. Case.reopenToSupervisorReview() moves the case back to "Supervisor Review".
4. CaseRepositoryImpl saves the new state.
5. ChainOfCustodyLog records the reopen entry.
6. NotificationService informs the assigned officer and analyst.

---

### UC9 – Mark Evidence as Verified
**Actor:** Digital Forensic Analyst

#### Primary classes
- EvidenceController
- VerificationMarkController
- EvidenceService

#### Supporting classes
- AbstractCaseWorkflowController
- EvidenceRepository + EvidenceRepositoryImpl
- CaseRepository + CaseRepositoryImpl
- Evidence
- Case
- ChainOfCustodyLog + AuditRepository + AuditLog
- EvidenceDecisionResult + EvidenceSnapshot
- EvidenceStatus + CaseState

#### How the implementation works
1. UC3 must already have stored a verification snapshot.
2. VerificationMarkController checks that the original and recalculated hashes match.
3. EvidenceRepositoryImpl marks the Evidence as "Verified".
4. ChainOfCustodyLog records the verification decision.

#### Document vs current code
- The PDF says UC9 should also move the case from "Forensic Review" to "Supervisor Review".
- The current code does not transition the case in UC9. The case stays in the forensic-review path until UC11 submits it for supervisor review.

---

### UC10 – Generate Summary Report
**Actor:** Supervisory Authority

#### Primary classes
- ReportController
- ReportService

#### Supporting classes
- CaseRepository + CaseRepositoryImpl
- SummaryReportRequest
- SummaryReportResult + SummaryReportCaseRecord + SummaryReportCaseFilter
- SeverityLevel + PriorityState + CaseState + EvidenceStatus
- AuditLogService + AuditRepository + AuditLog

#### How the implementation works
1. The supervisor selects filters in the report module.
2. ReportController sends the request to ReportService.
3. ReportService queries CaseRepositoryImpl for the requested case data.
4. The service builds SummaryReportResult and related record objects.
5. AuditLogService records the report-generation event.
6. The module returns the summary and can export it to CSV or PDF.

---

### UC11 – Submit Case for Supervisor Review
**Actor:** Investigating Officer

#### Primary classes
- SubmitReviewController
- SubmissionController
- CaseSubmissionService

#### Supporting classes
- AbstractCaseWorkflowController
- CaseRepository + CaseRepositoryImpl
- EvidenceRepository + EvidenceRepositoryImpl
- Case
- ChainOfCustodyLog + AuditRepository + AuditLog
- NotificationService + NotificationRepository + NotificationRecord
- CaseSubmissionResult
- CaseState + EvidenceStatus

#### How the implementation works
1. The investigating officer opens a case that has completed forensic handling.
2. SubmissionController verifies officer assignment and evidence status.
3. Case.moveToState(...) transitions the case to "Supervisor Review".
4. CaseRepositoryImpl stores the new state.
5. ChainOfCustodyLog records the submission.
6. NotificationService sends the review notifications to supervisors.

---

### UC12 – Reassign Investigating Officer
**Actor:** Supervisory Authority

#### Primary classes
- CaseController
- ReassignmentController
- CaseService

#### Supporting classes
- AbstractCaseWorkflowController
- CaseRepository + CaseRepositoryImpl
- UserRepository + UserRepositoryImpl
- Case + User
- ChainOfCustodyLog + AuditRepository + AuditLog
- NotificationService + NotificationRepository + NotificationRecord
- CaseReassignmentResult + NotificationDispatchResult
- CaseState

#### How the implementation works
1. The supervisor selects an active case and a new officer.
2. ReassignmentController validates authority and user roles.
3. CaseRepositoryImpl updates the assigned officer and case state.
4. ChainOfCustodyLog records the reassignment entry.
5. NotificationService sends notifications to the relevant officers.

---

### UC13 – View Chain-of-Custody Log
**Actor:** Supervisory Authority

#### Primary classes
- LogViewerController
- ChainOfCustodyViewerService

#### Supporting classes
- ChainOfCustodyLog
- AuditRepository + AuditRepositoryImpl
- CaseRepository + CaseRepositoryImpl
- ChainOfCustodySnapshot
- AuditLog
- Case

#### How the implementation works
1. The supervisor chooses a case for inspection.
2. ChainOfCustodyViewerService loads the case and retrieves all immutable entries through ChainOfCustodyLog.
3. ChainOfCustodySnapshot is returned to LogViewerController for chronological display in read-only form.

#### Document vs current code
- The PDF says a frozen and unacknowledged case should transition to "Under Supervisory Inspection".
- The current code does not create a formal new case state for that. Instead, ChainOfCustodySnapshot exposes an inspection-mode flag for the UI.

---

### UC14 – Review Escalated Case
**Actor:** Supervisory Authority

#### Primary classes
- ReviewEscalatedCaseController
- EscalatedCaseController
- EscalatedCaseReviewService

#### Supporting classes
- AbstractCaseWorkflowController
- CaseRepository + CaseRepositoryImpl
- EscalatedCaseReviewRepository + EscalatedCaseReviewRepositoryImpl
- EscalatedCaseReview
- EscalatedCaseSnapshot + EscalatedCaseReviewResult
- Case
- SLAManager
- ChainOfCustodyLog + AuditRepository + AuditLog
- PriorityState + SeverityLevel

#### How the implementation works
1. The supervisor opens a case that is marked escalated.
2. EscalatedCaseController confirms that SLA has been breached and that the case is in the correct priority state.
3. The controller records the review instructions.
4. CaseRepositoryImpl updates priority from "Escalated" to "Under Active Review".
5. EscalatedCaseReviewRepositoryImpl stores the review record.
6. ChainOfCustodyLog records the action.

#### Document vs current code
- The PDF implies the workflow should acknowledge the supervisory response to relevant parties.
- The current code records the review and updates priority, but no NotificationService call is wired into UC14.

---

### UC15 – Reject Case Closure
**Actor:** Supervisory Authority

#### Primary classes
- ManageCaseClosureController
- RejectionController
- CaseClosureService

#### Supporting classes
- AbstractCaseWorkflowController
- CaseRepository + CaseRepositoryImpl
- CaseClosureDecisionRepository + CaseClosureDecisionRepositoryImpl
- CaseClosureDecision + CaseClosureDecisionType
- CaseClosureSnapshot + CaseClosureRejectionResult
- Case
- ChainOfCustodyLog + AuditRepository + AuditLog

#### How the implementation works
1. The supervisor reviews a case in supervisor-review state.
2. RejectionController validates authority and the provided rejection reason.
3. Case.returnToForensicReview() moves the case back to "Forensic Review".
4. CaseRepositoryImpl stores the updated state.
5. CaseClosureDecisionRepositoryImpl stores the rejection record.
6. ChainOfCustodyLog records the action.

#### Document vs current code
- The PDF says the rejection reason is optional.
- The current code requires a non‑blank reason.
- The PDF says the case returns to the Digital Forensic Analyst for further investigation.
- The current code changes the state back to "Forensic Review" but does not send an analyst notification as part of UC15.

---

## 4. Master Class to Use‑Case Index

This section accounts for every Java source file in `src/main/java`.

### System / Module Shell
- `module-info.java` – Module declaration only; no direct UC mapping.
- `CaseManagementApplication` – Shared bootstrap for all UC1 to UC15.

### Controllers
- `AbstractCaseWorkflowController` – Shared base for UC1, UC2, UC3, UC4, UC5, UC6, UC7, UC8, UC9, UC11, UC12, UC14, UC15, and the current delete extension.
- `CaseClosureController` – UC5.
- `CaseController` – UC1, UC6, UC12, and the current‑code delete extension.
- `CaseDeletionController` – Current‑code delete extension (outside PDF scope).
- `CaseFreezeController` – UC4.
- `CaseRegistrationController` – UC1.
- `CaseStateTransitionController` – UC4, UC8.
- `DashboardController` – Shared navigation entry to UC1‑UC15.
- `EscalatedCaseController` – UC14.
- `EvidenceController` – UC2, UC3, UC7, UC9.
- `EvidenceUploadController` – UC2.
- `IntegrityVerificationController` – UC3.
- `LoginController` – Shared authentication precondition for UC1‑UC15.
- `LogViewerController` – UC13.
- `ManageCaseClosureController` – UC5, UC15.
- `NotificationController` – Supports notification outputs for UC5, UC8, UC11, UC12, and the delete extension.
- `ReassignmentController` – UC12.
- `RejectionController` – UC15.
- `ReopenController` – UC8.
- `ReportController` – UC10.
- `ReviewEscalatedCaseController` – UC14.
- `SeverityUpdateController` – UC6.
- `SignUpController` – Outside UC1‑UC15 scope.
- `SubmissionController` – UC11.
- `SubmitReviewController` – UC11.
- `TamperMarkController` – UC7 and supports UC4 via auto‑freeze.
- `VerificationMarkController` – UC9.

### Models / Enums
- `AuditLog` – Shared immutable record model for UC1‑UC15 and the delete extension.
- `Case` – Core domain model for UC1‑UC15 and the delete extension.
- `CaseClosureDecision` – UC5, UC15.
- `CaseClosureDecisionType` – UC5, UC15.
- `CaseState` – Shared state enum for UC1‑UC15 and the delete extension.
- `EscalatedCaseReview` – UC14.
- `Evidence` – UC2, UC3, UC7, UC9, and the delete extension.
- `EvidenceStatus` – UC2, UC3, UC7, UC9, and prerequisite in UC5, UC11.
- `NotificationRecord` – Supports UC5, UC8, UC11, UC12, and the delete extension.
- `PriorityState` – UC1, UC6, UC10, UC14.
- `SeverityLevel` – UC1, UC6, UC10, UC14.
- `SummaryReportCaseFilter` – UC10.
- `SummaryReportCaseRecord` – UC10.
- `SummaryReportRequest` – UC10.
- `User` – Shared actor model for UC1‑UC15 and the delete extension.
- `UserRole` – Shared role/enum for UC1‑UC15.

### Repositories
- `AuditRepository` / `AuditRepositoryImpl` – Shared persistence for logs in UC1‑UC15 and the delete extension.
- `CaseClosureDecisionRepository` / `Impl` – UC5, UC15, and the delete extension.
- `CaseRepository` / `Impl` – Shared case persistence across UC1‑UC15 and the delete extension.
- `EscalatedCaseReviewRepository` / `Impl` – UC14 and the delete extension.
- `EvidenceRepository` / `Impl` – UC2, UC3, UC7, UC9, and the delete extension.
- `NotificationRepository` / `Impl` – UC5, UC8, UC11, UC12, and the delete extension.
- `UserRepository` / `Impl` – Shared actor lookup/authentication for UC1‑UC15 and the delete extension.

### Services, Snapshots, and Result Objects
- `AuditLogService` – UC1, UC5, UC10, and shared audit helper.
- `AuthService` – Shared authentication precondition for UC1‑UC15.
- `CaseClosureApprovalResult` – UC5.
- `CaseClosureRejectionResult` – UC15.
- `CaseClosureService` – UC5, UC15.
- `CaseDeletionResult` – Current‑code delete extension.
- `CaseClosureSnapshot` – UC5, UC15.
- `CaseFreezeResult` – UC4.
- `CaseReassignmentResult` – UC12.
- `CaseReopenNotificationResult` – UC8.
- `CaseReopenResult` – UC8.
- `CaseService` – UC1, UC6, UC12, and the delete extension.
- `CaseSeverityUpdateResult` – UC6.
- `CaseStateTransitionService` – UC4, UC8.
- `CaseSubmissionResult` – UC11.
- `CaseSubmissionService` – UC11.
- `ChainOfCustodyLog` – UC2, UC3, UC4, UC6, UC7, UC8, UC9, UC11, UC12, UC13, UC14, UC15.
- `ChainOfCustodySnapshot` – UC13.
- `ChainOfCustodyViewerService` – UC13.
- `EscalatedCaseReviewResult` – UC14.
- `EscalatedCaseReviewService` – UC14.
- `EscalatedCaseSnapshot` – UC14.
- `EvidenceDecisionResult` – UC7, UC9.
- `EvidenceService` – UC2, UC3, UC7, UC9.
- `EvidenceSnapshot` – UC2, UC3, UC7, UC9.
- `EvidenceUploadResult` – UC2.
- `HashService` – UC2, UC3, UC7, UC9.
- `IntegrityVerificationResult` – UC3.
- `LogService` – Thin wrapper; no clear direct UC ownership.
- `NotificationDispatchResult` – UC12.
- `NotificationService` – UC5, UC8, UC11, UC12, and the delete extension.
- `ReportService` – UC10.
- `SecureFileStorage` – UC2 and the delete extension.
- `SignUpException` / `SignUpService` – Outside UC1‑UC15 scope.
- `SLAManager` – UC1, UC6, UC14.
- `SummaryReportResult` – UC10.

### Utilities
- `AppNavigator` – Shared navigation support for UC1‑UC15.
- `DBConnection` – Shared database infrastructure for UC1‑UC15.
- `IdGenerator` – UC1.
- `PasswordUtil` – Shared authentication utility; supports login preconditions.

---

## 5. High-Level Takeaways

1. **Architecture** – The codebase is strongly package‑organized by layer: controllers → services → repositories → models/utilities.

2. **Strongest one‑to‑one use‑case implementations** – UC1, UC2, UC4, UC5, UC6, UC8, UC10, UC11, UC12, UC13, UC14, UC15.

3. **Evidence integrity split** – UC3 performs hash comparison, while UC7 and UC9 finalize the result.

4. **Major document‑to‑code differences**  
   - UC3 outcome finalization is split into UC7 and UC9.  
   - UC9 does not directly move the case to Supervisor Review.  
   - UC13 uses an inspection flag instead of a formal new state.  
   - UC14 does not currently notify relevant parties.  
   - UC15 requires a reason even though the PDF marks it optional.

5. **Current extension** – A supervisor‑only hard‑delete extension exists in the Manage Case module (`CaseController`, `CaseService`, `CaseDeletionController`, `CaseDeletionResult`), outside the PDF’s UC1‑UC15 set.

6. **Classes outside PDF scope** – Mainly `SignUpController`, `SignUpService`, `SignUpException`, `CaseDeletionController`, and `CaseDeletionResult`. Shared repositories, models, and services also participate in the delete extension but remain cross‑use‑case infrastructure.
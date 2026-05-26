# Functional Requirements Document

## 1. Overview
`quota-journal` is an early scaffold providing local-first request journaling for quota-aware AI workloads. It treats LLM prompts as resumable jobs, ensuring that long-running tasks or free-tier workloads can survive disconnects and API limits.

## 2. Goals
* Persist request intent locally before execution.
* Track lifecycle states to allow interrupted workloads to resume.
* Provide a neutral snapshot of quota usage locally.

## 3. Non-Goals
* It is not a generic API gateway or an intelligent LLM load balancer.
* It does not contain proprietary model routing algorithms.
* It is not a managed infrastructure component for production deployments.

## 4. Target Users
* **Indie Hackers & Students:** Utilizing free-tier APIs or local CPU inference who need to protect long-running batch jobs.
* **Open-Source Contributors:** Looking for resume-friendly job tracking for local data processing.

## 5. Core User Stories
* As a developer, I want to journal requests before execution so that interrupted workflows can be inspected later.
* As a developer, I want request states such as pending, running, completed, and failed.
* As a maintainer, I want quota snapshots stored locally so that execution history is auditable.
* As a contributor, I want provider-neutral interfaces so that integrations can be added later.

## 6. Functional Requirements
* **FR-QJ-001:** The system must accept a structured request record representing an intended API call.
* **FR-QJ-002:** The system must assign and update a lifecycle state for each request (e.g., pending, completed).
* **FR-QJ-003:** The system must persist request records locally (e.g., SQLite or JSONL).
* **FR-QJ-004:** The system must support reading the latest request state to facilitate job resumption.
* **FR-QJ-005:** The system must represent quota snapshots in a standardized, provider-neutral format.
* **FR-QJ-006:** The CLI must support appending new requests and listing the status of local records.
* **FR-QJ-007:** The package must not require network access for its core journaling functionality.

## 7. Inputs and Outputs
| Component | Input | Output |
| :--- | :--- | :--- |
| **Journal Appender** | `RequestRecord` | `JournalEntry` (Saved) |
| **State Updater** | Job ID, `RequestState` | Updated `JournalEntry` |
| **Status CLI** | None | Summary View of Journal |

## 8. CLI Requirements
The CLI must allow users to view the current journal state (`quota-journal status`) and inject test records (`quota-journal append --data <file>`).

## 9. Configuration Requirements
Local configuration determines the storage backend (SQLite path vs JSONL) and simple logging verbosity. 

## 10. Error Handling Requirements
* Database lock errors must be caught and gracefully retried.
* Corrupted journal entries should be isolated and skipped to prevent halting the entire queue.

## 11. Observability Requirements
The state of the journal *is* the observability layer. Users should be able to query the local store to see exactly where a batch process failed.

## 12. Security and Privacy Requirements
* **Offline Defaults:** Journaling happens exclusively on the local machine.
* **Data Sanitization:** The framework does not obfuscate prompt text; users must secure their local journal files appropriately.

## 13. Accessibility and Developer Experience
Simple Python API hooks allow developers to wrap their existing LLM calls in a `with journal_context(request):` paradigm for low-friction integration.

## 14. Milestones
* **M1:** Define data models and state transition logic.
* **M2:** Implement local storage adapters (SQLite).
* **M3:** Build the CLI and resume capabilities.

## 15. Open Questions
* Should the journal automatically prune completed jobs after a certain timeframe, or keep them indefinitely for audit purposes?

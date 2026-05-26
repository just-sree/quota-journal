# Technical Design Document

## 1. Overview
This technical design covers `quota-journal`, focusing on local state management and durable logging for AI requests to enable reliable, resumable workloads.

## 2. Design Principles
* **Durability First:** Write intent to disk before attempting network/local execution.
* **Idempotent Resumption:** System must be able to restart without duplicating completed work.
* **Provider Agnosticism:** The journal records the *intent* of a job, not the proprietary API spec of the provider.

## 3. System Boundaries
`quota-journal` acts as a state wrapper around the user's execution logic. It tracks "what is about to happen" and "what just happened," but does not enforce rate limits or execute the API calls itself.

## 4. High-Level Architecture
```text
CLI / Python API
   ↓
RequestRecord parser
   ↓
JournalStore
   ↓
State transition helper
   ↓
QuotaSnapshot model
   ↓
Local JSONL or SQLite storage
```

## 5. Package Structure

```text
src/quota_journal/
  __init__.py
  cli.py
  models.py
  journal.py
  quota.py
  storage.py
tests/
  test_models.py
  test_journal.py
  test_cli.py
docs/
  FRD.md
  TDD.md
```

## 6. Core Data Models

* `RequestRecord`: Metadata defining the job (e.g., prompt hash, complexity heuristic).
* `RequestState`: Enum mapping to `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `DEFERRED`.
* `QuotaSnapshot`: A generic model capturing tokens used, requests made, and arbitrary quota tags.
* `JournalEntry`: The database row or JSON object combining the `RequestRecord`, `RequestState`, and timestamps.

## 7. Main Components

* **JournalStore:** The abstraction layer over SQLite or File I/O.
* **Transition Manager:** Ensures state changes follow logical paths (e.g., a `COMPLETED` job cannot move back to `RUNNING`).
* **CLI Controller:** Exposes journal queries to the terminal.

## 8. Control Flow

1. Developer defines a `RequestRecord` and submits it to the `JournalStore`.
2. Store writes the record as `PENDING`.
3. Developer script queries the store for `PENDING` jobs.
4. Script attempts execution, marking state as `RUNNING`.
5. Upon success/failure, script logs a `QuotaSnapshot` and updates state to `COMPLETED` or `FAILED`.
6. If the script crashes, the next run queries for `PENDING` and interrupted `RUNNING` jobs to resume.

## 9. Storage Design

SQLite is the primary recommended backend, leveraging its ACID compliance to handle state transitions safely without complex setup.

## 10. Configuration Design

Managed via environment variables or a `.env` file pointing to the target local database path.

## 11. CLI Design

Built using `Typer` or `Click`.
Key commands:

* `status`: Shows counts grouped by `RequestState`.
* `resume`: Outputs the IDs of jobs that were interrupted.

## 12. Testing Strategy

* **Unit Tests:** `pytest` focusing heavily on the State transition helper to ensure invalid state changes raise exceptions.
* **Integration Tests:** Simulating script crashes and verifying the `JournalStore` accurately recovers `RUNNING` jobs.

## 13. Failure Modes

* **Database Locking:** Addressed via standard SQLite retry timeouts.
* **Orphaned Running Jobs:** If a script crashes silently, a `RUNNING` job might stay stuck. A timeout mechanism will eventually mark them as `FAILED` or `PENDING` based on age.

## 14. Security and Privacy Considerations

The local database will store verbatim prompts and potentially API outputs (if the user configures it). Securing this file via OS-level permissions is required.

## 15. Extensibility

The `QuotaSnapshot` model is designed to be easily subclassed by future developers who wish to build specific adapters for token-counting APIs.

## 16. Deferred Work

* Complex, multi-node concurrent state locking.
* Built-in HTTP proxying (kept out of scope to maintain simplicity).

## 17. Open Questions

* What is the optimal timeout heuristic for identifying an orphaned `RUNNING` job vs. a legitimately slow local CPU inference task?

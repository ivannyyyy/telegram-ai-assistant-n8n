# v1 → v2 Architecture Evolution

This document compares the two public architecture iterations of the same Telegram AI Assistant project.

The purpose is not to call v2 generically better. The comparison ties concrete v1 limitations to specific changes visible in the supplied v2 workflows.

## Summary

v1:
- Airtable-driven scheduled workflows
- generic conversation/event state
- separate chat-memory layer
- blocking human escalation
- direct lifecycle workers

v2:
- Airtable control plane
- PostgreSQL runtime entities
- durable event queue
- explicit messages, facts and owner requests
- asynchronous human escalation
- state reconciliation
- durable completion notifications

## 1. Runtime orchestration

### v1

Multiple workflows directly perform operational actions: start task, process incoming messages, wait for administrator information, send queued messages, follow up and stop/timeout.

### v2

Runtime work is represented in agent_v3.events. The Event Runner claims due rows, marks them processing, records lock/attempt state and dispatches the event to the appropriate semantic path.

Observed event types include start_dialog, process_inbound, owner_answer and followup.

**Result:** the redesign introduces a durable unit-of-work model. The limitation is only partially addressed because v2 still uses minute-based polling to discover due events.

## 2. Serial workers and concurrency

v1 key workers often process a very small number of records per run.

The v2 Event Runner claim query uses FOR UPDATE SKIP LOCKED with LIMIT 10. The Completion Notifier uses a similar claim/lock pattern with its own batch limit.

**Result:** bounded batches and safer claim semantics are visible in the supplied workflows. This does not by itself prove horizontal scalability of the whole system.

## 3. Airtable coupling

Airtable remains the administrator-facing control plane in v2, so the dependency is not removed.

However, runtime responsibilities are represented explicitly in PostgreSQL through dialogs, events, messages, facts, owner_requests and file_cache.

**Result:** partially addressed. Airtable still defines important configuration and linked records, but it is less responsible for runtime state.

## 4. Human escalation

### v1

The flow is approximately: wait_admin → persist unresolved question → administrator updates Airtable knowledge → task new_info → scheduled admin-answer worker → resume conversation.

### v2

Agent Core can select ask_owner. A safe user-facing message is still sent, while a separate owner_requests record is created. The administrator answer later becomes an owner_answer event and returns to Agent Core.

The request also carries answer_scope = task or dialog.

**Result:** human-in-the-loop becomes explicit, asynchronous and scope-aware rather than a single blocking pause.

## 5. Knowledge and memory model

In v1, operational history lives in public.user_task while the LangChain agent also uses PostgreSQL chat memory.

In the supplied v2 Agent Core, runtime context is constructed from agent_v3.messages, agent_v3.facts, agent_v3.owner_requests and dialog state. No separate PostgreSQL chat-memory node is visible in the supplied v2 Agent Core workflow.

**Result:** conversation context becomes more domain-oriented: messages are history, facts are confirmed knowledge and owner requests are unresolved human dependencies.

## 6. Model decision boundary

v1 validates send_message, wait_admin and finish after the LLM response.

v2 keeps deterministic JavaScript validation but expands the semantic actions to send_message, ask_owner, no_action and complete_dialog. It also checks which actions are legal for each event type and validates required fields.

**Result:** the semantic state machine is stronger, but model JSON is still prompt-defined rather than schema-bound by a dedicated structured-output component.

## 7. Working-time configuration

v1 contains several Moscow-time and weekday assumptions.

v2 reads timezone, work_days, work_start, work_end, delay_min_sec and delay_max_sec from task configuration. The scheduling code also supports overnight windows. Moscow/weekday values remain as fallbacks.

**Result:** substantially addressed through task-level configuration.

## 8. Paused and Done state drift

v2 adds a PostgreSQL State Reconciler that copies Airtable task state into runtime metadata, clears scheduling when Paused/Done, cancels pending/processing events where appropriate and audits duplicate Telegram routes.

Telegram ingress still refuses ambiguous active routing.

**Result:** v2 combines repair with fail-closed routing.

## 9. Task lifecycle vs dialog lifecycle

v2 explicitly treats a task as a long-lived container. complete_dialog closes only the current dialog; the task can retain other or future dialogs. The Agent Core does not automatically mark the whole task Done.

**Architecture change:** task and dialog lifecycle are separate concepts.

## 10. Completion reliability

v2 stores durable notification state on completed dialogs using pending, locked and sent metadata. A separate Completion Notifier claims eligible dialogs, sends the administrator notification and then marks it sent.

**Architecture change:** conversation completion is separated from administrator-notification delivery.

## 11. File handling

v1 stores received files through Cloudinary and later converts URLs into Airtable attachments.

v2 introduces agent_v3.file_cache. Supported small attachments are cached with an expiring token, uploaded directly to Airtable by a separate File Upload workflow and then removed from the cache. The supplied direct-upload path enforces a 5 MB limit.

Telegram ingress contains a separate path for larger or unknown-size attachments and can forward the original file plus context to the administrator.

**Architecture change:** explicit transient file cache and separate upload worker.

## 12. Reporting utilities

v1 uses separate workflows for log export, interim result and characteristics.

v2 Dialog Tools polls Airtable checkboxes, expands them into log, interim or characteristic actions and uses one shared transcript builder over agent_v3.messages.

**Result:** duplicated transcript-building logic is consolidated.

## 13. RAG

v2 adds facts and a stronger file pipeline, but the supplied workflows still do not show document chunking, embeddings, vector search or semantic document retrieval.

**Result:** v2 still should not be described as RAG.

## 14. What v2 actually demonstrates

The strongest portfolio progression is:

v1: workflow orchestration around an LLM

v2: explicit runtime architecture around an AI agent

v2 visibly adds:
- durable event records
- claim/lock semantics
- explicit attempt state
- persisted message history
- confirmed facts
- owner-request lifecycle
- task/dialog fact scope
- strict route guards
- state reconciliation
- asynchronous human escalation
- durable completion notification
- consolidated reporting utilities
- transient PostgreSQL file cache

At the same time, the documentation should remain precise: workers still poll, Airtable remains a major dependency, LLM JSON is still prompt-defined and there is still no RAG pipeline.

## Public repository structure

workflows/v1/ contains the completed first architecture.

The planned public v2 files are:
- workflows/v2/01_task_watcher.json
- workflows/v2/02_telegram_ingress.json
- workflows/v2/03_event_runner_agent_core.json
- workflows/v2/04_completion_notifier.json
- workflows/v2/05_state_reconciler.json
- workflows/v2/06_file_upload.json
- workflows/v2/07_dialog_tools.json

Documentation:
- docs/v2-architecture.md
- docs/v2-data-model.md
- docs/v1-to-v2.md

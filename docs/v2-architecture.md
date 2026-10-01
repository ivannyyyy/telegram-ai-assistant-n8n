# v2 Architecture

## Positioning

This repository calls the redesigned system **v2** because it is the second public architecture of the same Telegram AI Assistant project.

The source workflow exports use internal names such as `V3` and PostgreSQL tables under the `agent_v3` schema. Those internal names are preserved where they are part of the implementation, but the public repository structure uses `workflows/v2/` so the evolution from the published v1 remains clear.

v2 is best described as a **stateful AI-agent runtime with a durable PostgreSQL event queue, polling workers, human-in-the-loop escalation and explicit reconciliation between Airtable and runtime state**.

It is not fully event-driven: several workers still poll every minute. The important redesign is that runtime work is represented as durable PostgreSQL events rather than being spread across independent scheduled flows.

## High-level architecture

```text
                         ┌─────────────────────────────┐
                         │      Airtable Control       │
                         │ tasks + dialogs + admin UI  │
                         └──────────────┬──────────────┘
                                        │
                           Task Watcher │ State Reconciler
                                        │
                                        ▼
┌───────────────┐           ┌───────────────────────────────┐
│   Telegram    │──────────►│          PostgreSQL           │
│   TelePilot   │ Telegram  │ dialogs / events / messages   │
└───────┬───────┘  Ingress  │ facts / owner_requests        │
        │                   │ file_cache                    │
        │                   └──────────────┬────────────────┘
        │                                  │
        │                           claim due events
        │                                  ▼
        │                   ┌───────────────────────────────┐
        └──────────────────►│ Event Runner + Agent Core     │
                            │ deterministic action routing  │
                            └───────┬───────────┬───────────┘
                                    │           │
                         send reply │           │ ask owner
                                    │           ▼
                                    │    owner_requests
                                    │           │
                                    │      owner answer
                                    │           │
                                    └───────────┴──────► new event

       Completion Notifier ◄── done dialogs + completion metadata
       Dialog Tools        ◄── Airtable checkbox requests
       File Upload         ◄── PostgreSQL file_cache + Airtable attachments
```

## 1. Airtable remains the control plane

Airtable still stores task configuration and administrator-facing records, but runtime state is moved much more explicitly into PostgreSQL.

The Task Watcher reads active tasks, expands linked dialogs and copies operational configuration into runtime scheduling. The source workflow uses task-level fields such as:

- `timezone`;
- `work_days`;
- `work_start`;
- `work_end`;
- `delay_min_sec`;
- `delay_max_sec`.

For a new dialog, the watcher upserts a row in `agent_v3.dialogs` and creates a `start_dialog` event in `agent_v3.events`.

The watcher also handles task state transitions. Active tasks can resume deferred events, while Paused and Done states are synchronized into runtime state.

## 2. PostgreSQL is the runtime backbone

The supplied workflows reference these runtime tables:

- `agent_v3.dialogs`
- `agent_v3.events`
- `agent_v3.messages`
- `agent_v3.facts`
- `agent_v3.owner_requests`
- `agent_v3.file_cache`

Instead of keeping operational state in one generic event/history table plus a separate chat-memory layer, v2 separates runtime responsibilities into explicit entities.

The exact DDL is not included in the supplied exports, so this document describes only behavior visible in the workflows.

## 3. Durable event queue

The Event Runner polls for due events every minute.

Due rows are claimed from `agent_v3.events` when:

```text
status = pending
run_at <= now
```

The claim query uses:

```sql
FOR UPDATE SKIP LOCKED
LIMIT 10
```

and updates claimed rows to `processing`, sets `locked_at`, and increments `attempts`.

This creates a stronger execution boundary than v1 because a unit of work is now represented by a persisted event with explicit status, schedule, lock and retry-related metadata.

Observed event types include:

- `start_dialog`
- `process_inbound`
- `owner_answer`
- `followup`

Unsupported event types are explicitly closed as errors.

## 4. Telegram ingress is separated from the agent runtime

The Telegram In workflow handles transport-facing concerns before the Agent Core executes.

It:

- ignores outgoing Telegram messages;
- normalizes text, photos, documents and other message types;
- extracts Telegram user/chat/message identifiers;
- resolves one routable active dialog from PostgreSQL;
- fails if more than one active runtime dialog matches the same Telegram route;
- persists incoming messages;
- schedules or reschedules a `process_inbound` event;
- routes administrator replies to open `owner_requests`;
- handles attachment transport and file-cache flow.

This means receiving a Telegram update and deciding the semantic next action are separate responsibilities.

## 5. Debounced inbound processing

Incoming messages are not required to trigger the Agent Core immediately.

The Telegram ingress calculates the next allowed runtime time using task configuration and a random human-like delay. If a pending `process_inbound` event already exists, the event can be moved to the new `run_at` rather than creating uncontrolled duplicate work.

The same scheduling logic supports normal and overnight working windows.

## 6. Agent Core as a semantic dispatcher

Before the LLM runs, the Event Runner builds context from runtime data.

For inbound processing this includes:

- task goal and required result;
- assistant role and identity;
- information base and rules;
- communication style;
- recent messages;
- confirmed facts;
- open owner requests;
- current waiting state.

The Agent Core chooses a semantic action, but deterministic JavaScript validates the result before any workflow branch executes.

The parser accepts only:

- `send_message`
- `ask_owner`
- `no_action`
- `complete_dialog`

Not every action is valid for every event type. For example:

- `ask_owner` is allowed for `process_inbound`;
- `no_action` is restricted to `followup`;
- `complete_dialog` is restricted to `process_inbound`;
- `owner_answer` context is constrained to `send_message`.

Invalid actions or missing required fields fail closed.

## 7. Human-in-the-loop becomes asynchronous

v2 introduces `agent_v3.owner_requests` as an explicit runtime entity.

When the model selects `ask_owner`:

1. the assistant must still create a safe user-facing message;
2. an owner/admin question is stored separately;
3. the user conversation does not have to freeze;
4. an administrator can answer later;
5. the answer creates an `owner_answer` event;
6. the Agent Core sends the clarified fact back into the current conversation context.

The action also classifies the answer scope:

- `task` — safe reusable knowledge for all dialogs in the task;
- `dialog` — information specific to one person/situation.

This makes human escalation a parallel asynchronous path rather than a single blocking pause.

## 8. Confirmed facts are explicit runtime context

The Event Runner loads active facts from `agent_v3.facts` and gives them to the Agent Core as confirmed information.

The Agent Core prompt treats those facts differently depending on scope:

- task-scoped facts may be reused across dialogs in the same task;
- dialog-scoped facts stay local to the current conversation.

The supplied workflows therefore show an explicit confirmed-fact layer, but not semantic retrieval over document embeddings.

## 9. Task lifecycle and dialog lifecycle are separated

A task is treated as a long-lived container that may contain multiple dialogs.

`complete_dialog` closes only the current dialog. Completion:

- sets the dialog to `done`;
- stores `summary` and `final_result`;
- records a completion reason;
- cancels remaining pending events for that dialog;
- cancels open owner requests for that dialog;
- updates the task-level aggregated result.

The Agent Core does not automatically close the whole Airtable task.

## 10. State reconciliation protects routing

The PostgreSQL State Reconciler continuously synchronizes Airtable task state into runtime metadata.

Observed task states:

- `Active`
- `Paused`
- `Done`

For Paused/Done tasks it can clear `next_action_at` and cancel pending/processing events. For Done tasks, active runtime dialogs are moved to `done`.

The reconciler also audits for remaining duplicate Telegram routes after synchronization. This is deliberately separate from the strict routing guard in Telegram ingress: reconciliation repairs state drift, while ingress still refuses ambiguous routing.

## 11. Durable completion notifications

Dialog completion and administrator notification are separated.

When a dialog completes, completion metadata is written to the dialog, including a pending-notification flag.

The Completion Notifier later claims eligible completed dialogs with its own lock fields and `FOR UPDATE SKIP LOCKED`, sends the result to the configured administrator, and marks the notification as sent.

This means a completed dialog does not depend on one synchronous Telegram-admin notification succeeding in the same execution path.

## 12. File handling

v2 introduces `agent_v3.file_cache` for transient file bytes.

The Telegram ingress can cache downloaded file data with an expiring token. The File Upload workflow resolves those tokens, reconstructs binary data and uploads supported small attachments directly to Airtable.

The supplied File Upload workflow enforces a direct-upload limit of 5 MB. The Telegram ingress contains separate handling for larger/unknown-size attachments, including forwarding the original file and context to the configured administrator.

The file cache is deleted after successful direct upload.

This is still a file transport/storage pipeline, not RAG.

## 13. Dialog utilities are consolidated

v1 used separate utility workflows for log export, interim result and conversation characteristics.

v2 uses one Dialog Tools workflow. Airtable checkboxes expand into three actions:

- `log`
- `interim`
- `characteristic`

A single transcript builder loads ordered messages from `agent_v3.messages` and then routes the requested operation.

## 14. Polling remains

v2 should not be described as a pure event-driven platform.

Task Watcher, Event Runner, Completion Notifier, State Reconciler and Dialog Tools still use scheduled polling.

The architectural improvement is narrower and more concrete:

```text
v1:
scheduled workflow directly performs business action

v2:
scheduled worker claims durable PostgreSQL work
→ runtime state is persisted
→ deterministic handler processes it
→ completion is recorded explicitly
```

## 15. No RAG claim

The supplied v2 workflows show:

- explicit messages;
- facts;
- owner requests;
- task information;
- file storage/cache.

They do not show:

- document chunking;
- embeddings;
- vector search;
- semantic document retrieval.

Therefore v2 should still not be presented as a RAG implementation.

## Public workflow mapping

The public repository will use the following v2 names:

| Public file | Source workflow responsibility |
|---|---|
| `01_task_watcher.json` | task state, scheduling and dialog/runtime creation |
| `02_telegram_ingress.json` | Telegram input, routing, owner replies and attachment intake |
| `03_event_runner_agent_core.json` | durable event queue and semantic Agent Core |
| `04_completion_notifier.json` | durable administrator completion notifications |
| `05_state_reconciler.json` | Airtable → PostgreSQL state reconciliation and route audit |
| `06_file_upload.json` | file-cache → Airtable attachment upload |
| `07_dialog_tools.json` | log/interim/characteristic utility actions |

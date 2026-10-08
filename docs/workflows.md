# Workflow Guide

This repository contains two published workflow sets:

- `workflows/v1/` — first production-oriented architecture;
- `workflows/v2/` — redesigned runtime architecture with durable PostgreSQL events and explicit state entities.

# v1 workflows

| File | Responsibility |
|---|---|
| `01_start_task.json` | Polls Airtable for a new assistant conversation, loads task context, creates the opening Telegram message, stores initial conversation state and marks the record `In progress`. |
| `02_incoming_message.json` | Main v1 runtime: receives Telegram updates, resolves active conversation state, handles text/files, writes message history, runs the dialogue LLM, validates structured output, routes `send_message` / `wait_admin` / `finish`, queues out-of-hours responses and completes finished tasks. |
| `03_admin_answer.json` | Detects task records marked with new administrator information, finds unresolved dialogue rows, generates a narrowly scoped answer and sends or queues it. |
| `04_wait_message.json` | Polls PostgreSQL rows with `wait = true` and delivers queued messages when the task-level communication window is open. |
| `05_follow_up_on_pause.json` | Checks an active conversation for an unresolved promise by the user and may send a short follow-up after the v1 working-time threshold. |
| `06_stop_and_timeout.json` | Handles manual stop and `days_limit` timeout, updates Airtable/PostgreSQL and clears chat memory. |
| `07_upload_documents_to_airtable.json` | Webhook helper that converts URLs from `docs_url` into Airtable attachment objects. |
| `08_dialog_characteristics.json` | Builds a chronological dialogue transcript and passes it to an LLM using the task-configured `Promt_comunication`; stores the returned analysis in Airtable. |
| `09_dialog_log_export.json` | Builds a chronological dialogue transcript and stores it in the Airtable `Log` field. |
| `10_interim_result.json` | Builds the dialogue transcript and asks the LLM for a structured intermediate progress report against the task goal. |
| `11_error_handler.json` | Error-trigger workflow that sends administrator notifications. |

## v1 main statuses

### Airtable conversation

Observed values:

- `new`
- `In progress`
- `stop`
- `Done`

### Airtable task

The v1 admin-answer path uses:

- `new_info`

and later marks the processed record:

- `Done`

### LLM routing

The v1 incoming parser accepts:

- `send_message`
- `wait_admin`
- `finish`

### PostgreSQL state

`public.user_task` uses fields such as:

- `wait` — delayed-delivery flag;
- `airtable_status` — active/done state;
- `status_bot_message` — sent/pending administrator-question state.

# v2 workflows

| File | Responsibility |
|---|---|
| `01_task_watcher.json` | Polls Airtable task state, schedules new dialogs, upserts `agent_v3.dialogs`, creates `start_dialog` events and handles task-state transitions such as Active/Paused/Done. |
| `02_telegram_ingress.json` | Normalizes incoming TelePilot updates, resolves one routable runtime dialog, persists inbound messages, debounces `process_inbound` work, routes administrator replies, saves confirmed owner answers as facts and handles attachment intake. |
| `03_event_runner_agent_core.json` | Central runtime worker. Claims due events, builds semantic context, runs Agent Core, validates its decision and performs `send_message`, `ask_owner`, `no_action` or `complete_dialog` transitions. |
| `04_completion_notifier.json` | Claims completed dialogs that still need administrator notification, resolves the admin route, sends the notification and records durable sent/lock metadata. |
| `05_state_reconciler.json` | Reconciles Airtable task state into PostgreSQL runtime state, clears/cancels work for Paused/Done tasks and audits ambiguous Telegram routing after reconciliation. |
| `06_file_upload.json` | Resolves temporary file-cache tokens, reconstructs file bytes, uploads supported attachments to Airtable and removes successfully uploaded cache rows. |
| `07_dialog_tools.json` | Consolidates transcript log export, interim-result analysis and dialog/person characteristics into one polling utility workflow. |

## v2 execution flow

A simplified runtime path is:

```text
Airtable task/dialog
        │
        ▼
Task Watcher
        │
        ├── upsert agent_v3.dialogs
        └── create start_dialog event
                    │
                    ▼
             Event Runner
                    │
              Agent Core
                    │
       ┌────────────┼──────────────┐
       │            │              │
send_message    ask_owner     complete_dialog
       │            │              │
       │            ▼              ▼
       │      owner_requests   completion metadata
       │            │              │
       │       owner reply         ▼
       │            │       Completion Notifier
       │            ▼
       │       owner_answer event
       │            │
       └────────────┴──────► Event Runner
```

Incoming Telegram messages enter separately through `02_telegram_ingress.json`, which persists the message and schedules/reschedules `process_inbound`.

## v2 event queue

The central Event Runner works against `agent_v3.events`.

Observed event types:

- `start_dialog`
- `process_inbound`
- `owner_answer`
- `followup`

Observed event statuses:

- `pending`
- `processing`
- `done`
- `error`
- `cancelled`

Due events are claimed with a PostgreSQL locking pattern using `FOR UPDATE SKIP LOCKED` and a bounded batch.

The workflows still use scheduled polling; v2 is therefore not described as a pure event-driven platform.

## v2 dialog state

Observed PostgreSQL dialog states:

- `new`
- `scheduled`
- `active`
- `waiting_user`
- `waiting_owner`
- `done`
- `error`

The corresponding Airtable dialog labels are:

- `New`
- `Scheduled`
- `Active`
- `Waiting user`
- `Waiting owner`
- `Done`
- `Error`

Task state is separately observed as:

- `Active`
- `Paused`
- `Done`

The State Reconciler mirrors task state into runtime metadata and applies runtime consequences for Paused/Done tasks.

## v2 Agent Core actions

The deterministic parser accepts:

- `send_message`
- `ask_owner`
- `no_action`
- `complete_dialog`

Action availability depends on event type.

Important distinctions:

- `ask_owner` is asynchronous and can create an owner request without freezing the dialog;
- `no_action` is used by follow-up handling when no reminder should be sent;
- `complete_dialog` closes the current dialog, not the entire task.

The public Agent Core prompt is a sanitized portfolio version. The runtime action/state contract is preserved.

## v2 human-in-the-loop flow

```text
process_inbound
      │
      ▼
Agent Core: ask_owner
      │
      ├── send safe user-facing message
      │
      └── create agent_v3.owner_requests
                  │
                  ▼
          administrator reply
                  │
                  ├── mark request answered
                  ├── create confirmed fact
                  └── create owner_answer event
                               │
                               ▼
                        Event Runner
                               │
                               ▼
                     clarified outbound message
```

Confirmed facts use `scope = task` or `scope = dialog`.

## v2 message history

`agent_v3.messages` is the runtime transcript source.

The workflows store inbound and outbound messages with fields such as:

- `dialog_id`
- `telegram_message_id`
- `direction`
- `sender_type`
- `message_type`
- `text`
- Telegram/database timestamps
- metadata

Dialog Tools builds transcript output from this table instead of reconstructing the same history independently in three separate workflows.

## v2 file handling

Telegram ingress handles attachment classification and transport.

For the temporary PostgreSQL path:

```text
Telegram attachment
        ↓
agent_v3.file_cache
        ↓
temporary token / URL
        ↓
06_file_upload.json
        ↓
Airtable attachment
        ↓
cache deletion
```

The supplied direct Airtable upload branch enforces a 5 MB limit.

Large or unknown-size attachments use a separate administrator-forwarding branch in Telegram ingress.

This is a storage/transport pipeline, not RAG.

## v2 completion

`complete_dialog` stores:

- `summary`
- `final_result`
- `completion_reason`
- completion timestamp
- pending completion-notification metadata

It also cancels remaining pending work and open owner requests for that dialog.

`04_completion_notifier.json` later claims pending completion notices and marks them sent after administrator delivery.

## What changed from v1

The main architectural shift is:

```text
v1:
scheduled workflows directly drive most business actions

v2:
scheduled workers claim durable PostgreSQL runtime work
→ explicit events/messages/facts/owner requests
→ deterministic handler transitions
→ explicit completion/reconciliation state
```

For the detailed comparison, see `v1-to-v2.md`.

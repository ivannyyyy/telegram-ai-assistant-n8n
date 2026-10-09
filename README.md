# Human-in-the-Loop Telegram AI Assistant

<p align="center">
  <img src="https://raw.githubusercontent.com/ivannyyyy/ivannyyyy/main/assets/telegram-ai-assistant-pixel.png" width="100%" alt="Telegram AI Assistant — stateful dialogue, durable events and human escalation architecture" />
</p>

A task-driven Telegram AI assistant built with n8n, Airtable and PostgreSQL.

The repository intentionally preserves **two architecture iterations** of the same system:

- **v1** — the original production-oriented workflow architecture;
- **v2** — a redesign around explicit PostgreSQL runtime state, a durable event queue, asynchronous human escalation, state reconciliation and separate reliability workers.

**Status: v2 complete.**

The public repository currently contains **18 sanitized n8n workflows**:

- 11 workflows in `workflows/v1/`;
- 7 workflows in `workflows/v2/`.

## Why two versions are preserved

v2 does not overwrite v1.

The purpose of the repository is to show the engineering evolution of the same automation:

```text
v1
direct workflow orchestration around an LLM
        │
        ▼
known runtime and maintainability limitations
        │
        ▼
architecture redesign
        │
        ▼
v2
explicit AI-agent runtime
+ durable PostgreSQL events
+ messages / facts / owner requests
+ asynchronous human-in-the-loop
+ state reconciliation
+ durable completion notifications
```

This makes the repository useful not only as a workflow collection, but as an example of how an AI automation architecture can evolve after operating requirements become more complex.

## v2 architecture

v2 is best described as a **stateful AI-agent runtime with durable PostgreSQL event records and polling workers**.

It is not presented as a pure event-driven architecture: several workers intentionally poll every minute. The architectural change is that runtime work is now persisted and claimed from PostgreSQL instead of being encoded only in independent scheduled workflows.

```text
                         ┌─────────────────────────────┐
                         │      Airtable Control       │
                         │ tasks / dialogs / admin UI  │
                         └──────────────┬──────────────┘
                                        │
                           Task Watcher │ State Reconciler
                                        │
                                        ▼
┌───────────────┐           ┌───────────────────────────────┐
│   Telegram    │──────────►│          PostgreSQL           │
│   TelePilot   │ ingress   │ dialogs / events / messages   │
└───────┬───────┘           │ facts / owner_requests        │
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
                         user reply │           │ ask_owner
                                    │           ▼
                                    │     owner_requests
                                    │           │
                                    │      owner answer
                                    │           ▼
                                    └──────► owner_answer event

          Completion Notifier ◄── completed dialogs
          Dialog Tools        ◄── admin utility requests
          File Upload         ◄── PostgreSQL file_cache
```

## What v2 demonstrates

- durable runtime events in PostgreSQL;
- bounded event claiming with `FOR UPDATE SKIP LOCKED`;
- explicit dialog lifecycle and task/dialog separation;
- persisted message history;
- confirmed facts with `task` or `dialog` scope;
- non-blocking human-in-the-loop escalation;
- owner answers returned asynchronously through `owner_answer` events;
- deterministic validation around LLM decisions;
- configurable work days, time windows, timezone and randomized delivery delay;
- debounced inbound processing;
- follow-up events;
- explicit Paused/Done state reconciliation;
- duplicate Telegram-route guards;
- durable completion notification state;
- transient PostgreSQL file cache;
- consolidated log/interim/characteristic tooling.

## Agent Core actions

The v2 runtime accepts four semantic actions after deterministic validation:

```text
send_message
ask_owner
no_action
complete_dialog
```

Their meaning is intentionally narrow:

- `send_message` — continue the current dialog;
- `ask_owner` — create an asynchronous human escalation while the dialog may continue;
- `no_action` — explicitly do nothing for a follow-up event;
- `complete_dialog` — finish the current dialog only, not the whole task.

The public Agent Core prompt is a **sanitized portfolio version**, while the runtime action/state contract is preserved.

## Human-in-the-loop redesign

### v1

```text
missing information
      ↓
wait_admin
      ↓
persist unresolved question
      ↓
administrator updates Airtable
      ↓
scheduled resume
```

### v2

```text
Agent Core: ask_owner
      │
      ├──► send safe user-facing message
      │
      └──► agent_v3.owner_requests
                    │
             administrator reply
                    │
                    ├──► confirmed fact
                    └──► owner_answer event
                                 │
                                 ▼
                           Event Runner
                                 │
                                 ▼
                        clarified user reply
```

Owner answers are classified as:

- `task` — reusable task-wide knowledge;
- `dialog` — person/situation-specific knowledge.

## Runtime data model

The supplied v2 workflows reference these PostgreSQL runtime entities:

```text
agent_v3.dialogs
agent_v3.events
agent_v3.messages
agent_v3.facts
agent_v3.owner_requests
agent_v3.file_cache
```

Airtable remains the administrator-facing control plane.

The main logical Airtable entities are:

```text
agent_tasks_v3
agent_dialogs_v3
agent_owner_requests_v3
```

The original PostgreSQL DDL/migrations are not part of the supplied source exports, so the repository documents the observed model without inventing unsupported schema details.

## v2 workflow set

| Workflow | Responsibility |
|---|---|
| `01_task_watcher.json` | Task-state polling, dialog creation, start scheduling and resume/cancel behavior |
| `02_telegram_ingress.json` | Telegram normalization, routing, inbound persistence, debounce, owner replies and attachment intake |
| `03_event_runner_agent_core.json` | Durable event claiming, semantic context, Agent Core and runtime state transitions |
| `04_completion_notifier.json` | Reliable administrator notification for completed dialogs |
| `05_state_reconciler.json` | Airtable → PostgreSQL state reconciliation and duplicate-route audit |
| `06_file_upload.json` | Temporary file-cache → Airtable attachment upload |
| `07_dialog_tools.json` | Transcript log, interim result and dialog/person analysis |

## Repository structure

```text
workflows/
├── v1/
│   ├── 01_start_task.json
│   ├── 02_incoming_message.json
│   ├── 03_admin_answer.json
│   ├── 04_wait_message.json
│   ├── 05_follow_up_on_pause.json
│   ├── 06_stop_and_timeout.json
│   ├── 07_upload_documents_to_airtable.json
│   ├── 08_dialog_characteristics.json
│   ├── 09_dialog_log_export.json
│   ├── 10_interim_result.json
│   └── 11_error_handler.json
│
└── v2/
    ├── 01_task_watcher.json
    ├── 02_telegram_ingress.json
    ├── 03_event_runner_agent_core.json
    ├── 04_completion_notifier.json
    ├── 05_state_reconciler.json
    ├── 06_file_upload.json
    └── 07_dialog_tools.json

docs/
├── architecture.md
├── data-model.md
├── known-limitations-v1.md
├── v1-to-v2.md
├── v2-architecture.md
├── v2-data-model.md
├── workflows.md
├── setup.md
└── security.md
```

## Documentation

### Architecture evolution

- [v1 Architecture](docs/architecture.md)
- [Known Limitations of v1](docs/known-limitations-v1.md)
- [v1 → v2 Architecture Evolution](docs/v1-to-v2.md)
- [v2 Architecture](docs/v2-architecture.md)

### Data and runtime

- [v1 Data Model](docs/data-model.md)
- [v2 Runtime Data Model](docs/v2-data-model.md)
- [Workflow Guide](docs/workflows.md)

### Deployment and sanitization

- [Setup Notes](docs/setup.md)
- [Security and Sanitization](docs/security.md)

## File handling evolution

v1 stores received files through Cloudinary before linking them back to Airtable.

v2 introduces a transient PostgreSQL `file_cache`:

```text
Telegram
   ↓
PostgreSQL file_cache
   ↓
temporary token
   ↓
File Upload workflow
   ↓
Airtable attachment
```

The supplied direct-upload path enforces a 5 MB limit. Larger or unknown-size attachments use a separate administrator-forwarding path in Telegram ingress.

## Not RAG

Neither published architecture should be described as RAG.

The supplied workflows do not show:

- document chunking;
- embeddings;
- a vector database;
- semantic document retrieval.

v2 has a stronger knowledge model through confirmed `facts`, explicit message history and owner-request state, but that is not the same as retrieval-augmented generation.

## Public sanitization

Production-specific configuration was removed before publication.

Examples of public placeholders include:

```text
YOUR_AIRTABLE_BASE_ID
YOUR_V2_TASK_TABLE_ID
YOUR_V2_DIALOG_TABLE_ID
YOUR_V2_OWNER_REQUEST_TABLE_ID
YOUR_V2_ATTACHMENTS_FIELD_ID
YOUR_N8N_PUBLIC_BASE_URL
YOUR_V2_FILE_DOWNLOAD_PATH
```

Credential bindings, production Airtable identifiers, cached Airtable URLs, generated n8n webhook metadata and instance-specific deployment references were removed.

See [Security and Sanitization](docs/security.md) for details.

## Main technologies

- n8n
- PostgreSQL
- Airtable
- OpenAI chat model node
- TelePilot
- Telegram
- Cloudinary in v1

## Project status

**v1 complete.**

The first architecture is preserved as the baseline implementation and its limitations are documented rather than hidden.

**v2 complete.**

The redesigned architecture, all seven sanitized workflows, runtime/data-model documentation, setup guide, security notes and v1→v2 comparison are now published.

The repository therefore represents both the working baseline and the architectural redesign that followed it.

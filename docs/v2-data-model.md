# v2 Runtime Data Model

## Scope

This document describes the data model that is **visible in the supplied v2 workflow exports**.

The exports do not contain the original PostgreSQL DDL, migrations, full Airtable schema definitions, index definitions or constraint definitions. Therefore:

- field names below are taken from SQL, Airtable mappings and workflow expressions;
- relationships are described only where the workflows use them;
- PostgreSQL types are stated only when they are explicit in casts/operations;
- no undocumented primary keys, foreign keys, defaults or indexes are invented.

The source workflows use the internal PostgreSQL schema name `agent_v3` and Airtable tables with `_v3` names. The repository calls this architecture **v2** because it is the second public iteration of the project.

## High-level model

```text
Airtable
┌────────────────────┐
│ agent_tasks_v3     │
│ task/config layer  │
└─────────┬──────────┘
          │ linked dialogs
          ▼
┌────────────────────┐
│ agent_dialogs_v3   │
│ admin-facing state │
└─────────┬──────────┘
          │ mirrored by airtable_dialog_id
          ▼
PostgreSQL
┌──────────────────────────────┐
│ agent_v3.dialogs             │
│ runtime conversation state   │
└──────┬────────┬────────┬─────┘
       │        │        │
       │        │        └──────────────┐
       │        │                       │
       ▼        ▼                       ▼
┌────────────┐ ┌──────────────┐ ┌──────────────────────┐
│ events     │ │ messages     │ │ owner_requests       │
│ work queue │ │ transcript   │ │ human escalation     │
└────────────┘ └──────────────┘ └──────────┬───────────┘
                                           │ confirmed answer
                                           ▼
                                  ┌──────────────────────┐
                                  │ facts                │
                                  │ confirmed knowledge  │
                                  └──────────────────────┘

Attachments:
Telegram → agent_v3.file_cache → Airtable Attachments

Admin mirror for escalation:
agent_v3.owner_requests ↔ Airtable agent_owner_requests_v3
```

# Airtable control-plane entities

## 1. `agent_tasks_v3`

Role: task-level configuration and aggregate state.

Observed fields referenced by the workflows include:

| Field | Observed purpose |
|---|---|
| `id` | Airtable task record id used as `airtable_task_id` in PostgreSQL |
| `Name_task` | task display name |
| `Status` | task lifecycle: observed `Active`, `Paused`, `Done` |
| `agent_dialogs_v3` | linked dialog records |
| `goal` | business/conversation goal supplied to Agent Core |
| `required_result` | required outcome supplied to Agent Core |
| `assistant_role` | role instructions |
| `assistant_identity` | identity/persona instructions |
| `information_base` | task-level knowledge/context |
| `rules` | task rules |
| `communication_style` | communication style |
| `timezone` | scheduling timezone |
| `work_days` | allowed work weekdays |
| `work_start` | beginning of communication window |
| `work_end` | end of communication window |
| `delay_min_sec` | minimum random response/start delay |
| `delay_max_sec` | maximum random response/start delay |
| `followup_after_work_minutes` | work-time delay before a follow-up |
| `max_followups` | maximum follow-up count |
| `admin_telegram_chat_id` | preferred administrator Telegram route when already known |
| `admin_telegram_username` | fallback administrator Telegram username |
| `result` | aggregate task result built from dialog results |

The Task Watcher reads only tasks with `Status = Active` for normal scheduling. Separate runtime logic handles Paused/Done transitions.

## 2. `agent_dialogs_v3`

Role: administrator-facing representation of one conversation/dialog.

Observed fields include:

| Field | Observed purpose |
|---|---|
| `id` | Airtable dialog record id; mirrored to PostgreSQL `airtable_dialog_id` |
| `task` | linked task record |
| `person_name` | target person name |
| `person_role` | target person's role/context |
| `person_context` | additional person-specific context |
| `telegram_username` | Telegram username used before a numeric route is known |
| `telegram_user_id` | resolved Telegram user id |
| `telegram_chat_id` | resolved Telegram chat id |
| `Status` | observed values: `New`, `Scheduled`, `Active`, `Waiting user`, `Waiting owner`, `Done`, `Error` |
| `waiting_for` | description of information currently expected |
| `followup_count` | current follow-up counter |
| `last_activity_at` | last activity timestamp |
| `next_action_at` | next scheduled runtime action |
| `summary` | completed-dialog summary |
| `final_result` | completed-dialog result |
| `finished_at` | completion timestamp |
| `file_urls` | tokenized/public file URLs used by the attachment helper |
| `Attachments` | Airtable attachment field |
| `unload_log` | checkbox/action request for transcript export |
| `update_interim` | checkbox/action request for interim analysis |
| `update_characteristic` | checkbox/action request for dialog/person analysis |

The utility workflow also writes derived outputs back to the dialog record. Exact full Airtable schema is not present in the exports, so this document does not claim fields that are not explicitly referenced.

## 3. `agent_owner_requests_v3`

Role: administrator-facing mirror of a human escalation request.

Observed create mapping:

| Field | Purpose |
|---|---|
| `Request` | display identifier such as request id + task name |
| `task` | linked task |
| `dialog` | linked dialog |
| `question` | question sent to the administrator |
| `context` | optional explanation/context |
| `Status` | created as `Open` |
| `owner_telegram_message_id` | Telegram message id of the admin request |
| `created_at` | request creation timestamp |

The PostgreSQL runtime record remains the authoritative object used for matching answers and scheduling the `owner_answer` event.

# PostgreSQL runtime entities

## 4. `agent_v3.dialogs`

Role: durable runtime state for one Telegram conversation.

Observed fields:

| Field | Observed use |
|---|---|
| `id` | internal dialog identifier used by related runtime tables |
| `airtable_dialog_id` | link back to `agent_dialogs_v3` |
| `airtable_task_id` | link back to `agent_tasks_v3` |
| `person_name` | target person name |
| `person_role` | target person role |
| `telegram_username` | Telegram username |
| `telegram_user_id` | numeric user route once known |
| `telegram_chat_id` | numeric chat route once known |
| `status` | runtime state |
| `waiting_for` | current user-side waiting condition |
| `followup_count` | follow-up counter |
| `summary` | final/near-final summary |
| `final_result` | dialog result |
| `next_action_at` | next planned runtime action |
| `started_at` | first-start timestamp |
| `finished_at` | completion timestamp |
| `last_activity_at` | latest runtime activity |
| `last_inbound_at` | latest incoming Telegram activity |
| `last_outbound_at` | latest outgoing agent activity |
| `metadata` | JSONB-style runtime metadata |
| `created_at` | used by routing/order logic |
| `updated_at` | runtime update timestamp |

Observed runtime statuses include:

- `new`
- `scheduled`
- `active`
- `waiting_user`
- `waiting_owner`
- `done`
- `error`

Task state is also mirrored inside `metadata.task_status`.

Observed metadata keys include:

- `task_status`
- `task_state_synced_at`
- `airtable_task_id`
- `completion_reason`
- `completion_notify_pending`
- `completion_notify_created_at`
- `completion_notify_locked_at`
- `completion_notify_last_claim_at`
- `completion_notify_sent_at`

### Creation/scheduling relationship

Task Watcher upserts a runtime dialog by `airtable_dialog_id`. For a new Airtable dialog it sets runtime status to `scheduled`, calculates `next_action_at`, and creates a `start_dialog` event.

## 5. `agent_v3.events`

Role: durable unit-of-work queue for the runtime.

Observed fields:

| Field | Observed use |
|---|---|
| `id` | event id |
| `dialog_id` | owning runtime dialog |
| `event_type` | semantic work type |
| `run_at` | earliest execution time |
| `status` | queue state |
| `payload` | JSONB-style event-specific data |
| `attempts` | incremented when an event is claimed |
| `locked_at` | claim lock timestamp |
| `processed_at` | completion timestamp |
| `last_error` | stored processing error |
| `updated_at` | update timestamp |

Observed event types:

- `start_dialog`
- `process_inbound`
- `owner_answer`
- `followup`

Observed statuses:

- `pending`
- `processing`
- `done`
- `error`
- `cancelled`

The Event Runner claims due rows with:

```sql
status = 'pending'
AND run_at <= NOW()
ORDER BY run_at, id
FOR UPDATE SKIP LOCKED
LIMIT 10
```

and immediately moves them to `processing`.

### Payload examples

Different event types store different metadata in `payload`.

Observed keys include:

- `airtable_task_id`
- `airtable_dialog_id`
- `last_message_id`
- `last_telegram_message_id`
- `waiting_for`
- `followup_number`
- `origin_event_id`
- `owner_request_id`
- `answer_scope`
- `deferred_by_task_status`
- `deferred_original_run_at`
- `deferred_at`

The exact JSON schema is event-type dependent and is not separately defined in the supplied exports.

## 6. `agent_v3.messages`

Role: canonical runtime conversation history.

Observed fields:

| Field | Observed use |
|---|---|
| `id` | internal message row id |
| `dialog_id` | owning dialog |
| `telegram_message_id` | Telegram message id when one exists |
| `direction` | observed `inbound` / `outbound` |
| `sender_type` | observed `user`, `agent`; utility code also recognizes `owner` |
| `message_type` | text/photo/document/etc. classification |
| `text` | message text/caption/derived text |
| `telegram_created_at` | Telegram-side timestamp |
| `created_at` | database timestamp used as fallback/order field |
| `metadata` | JSONB-style source/event metadata |

The workflows repeatedly use a conflict boundary on:

```text
(dialog_id, telegram_message_id)
```

when `telegram_message_id` is not null.

Observed message metadata includes values such as:

- source `telepilot`;
- raw Telegram message JSON;
- originating `event_id`;
- semantic source such as `start_dialog`, `owner_answer_nonblocking`, `ask_owner_nonblocking` or `complete_dialog`;
- completion reason for a final outbound message.

Dialog Tools reconstructs the transcript from this table ordered by Telegram/database time and row id.

## 7. `agent_v3.owner_requests`

Role: durable human-in-the-loop request.

Observed fields:

| Field | Observed use |
|---|---|
| `id` | internal owner request id |
| `airtable_owner_request_id` | mirror record id in Airtable |
| `airtable_task_id` | owning task |
| `dialog_id` | originating dialog |
| `question` | question for administrator |
| `context` | optional context |
| `status` | request lifecycle |
| `answer` | administrator answer |
| `owner_telegram_message_id` | Telegram id of the question sent to admin |
| `owner_answer_telegram_message_id` | Telegram id of the admin answer |
| `answered_at` | answer timestamp |
| `cancelled_at` | cancellation timestamp |
| `metadata` | answer scope, source ids and admin route metadata |
| `created_at` | creation timestamp |
| `updated_at` | update timestamp |

Observed statuses:

- `open`
- `answered`
- `cancelled`

Observed metadata keys include:

- `answer_scope`
- `source_dialog_id`
- `source_event_id`
- `admin_chat_id`
- `admin_username`

The Agent Core uses `answer_scope` values:

- `task`
- `dialog`

The creation path is idempotency-oriented: it checks the pair of `dialog_id` and source-event metadata so the same semantic event does not create uncontrolled duplicate owner requests.

## 8. `agent_v3.facts`

Role: confirmed reusable context.

Observed fields:

| Field | Observed use |
|---|---|
| `id` | internal fact id |
| `airtable_task_id` | owning task |
| `dialog_id` | optional dialog scope |
| `scope` | observed `task` or `dialog` |
| `category` | observed category such as `owner_answer` |
| `fact_text` | confirmed fact text |
| `source_type` | observed `owner` |
| `is_confirmed` | only confirmed facts are loaded into Agent Core context |
| `is_active` | only active facts are loaded |
| `metadata` | source request/question/scope metadata |

When an administrator answers an owner request, the Telegram ingress:

1. marks the request `answered`;
2. creates a confirmed fact;
3. creates an `owner_answer` event;
4. schedules that event inside the task's working window.

For `scope = task`, `dialog_id` is stored as null. For `scope = dialog`, the current dialog id is stored.

This is an explicit confirmed-knowledge layer. It is not a vector store and does not imply RAG.

## 9. `agent_v3.file_cache`

Role: temporary binary transport between Telegram intake and Airtable attachment upload.

Observed fields:

| Field | Observed use |
|---|---|
| `token` | generated file token used in the temporary URL |
| `dialog_id` | owning runtime dialog |
| `airtable_dialog_id` | Airtable dialog mirror |
| `telegram_message_id` | source Telegram message |
| `file_name` | original/derived filename |
| `mime_type` | MIME type |
| `file_size_bytes` | byte count |
| `file_data` | PostgreSQL BYTEA-style binary data |
| `expires_at` | cache expiry |

The ingress creates tokens using an MD5 expression over random/time/message/dialog inputs and stores cached bytes for 24 hours.

The File Upload workflow:

1. extracts the token from the generated file URL;
2. loads a non-expired cache row;
3. reconstructs the binary payload;
4. validates byte count;
5. uploads supported files directly to Airtable;
6. deletes the cache row after successful upload.

The supplied direct Airtable upload path rejects files above 5 MB.

# Relationships visible in the workflows

## Task → dialogs

```text
Airtable agent_tasks_v3
        │
        │ one task may link multiple dialogs
        ▼
Airtable agent_dialogs_v3
        │
        │ airtable_dialog_id / airtable_task_id
        ▼
PostgreSQL agent_v3.dialogs
```

The runtime explicitly supports multiple dialogs per task. Completing one dialog does not automatically close the task.

## Dialog → events

```text
agent_v3.dialogs.id
        │
        └──► agent_v3.events.dialog_id
```

A dialog may accumulate multiple events over time, although some event classes are deduplicated or rescheduled rather than blindly inserted.

## Dialog → messages

```text
agent_v3.dialogs.id
        │
        └──► agent_v3.messages.dialog_id
```

Messages are the chronological conversation record used for transcript reconstruction and Agent Core context.

## Dialog → owner requests

```text
agent_v3.dialogs.id
        │
        └──► agent_v3.owner_requests.dialog_id
```

Open owner requests are unresolved human dependencies associated with one dialog.

## Owner request → fact → owner_answer event

```text
owner_requests(open)
        │ administrator reply
        ▼
owner_requests(answered)
        │
        ├──► facts(scope = task|dialog)
        │
        └──► events(event_type = owner_answer)
```

The fact persists the confirmed answer; the event re-enters the Agent Core so the clarification can be communicated asynchronously.

## Dialog → file cache

```text
Telegram attachment
        │
        ▼
agent_v3.file_cache
        │ token
        ▼
Airtable dialog file_urls
        │
        ▼
File Upload workflow
        │
        ▼
Airtable Attachments
```

# State synchronization

There are two visible state layers:

1. **Airtable task/dialog state** — administrator-facing control plane.
2. **PostgreSQL runtime state** — execution state.

The State Reconciler copies task state into `agent_v3.dialogs.metadata.task_status`.

Observed behavior:

| Airtable task status | Runtime effect |
|---|---|
| `Active` | runtime can remain routable; deferred events may be resumed by Task Watcher |
| `Paused` | `next_action_at` is cleared and pending/processing events can be cancelled/deferred |
| `Done` | active runtime dialog states are converted to `done`; waiting/scheduled work is cleared/cancelled |

Telegram ingress only routes to dialogs whose runtime state is routable and whose mirrored task state is Active.

It also fails if more than one active dialog matches the same Telegram route.

# Dialog state mapping

Observed Airtable and PostgreSQL values differ only in formatting/case.

| Airtable dialog status | Runtime dialog status |
|---|---|
| `New` | `new` |
| `Scheduled` | `scheduled` |
| `Active` | `active` |
| `Waiting user` | `waiting_user` |
| `Waiting owner` | `waiting_owner` |
| `Done` | `done` |
| `Error` | `error` |

The workflows sometimes keep the dialog `active` or `waiting_user` while an owner request is open because owner escalation is designed to be non-blocking.

# Completion metadata

Dialog completion updates both normalized columns and metadata.

Normalized columns include:

- `status = done`;
- `summary`;
- `final_result`;
- `finished_at`;
- cleared `waiting_for`;
- cleared `next_action_at`.

Metadata includes:

- `completion_reason`;
- `completion_notify_pending`;
- `completion_notify_created_at`.

The separate Completion Notifier later maintains additional lock/sent metadata. This separation allows dialog completion to be durable even if administrator notification delivery fails temporarily.

# What is intentionally not claimed

The supplied workflows do **not** provide enough evidence to publish:

- exact PostgreSQL CREATE TABLE statements;
- exact data types for every field;
- exact primary-key definitions;
- complete foreign-key definitions;
- complete unique constraints/indexes;
- every Airtable field in each table;
- database migration history.

Those should only be added later if the original schema/migration files are available.

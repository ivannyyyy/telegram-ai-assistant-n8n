# Setup Notes

The public workflows are sanitized portfolio exports, not one-click production deployment files.

This repository contains two architecture iterations:

- `workflows/v1/` — first published production-oriented version;
- `workflows/v2/` — redesigned runtime with durable PostgreSQL events, explicit dialog/message/fact state and asynchronous human escalation.

The source v2 exports internally use names such as `V3` and the PostgreSQL schema `agent_v3`. Those internal implementation names are intentionally preserved where they are part of the runtime logic.

# v1 setup

## 1. Import workflows

Import the files from `workflows/v1/` into n8n.

All public workflows are intentionally inactive.

## 2. Reconnect v1 credentials

Configure credentials for the integrations used by v1:

- Airtable
- PostgreSQL
- OpenAI
- TelePilot
- Telegram
- Cloudinary

Credential IDs and credential names were removed from the public JSON.

## 3. Replace v1 placeholders

Replace:

- `YOUR_AIRTABLE_BASE_ID`
- `YOUR_ASSISTANT_TABLE_ID`
- `YOUR_TASK_TABLE_ID`
- `YOUR_TELEGRAM_ADMIN_CHAT_ID`
- `YOUR_TELEGRAM_SECONDARY_ADMIN_CHAT_ID`

The observed v1 fields are documented in `data-model.md`.

## 4. Verify v1 prompt fields

The v1 incoming-message workflow references `promt_incoming_message`.

The characteristics workflow references `Promt_comunication`.

These fields must exist in the corresponding Airtable task data if the v1 workflows are imported as published.

## 5. Verify v1 PostgreSQL dependencies

The supplied v1 workflows reference:

- `public.user_task`
- the PostgreSQL table used by the n8n chat-memory node, observed as `n8n_chat_histories`

The original DDL is not included in the exported source files.

## 6. Reconnect the v1 error workflow

The production exports referenced an instance-specific error-workflow ID. That reference was removed.

After importing `11_error_handler.json`, configure the error workflow manually in n8n where required.

# v2 setup

## 7. Import the seven v2 workflows

Import:

```text
workflows/v2/
├── 01_task_watcher.json
├── 02_telegram_ingress.json
├── 03_event_runner_agent_core.json
├── 04_completion_notifier.json
├── 05_state_reconciler.json
├── 06_file_upload.json
└── 07_dialog_tools.json
```

All are intentionally published with:

```json
"active": false
```

## 8. Reconnect v2 credentials

The v2 workflows require the integrations visible in the supplied exports:

- Airtable
- PostgreSQL
- OpenAI
- TelePilot

Not every workflow uses every integration.

Credential references were removed from every public v2 JSON file.

## 9. Replace v2 Airtable placeholders

Replace:

- `YOUR_AIRTABLE_BASE_ID`
- `YOUR_V2_TASK_TABLE_ID`
- `YOUR_V2_DIALOG_TABLE_ID`
- `YOUR_V2_OWNER_REQUEST_TABLE_ID`

The expected logical tables are:

- `agent_tasks_v3`
- `agent_dialogs_v3`
- `agent_owner_requests_v3`

Their observed fields and relationships are documented in `v2-data-model.md`.

## 10. Prepare the PostgreSQL runtime

The supplied v2 workflows reference:

- `agent_v3.dialogs`
- `agent_v3.events`
- `agent_v3.messages`
- `agent_v3.facts`
- `agent_v3.owner_requests`
- `agent_v3.file_cache`

The original CREATE TABLE statements, migrations, full index definitions and full constraint definitions are not included in the supplied exports.

Therefore the repository documents the observed model but does not provide a fabricated SQL bootstrap script.

## 11. Configure the file pipeline

The public v2 exports use these additional placeholders:

- `YOUR_N8N_PUBLIC_BASE_URL`
- `YOUR_V2_FILE_DOWNLOAD_PATH`
- `YOUR_V2_ATTACHMENTS_FIELD_ID`

`02_telegram_ingress.json` creates temporary file URLs from the configured n8n public base/path.

`06_file_upload.json` exposes the preserved webhook path:

```text
v3-upload-attachments
```

and resolves cached bytes from `agent_v3.file_cache` before uploading an Airtable attachment.

The supplied direct-upload path rejects files above 5 MB. Telegram ingress has a separate administrator-forwarding branch for larger or unknown-size files.

## 12. Configure administrator routing

v2 reads administrator routing from task configuration rather than publishing a hard-coded production chat ID.

Observed task-level fields include:

- `admin_telegram_chat_id`
- `admin_telegram_username`

The Completion Notifier and owner-request path can resolve/cache the administrator route through these fields.

## 13. Verify task-level scheduling fields

v2 scheduling uses task configuration such as:

- `timezone`
- `work_days`
- `work_start`
- `work_end`
- `delay_min_sec`
- `delay_max_sec`
- `followup_after_work_minutes`
- `max_followups`

The supplied code includes fallback values when some fields are absent, including `Europe/Moscow` and weekday defaults.

Review these values before using the workflows in another deployment.

## 14. Verify the Agent Core contract

The public `03_event_runner_agent_core.json` preserves the v2 runtime action contract:

- `send_message`
- `ask_owner`
- `no_action`
- `complete_dialog`

The public Agent Core prompt is a sanitized portfolio version rather than a verbatim production prompt. The routing/state contract, factual-grounding behavior and asynchronous owner-escalation model are preserved.

## 15. Webhook metadata

Generated n8n `webhookId` values were removed from public exports.

Re-save the webhook workflows on the target n8n instance so deployment-specific webhook metadata is generated there.

## 16. Validate before activation

The exports do not include a production deployment manifest or an authoritative activation sequence.

Before activation:

1. verify Airtable fields and linked-record relationships;
2. verify the `agent_v3` PostgreSQL schema;
3. reconnect all required credentials;
4. replace every public placeholder;
5. test Telegram routing with non-production users;
6. test owner-request routing;
7. test file handling with small and large attachments;
8. test Paused/Done reconciliation;
9. test completion notifications;
10. activate scheduled workers only after the dependencies they call are ready.

Use non-production Airtable/PostgreSQL data and a test Telegram account for validation.

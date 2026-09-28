# Workflow Guide

| File | Responsibility |
|---|---|
| `01_start_task.json` | Polls Airtable for a new assistant conversation, loads task context, creates the opening Telegram message, stores initial conversation state and marks the record `In progress`. |
| `02_incoming_message.json` | Main runtime: receives Telegram updates, resolves active conversation state, handles text/files, writes message history, runs the dialogue LLM, validates structured output, routes `send_message` / `wait_admin` / `finish`, queues out-of-hours responses and completes finished tasks. |
| `03_admin_answer.json` | Detects task records marked with new administrator information, finds unresolved dialogue rows, generates a narrowly scoped answer and sends or queues it. |
| `04_wait_message.json` | Polls PostgreSQL rows with `wait = true` and delivers queued messages when the task-level communication window is open. |
| `05_follow_up_on_pause.json` | Checks an active conversation for an unresolved promise by the user and may send a short follow-up after the v1 working-time threshold. |
| `06_stop_and_timeout.json` | Handles manual stop and `days_limit` timeout, updates Airtable/PostgreSQL and clears chat memory. |
| `07_upload_documents_to_airtable.json` | Webhook helper that converts URLs from `docs_url` into Airtable attachment objects. |
| `08_dialog_characteristics.json` | Builds a chronological dialogue transcript and passes it to an LLM using the task-configured `Promt_comunication`; stores the returned analysis in Airtable. |
| `09_dialog_log_export.json` | Builds a chronological dialogue transcript and stores it in the Airtable `Log` field. |
| `10_interim_result.json` | Builds the dialogue transcript and asks the LLM for a structured intermediate progress report against the task goal. |
| `11_error_handler.json` | Error-trigger workflow that sends administrator notifications. |

## Main runtime statuses

### Airtable conversation status

Observed values used by workflows:

- `new`
- `In progress`
- `stop`
- `Done`

### Airtable task status

The admin-answer workflow looks for:

- `new_info`

and then updates the processed task record to:

- `Done`

### LLM routing status

The incoming-message parser allows only:

- `send_message`
- `wait_admin`
- `finish`

### PostgreSQL message state

The `user_task` rows use:

- `wait` — delayed delivery flag;
- `airtable_status` — active/done conversation state;
- `status_bot_message` — normally `send`, but also used to persist a pending administrator question.

## File handling

The incoming-message workflow distinguishes text from supported file content. For the stored-document path it accepts Telegram photos and generic documents while filtering audio/video/GIF-style media from that branch. It validates the downloaded file again and enforces a size below 100 MB before converting and uploading it.

## Transcript utilities

The log, interim-result and characteristics workflows reconstruct dialogue events from `user_task`, normalize old/new date-field variants, sort events chronologically and generate a readable transcript. The transcript builder intentionally does not use Telegram message IDs as the only deduplication key because the source code notes that IDs may repeat in older rows.

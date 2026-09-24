# Data Model

This document describes only fields that are visible or referenced in the supplied n8n exports.

## Airtable: `task`

Observed fields:

| Field | Purpose in v1 |
|---|---|
| `task_number` | Human-readable task identifier. |
| `Name_task` | Task name; linked from per-user assistant records. |
| `person_role` | Role of the target person. |
| `assistant_role` | Role used by the AI assistant. |
| `assistant_ info` | Assistant identity/context injected into prompts. |
| `rules` | Task-specific communication rules. |
| `Information_base` | Task-level knowledge injected into the dialogue prompt. |
| `goal` | Goal of the conversation. |
| `start_bot` | Beginning of allowed communication interval, stored as a numeric seconds-of-day value in v1 expressions. |
| `stop_bot` | End of allowed communication interval. |
| `Status` | Task operational state; `new_info` and `Done` are used by the admin-answer flow. |
| `promt_incoming_message` | Referenced dynamically by the incoming-message workflow; required for the dialogue-agent output contract. |
| `Promt_comunication` | Referenced dynamically by the characteristics workflow; controls the analysis requested from the LLM. |

The last two fields are referenced by runtime expressions even though they are not present in every embedded Airtable schema snapshot inside the exports. They should therefore be treated as required v1 configuration fields.

## Airtable: `ai_assistant`

Observed fields include:

| Field | Purpose in v1 |
|---|---|
| `task_number` | Conversation/task identifier. |
| `Status` | Per-target lifecycle state. |
| `Name_task` | Link/reference to the `task` record. |
| `telegram_username` | Telegram username used to find the target chat. |
| `person_name` | Human-readable target name. |
| `created_at` | Start timestamp for the conversation. |
| `days_limit` | Maximum task duration used by timeout logic. |
| `final_result` | Stored final outcome. |
| `docs_url` | Newline/list-style storage for received file URLs. |
| `docs` | Airtable attachment field populated by helper workflow. |
| `Log` | Generated dialogue transcript. |
| `interim_result` | Generated progress report. |
| `characteristic_person` | Generated dialogue/person analysis output. |

Additional fields embedded in Airtable node schemas but not central to the runtime path include `person_role`, `assistant_role`, `goal`, `communication_rules`, `required_result` and formula/read-only helper fields. The main conversation prompts load role/goal/rules from the linked `task` record.

## PostgreSQL: `user_task`

The n8n nodes expose the following columns:

| Column | n8n type | Use |
|---|---|---|
| `id` | number | Row identifier and update key. |
| `airtable_id` | string | Links event rows to a per-target Airtable conversation record. |
| `tg_user_id` | string | Telegram user identifier. |
| `tg_name` | string | Telegram username/name. |
| `airtable_status` | string | Runtime conversation state. |
| `user_message_id` | string | Telegram user-message ID. |
| `user_message_data` | dateTime | Timestamp of user-side event. |
| `user_message_text` | string | User text or normalized file marker. |
| `bot_message_id` | string | Telegram assistant-message ID. |
| `bot_message_date` | dateTime | Assistant-message timestamp. |
| `bot_message_text` | string | Outgoing assistant text. |
| `status_bot_message` | string | Send state or pending administrator question. |
| `wait` | boolean | Whether outgoing text is queued for later delivery. |

The supplied files do not include the original PostgreSQL DDL, constraints or indexes, so this repository intentionally does not claim an exact production SQL schema.

## PostgreSQL chat memory

The LangChain PostgreSQL memory node uses `n8n_chat_histories` and a composite session ID built from:

```text
airtable_id:tg_user_id
```

The stop/finish paths delete memory for that session when the dialogue is closed.

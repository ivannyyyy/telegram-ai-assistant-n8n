# Setup Notes

The public workflows are sanitized portfolio exports, not one-click production deployment files.

## 1. Import workflows

Import files from `workflows/v1/` into n8n.

All workflows are intentionally exported as inactive.

## 2. Reconnect credentials

Configure credentials in n8n for:

- Airtable
- PostgreSQL
- OpenAI
- TelePilot
- Telegram
- Cloudinary

Credential IDs and credential names were removed from the public JSON.

## 3. Replace Airtable placeholders

Replace:

- `YOUR_AIRTABLE_BASE_ID`
- `YOUR_ASSISTANT_TABLE_ID`
- `YOUR_TASK_TABLE_ID`

The expected field names are documented in `data-model.md`.

## 4. Configure administrator notifications

Replace:

- `YOUR_TELEGRAM_ADMIN_CHAT_ID`
- `YOUR_TELEGRAM_SECONDARY_ADMIN_CHAT_ID`

If only one administrator is required, both notifications may be routed to the same destination.

## 5. Verify required task prompt fields

The main conversation workflow references `promt_incoming_message`. In v1 this prompt is responsible for instructing the LLM to return the status/message/result contract expected by the parser.

The characteristics workflow references `Promt_comunication`.

These fields should exist in the `task` table.

## 6. Verify PostgreSQL tables

The workflows expect:

- `public.user_task`
- the table used by the n8n PostgreSQL chat-memory node (`n8n_chat_histories` in the supplied exports)

The exact original DDL is not included in the source exports.

## 7. Reconnect the error workflow

Production exports referenced a separate n8n error-workflow ID. That instance-specific reference was removed. After importing `11_error_handler.json`, configure it as the error workflow in n8n for workflows where error notifications are required.

## 8. Webhook helpers

Public exports preserve the original webhook path strings, but generated n8n webhook IDs were removed. Re-save/activate the webhook workflows on your own n8n instance so n8n generates deployment-specific webhook metadata.

## 9. Review timezone assumptions

Several v1 expressions use `Europe/Moscow`, and the follow-up worker prompt assumes weekday working time 09:00–18:00 Moscow time. Change these if the deployment requires another timezone or calendar.

## 10. Activate only after validation

Before activation, test each workflow against non-production Airtable/PostgreSQL data and a test Telegram account.

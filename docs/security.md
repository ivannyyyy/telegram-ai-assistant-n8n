# Security and Sanitization

The public v1 exports were sanitized before packaging.

Removed or replaced:

- n8n credential references and credential names;
- Airtable production base ID;
- Airtable production table IDs;
- cached Airtable production URLs;
- generated `webhookId` values;
- hard-coded Telegram administrator chat IDs;
- top-level n8n workflow IDs;
- `versionId`;
- n8n `meta.instanceId`;
- production error-workflow ID references.

Public workflows are set to:

```json
"active": false
```

The source exports contained credential references, not raw API secrets. Nevertheless, deployment-specific references were removed because they are unnecessary in a public portfolio.

## Placeholders

- `YOUR_AIRTABLE_BASE_ID`
- `YOUR_ASSISTANT_TABLE_ID`
- `YOUR_TASK_TABLE_ID`
- `YOUR_TELEGRAM_ADMIN_CHAT_ID`
- `YOUR_TELEGRAM_SECONDARY_ADMIN_CHAT_ID`

## What remains intentionally visible

The public workflows preserve:

- node structure;
- prompts;
- deterministic JavaScript routing/parsing;
- table/field names required to understand the architecture;
- schedule logic;
- PostgreSQL table names used by the workflow;
- webhook path names;
- third-party node types and integration choices.

These elements are part of the technical portfolio value and are not credentials.

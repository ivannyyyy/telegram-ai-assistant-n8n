# Security and Sanitization

The public workflow exports are portfolio-safe copies. They preserve architecture and implementation detail while removing production identifiers and credential bindings.

All public workflows are intentionally published as:

```json
"active": false
```

# v1 sanitization

Removed or replaced from the v1 source exports:

- n8n credential references and credential names;
- Airtable production base ID;
- Airtable production table IDs;
- cached Airtable production URLs;
- generated `webhookId` values;
- hard-coded Telegram administrator chat IDs;
- top-level n8n workflow IDs;
- `versionId`;
- n8n instance metadata;
- production error-workflow ID references.

v1 placeholders include:

- `YOUR_AIRTABLE_BASE_ID`
- `YOUR_ASSISTANT_TABLE_ID`
- `YOUR_TASK_TABLE_ID`
- `YOUR_TELEGRAM_ADMIN_CHAT_ID`
- `YOUR_TELEGRAM_SECONDARY_ADMIN_CHAT_ID`

# v2 sanitization

The seven public v2 workflows were sanitized independently before publication.

Removed or replaced:

- Airtable credential references and credential labels;
- PostgreSQL credential references and credential labels;
- OpenAI credential references and credential labels;
- TelePilot credential references and credential labels;
- production Airtable base ID;
- production task/dialog/owner-request table IDs;
- cached Airtable production URLs;
- generated n8n `webhookId` values;
- top-level workflow `versionId`;
- n8n `meta` and instance-specific metadata;
- `staticData` when present;
- deployment-specific n8n public host;
- generated file-download webhook identifier;
- production Airtable attachment-field identifier;
- person-specific credential names that are unnecessary for the portfolio.

The source exports contained credential **references**, not raw API keys/tokens. Those references were still removed because they disclose production configuration and provide no portfolio value.

## v2 public placeholders

The currently published v2 workflows use:

- `YOUR_AIRTABLE_BASE_ID`
- `YOUR_V2_TASK_TABLE_ID`
- `YOUR_V2_DIALOG_TABLE_ID`
- `YOUR_V2_OWNER_REQUEST_TABLE_ID`
- `YOUR_V2_ATTACHMENTS_FIELD_ID`
- `YOUR_N8N_PUBLIC_BASE_URL`
- `YOUR_V2_FILE_DOWNLOAD_PATH`

Not every placeholder appears in every workflow.

## Agent Core prompt

The public v2 Event Runner keeps the Agent Core action/state contract and factual-grounding behavior, but its prompt is not a verbatim production prompt.

The public portfolio prompt preserves the architectural behavior needed to understand the implementation:

- `send_message`;
- `ask_owner`;
- `no_action`;
- `complete_dialog`;
- task/dialog fact scope;
- asynchronous owner escalation;
- follow-up handling;
- dialog completion;
- deterministic validation after the model response.

Identity-specific production wording that is unnecessary for a public portfolio was removed or neutralized.

# What remains intentionally visible

The public repository intentionally preserves technical details that are useful for architecture review:

- node names and workflow topology;
- JavaScript routing/parsing code;
- SQL queries and state transitions;
- the internal PostgreSQL schema name `agent_v3`;
- runtime table names;
- Airtable logical table/field names;
- event types and statuses;
- scheduling and retry/lock behavior;
- public webhook path names where they are part of the workflow contract;
- third-party node types and integration choices;
- non-secret model/configuration choices visible in the workflow.

These elements describe the system and are not authentication secrets.

# Items that still require deployment review

Sanitization does not make the exports one-click production templates.

Before deploying a public workflow copy, review:

- all placeholder substitutions;
- environment-specific webhook/base URLs;
- Airtable table/field mappings;
- administrator Telegram routing;
- PostgreSQL permissions;
- n8n execution/error settings;
- data-retention requirements for messages, facts, owner requests and file cache;
- access controls around administrator-facing Airtable tables.

The repository documents what was visible in the supplied exports; it does not claim a complete security architecture for the production environment.

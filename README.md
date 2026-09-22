# Human-in-the-Loop Telegram AI Assistant

<p align="center">
  <img src="https://raw.githubusercontent.com/ivannyyyy/ivannyyyy/main/assets/telegram-ai-assistant-pixel.png" width="100%" alt="Telegram AI Assistant — stateful dialogue, memory and human escalation architecture" />
</p>

A stateful AI assistant built with n8n for task-driven Telegram conversations.

The administrator defines a task, assistant role, target person, communication rules, working hours, goals and task-level knowledge in Airtable. The system can initiate a Telegram conversation, continue it with persistent context, pause delivery outside the allowed time window, ask an administrator for missing information, resume the dialogue after the knowledge base is updated, and produce operational summaries of the conversation.

This repository contains the sanitized public version of **v1**. A later v2 is intended to show how the architecture evolved.

## What this project demonstrates

- AI-agent orchestration with n8n
- task-driven outbound Telegram conversations
- persistent conversation state in PostgreSQL
- separate LangChain chat memory
- deterministic routing around LLM decisions
- human-in-the-loop escalation
- delayed delivery based on configured working hours
- lifecycle states such as `new`, `In progress`, `stop`, and `Done`
- follow-up logic for unresolved user promises
- file intake and Cloudinary storage
- admin-facing logs, interim results and dialogue analysis
- error notification workflow

## Architecture

```text
                         ┌──────────────────────────┐
                         │      Airtable Admin      │
                         │ task + assistant config  │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                         ┌──────────────────────────┐
                         │     Start Task Worker    │
                         │ selects Status = new     │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                              LLM opening message
                                      │
                                      ▼
┌───────────┐        ┌──────────────────────────┐        ┌───────────────────┐
│ Telegram  │◄──────►│ Incoming Message Worker  │◄──────►│ PostgreSQL        │
│ TelePilot │        │ + deterministic router   │        │ user_task + memory│
└───────────┘        └────────────┬─────────────┘        └───────────────────┘
                                  │
                ┌─────────────────┼──────────────────┐
                │                 │                  │
                ▼                 ▼                  ▼
          send_message        wait_admin           finish
                │                 │                  │
                │                 ▼                  ▼
                │         notify administrator   mark Done
                │                 │              store result
                │                 ▼
                │       Airtable knowledge update
                │                 │
                │                 ▼
                │         Admin Answer Worker
                │                 │
                └─────────────────┴──────► Telegram
```

## Core state flow

The main LLM response is normalized and validated before routing. The v1 parser accepts only:

```json
{
  "status": "send_message | wait_admin | finish",
  "message": "text for the Telegram user",
  "result": "admin question or final result when required"
}
```

The deterministic code node rejects unknown statuses, missing messages, missing escalation questions, or missing final results.

## Human-in-the-loop flow

When the assistant cannot answer safely from the available task context:

1. the LLM returns `wait_admin`;
2. the user-facing message is sent or queued according to the configured time window;
3. the unresolved question is stored in PostgreSQL;
4. an administrator is notified;
5. the administrator updates task information in Airtable and marks it as new information;
6. the admin-answer worker generates a narrowly scoped answer;
7. the answer is sent immediately or queued until the allowed communication window;
8. the dialogue continues through the normal incoming-message workflow.

## Data layers

**Airtable** is the administrator-facing control plane. It stores task configuration, target users, task status, knowledge, limits and derived results.

**PostgreSQL** stores conversation events in `user_task` and is also used by the n8n PostgreSQL chat-memory node.

**Telegram / TelePilot** is the conversation channel.

**Cloudinary** stores received files that pass the v1 file checks.

## Repository structure

```text
workflows/
  v1/
    01_start_task.json
    02_incoming_message.json
    03_admin_answer.json
    04_wait_message.json
    05_follow_up_on_pause.json
    06_stop_and_timeout.json
    07_upload_documents_to_airtable.json
    08_dialog_characteristics.json
    09_dialog_log_export.json
    10_interim_result.json
    11_error_handler.json

docs/
  architecture.md
  workflows.md
  data-model.md
  setup.md
  security.md
  known-limitations-v1.md
  publication-plan.md
```

## Important distinction

v1 is **not a RAG implementation**. Task knowledge is injected from the Airtable `Information_base` field. Received documents are stored and linked, but the provided workflows do not parse those documents into embeddings or perform semantic retrieval.

## Public workflow placeholders

Production-specific values were removed from the public exports. Configure these after import:

- `YOUR_AIRTABLE_BASE_ID`
- `YOUR_ASSISTANT_TABLE_ID`
- `YOUR_TASK_TABLE_ID`
- `YOUR_TELEGRAM_ADMIN_CHAT_ID`
- `YOUR_TELEGRAM_SECONDARY_ADMIN_CHAT_ID`

Credential references, workflow instance IDs, webhook IDs, Airtable URLs and n8n instance metadata were removed.

## Required integrations

- n8n
- Airtable
- PostgreSQL
- OpenAI chat model node
- TelePilot n8n community nodes
- Telegram node for administrator notifications
- Cloudinary for received-file storage

Exact product/account versions are intentionally not asserted here because they are not contained in the supplied source exports.

## Status

This repository documents the first production-oriented iteration of the assistant architecture. The main value of v1 is the explicit state and escalation model; known limitations are documented separately so that later revisions can show the architectural progression.

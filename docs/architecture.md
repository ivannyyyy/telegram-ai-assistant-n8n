# Architecture

## 1. Control plane

Airtable acts as the administrator interface. Two logical tables are used by the exported workflows:

- `task` — shared task definition and agent configuration;
- `ai_assistant` — per-target conversation instance and operational state.

The start worker takes a new conversation instance, loads the linked task and generates the first Telegram message.

## 2. Conversation runtime

Incoming Telegram events are matched to active conversation state using the Telegram user ID and an `In progress` state.

Each incoming message is written to PostgreSQL before the LLM continues the dialogue. The LangChain PostgreSQL memory uses a composite session key:

```text
airtable_id:tg_user_id
```

This keeps memory isolated per Airtable conversation instance and Telegram user.

## 3. Deterministic decision boundary

The conversation LLM does not directly decide which n8n branch executes. Its output is parsed by JavaScript and validated against three allowed states:

- `send_message`
- `wait_admin`
- `finish`

Invalid JSON, invalid states and missing required fields fail closed by raising an execution error.

This boundary is one of the strongest architectural elements of v1: AI generates a decision, deterministic code validates it, and workflow logic performs the action.

## 4. Human escalation

`wait_admin` creates a human-in-the-loop branch.

The workflow keeps the user-facing response separate from the concrete question that must be answered by an administrator. The unresolved question is persisted in PostgreSQL. An administrator can update the task-level information in Airtable and mark the task record for processing. The admin-answer workflow then produces a single scoped answer to the unresolved question.

## 5. Time-aware delivery

Conversation messages are checked against task-level `start_bot` and `stop_bot` values. If sending is not allowed at the current time, the row is marked with `wait = true`.

A separate wait-message workflow polls pending rows and sends them when the configured communication window is open.

Several workflow expressions use the `Europe/Moscow` timezone. This is a v1 deployment assumption rather than a generic timezone abstraction.

## 6. Follow-up worker

The follow-up workflow analyzes open dialogue memory and checks whether the user promised to provide information but has not yet done so. The embedded v1 prompt uses a four-working-hour rule and weekday working hours of 09:00–18:00 Moscow time.

If the follow-up is not justified, the model must return `stop`. Otherwise, one short context-specific follow-up is sent or queued.

## 7. Completion and cleanup

The conversation can end in three ways visible in the supplied workflows:

- LLM decision `finish`;
- administrator sets the conversation state to `stop`;
- the configured `days_limit` expires.

Completion updates Airtable, marks conversation rows as done, stores a final result when one exists and removes the corresponding PostgreSQL chat-memory rows.

## 8. File path

Incoming Telegram photos/documents are checked by type and size. The v1 workflow rejects unsupported media branches for this path and enforces a 100 MB limit before upload.

Accepted files are uploaded to Cloudinary and their URLs are appended to Airtable. A separate webhook helper can convert stored URLs into Airtable attachments.

This file path is storage-oriented; it is not a retrieval-augmented generation pipeline.

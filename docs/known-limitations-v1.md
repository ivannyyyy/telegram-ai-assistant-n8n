# Known Limitations of v1

v1 is intentionally preserved as the first production-oriented iteration of the assistant. The points below describe architectural constraints visible in the supplied workflows and provide concrete reasons for a later v2 redesign.

## 1. Polling-heavy orchestration

Several background flows use scheduled polling rather than event-driven triggers.

Examples include:

- searching Airtable for new assistant tasks;
- checking for administrator updates;
- checking queued messages;
- follow-up checks;
- stop and timeout checks.

This makes the system straightforward to operate, but it also means some actions happen only on the next scheduled run and unnecessary queries can occur when nothing has changed.

## 2. Serial selection in key workers

Some v1 workers deliberately process a very small number of records per run.

The start-task flow searches for one new assistant record, and the follow-up flow also works on a limited active selection. This is sufficient for a small deployment but does not yet represent a high-throughput worker model.

A later version can make concurrency, batching and idempotent processing more explicit.

## 3. Strong Airtable coupling

Airtable is both the administrator-facing control plane and an important part of the runtime configuration model.

The workflows depend on specific field names, status values and linked records. This makes the system easy to operate manually, but the orchestration layer is tightly coupled to the Airtable schema.

## 4. Human escalation uses a manual handoff

The human-in-the-loop flow works, but the v1 handoff is operationally simple:

1. the assistant reaches `wait_admin`;
2. the unresolved question is persisted;
3. an administrator updates task knowledge in Airtable;
4. the task is marked `new_info`;
5. a scheduled worker resumes the dialogue.

This is a valid escalation loop, but there is no dedicated approval/inbox object, explicit answer record or event-driven resume trigger.

## 5. No RAG over received documents

v1 should not be described as a RAG system.

Task knowledge is injected from the Airtable `Information_base` field. Received files can be stored in Cloudinary and referenced from Airtable, but the supplied workflows do not implement:

```text
document parsing
    ↓
chunking
    ↓
embeddings
    ↓
vector search
    ↓
context retrieval
```

Document storage and semantic retrieval are separate capabilities.

## 6. Structured output is prompt-defined

The incoming-message workflow has a deterministic parser and rejects invalid or incomplete routing states.

However, the model-side contract is still instructed through the task prompt. The v1 workflow does not use a dedicated schema-bound structured-output component for the LLM response itself.

So the boundary is:

```text
prompt asks for JSON
        ↓
LLM response
        ↓
deterministic JavaScript validation
        ↓
workflow routing
```

The validation layer is deterministic, while the generation contract remains prompt-driven.

## 7. Deployment assumptions are embedded in workflow logic

v1 contains environment-specific scheduling assumptions, especially around Moscow time.

Examples include:

- `Europe/Moscow` expressions;
- weekday follow-up logic;
- a 09:00–18:00 working schedule used by the follow-up prompt.

These values are configuration candidates in a more reusable version.

## 8. Conversation events and LLM memory are separate stores

The architecture uses both:

- `public.user_task` for operational conversation/event state;
- PostgreSQL chat memory for the LangChain agent.

This separation is useful, but it means two representations of conversation history must remain consistent. Completion logic therefore also includes explicit chat-memory cleanup.

## 9. Transcript-building logic is duplicated

The dialogue-characteristics, log-export and interim-result workflows each reconstruct a chronological transcript from `user_task`.

The code handles useful details such as legacy timestamp-field variants, filtering, sorting and deduplication, but similar transcript-building logic appears in multiple utility workflows.

A later version can extract that behavior into one reusable component.

## Why keep these limitations visible?

The purpose of publishing v1 is not to present every design choice as final. It establishes a working baseline with:

- explicit conversation state;
- persistent history;
- deterministic routing around LLM decisions;
- human escalation;
- delayed delivery;
- lifecycle automation;
- operational reporting.

A future v2 can then be compared against these concrete constraints instead of being presented only as a generic “improved version.”

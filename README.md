# MLS Assistant — Internal Process Knowledge Assistant

An AI-powered knowledge system built for a land surveying firm that makes
internal process knowledge easy to find and easy to capture — combining a
RAG-based retrieval pipeline with automated capture of employee process
conversations into a single searchable admin interface.

https://github.com/user-attachments/assets/e77aa39f-3452-4e27-80dc-19753afab662

## Overview

Surveying firms carry a huge amount of process knowledge — how a particular
job type is handled, why a certain workflow decision was made, how to use a
specific piece of internal tooling — that lives in scattered documents and
informal conversations between employees rather than anywhere searchable.
MLS Assistant exists to solve two problems: making that knowledge easy to
*find* (retrieval-augmented Q&A over internal documents and recorded process
conversations) and easy to *collect in the first place* (automated capture
and transcription of process conversations as they happen), rather than
relying on staff to track down whoever remembers the answer.

## What it does

- **Unified knowledge base** — ingests documents and process conversation
  transcripts into a single vector store for semantic search
- **Process Q&A** — employees ask plain-English questions about internal
  processes (e.g. "how do we search for a plan?") and
  get answers sourced from real internal documents and past conversations,
  not hallucinated
- **Automated process knowledge collection** — records and transcribes
  conversations between employees (e.g. a senior surveyor walking a junior
  through a job, workflow decisions, informal training moments) so tacit
  knowledge is captured and made searchable without anyone needing to write
  it down
- **Admin panel** — a Docs / Transcripts interface for browsing, searching,
  and managing ingested content

## Architecture

```
                     ┌──────────────┐     ┌───────────────┐
                     │              │◀────│  mls-capture  │
                     │ mls-assistant│     │  (employee    │
                     │  (FastAPI +  │◀────│   process     │
                     │   pgvector)  │     │  conversation │
                     └──────┬───────┘     │ → transcript) │
                            │             └───────────────┘
                     ┌──────▼───────┐
                     │  Admin Panel │
                     │    Docs /    │
                     │  Transcripts │
                     └──────────────┘
```

**Stack:** FastAPI · PostgreSQL + pgvector · Docker · Claude for
retrieval/answer generation · Whisper + SpeechBrain for transcription/speaker
ID · deployed on-prem via Docker on Windows

## Technical highlights

**Retrieval-augmented generation**
Documents are chunked, embedded, and stored in Postgres via `pgvector`,
letting the system answer questions with citations back to source documents
rather than relying on model memory alone.

**Tacit process knowledge capture**
A lot of institutional knowledge in a small surveying firm lives in informal
conversations — a senior surveyor walking a junior through a tricky boundary
determination, a decision made verbally and never written down. mls-capture
records and transcribes these process conversations (speaker-diarized via
SpeechBrain, transcribed via Whisper) so that knowledge becomes searchable
rather than lost the moment the conversation ends.

**Capture-to-retrieval pipeline**
mls-assistant sits downstream of mls-capture, a separate service that
records and transcribes employee process conversations. Once a conversation
is transcribed and speaker-diarized, mls-assistant automatically drafts a
suggested process write-up from it, which staff can review, edit, and push
into the knowledge base — turning a raw conversation into a structured,
queryable process record with no manual write-up from scratch.

**On-prem deployment on non-technical infrastructure**
Deployed the full stack (multiple Docker containers, Postgres, FastAPI) onto
a client-owned Windows machine with no dedicated IT support — resolved
Docker Desktop networking (`host.docker.internal`), Python version
mismatches, and port conflicts, then wrapped the whole thing in `.bat` files
so a non-technical end user can launch/update the stack with a double-click.

## Example: retrieval query flow

```python
# Simplified from the actual retrieval endpoint
async def query_knowledge_base(question: str) -> AnswerResponse:
    query_embedding = await embed_text(question)

    results = await db.fetch(
        """
        SELECT content, source, metadata
        FROM documents
        ORDER BY embedding <=> $1
        LIMIT 5
        """,
        query_embedding,
    )

    context = "\n\n".join(r["content"] for r in results)
    answer = await claude.generate(
        system=RAG_SYSTEM_PROMPT,
        prompt=f"Context:\n{context}\n\nQuestion: {question}",
    )

    return AnswerResponse(answer=answer, sources=[r["source"] for r in results])
```

## Notes

This is a showcase repository — the full source is part of a private client
deployment. This README documents the architecture and my technical
contributions.

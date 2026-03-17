---
name: customgpt-ai-rag:_check-file-by-id
description: Internal utility — look up a document's status and ID by paginating through the agent's file list. Primarily used by other skills that need a document ID before performing an action.
argument-hint: "[filename or URL to look up]"
allowed-tools: Bash, Read
triggers:
  - "check file"
  - "find file"
  - "lookup file"
  - "file status"
  - "document status"
  - "was added"
  - "was this added"
  - "was processed"
  - "check if added"
  - "check if processed"
  - "did it process"
  - "is this file added"
  - "is this file processed"
  - "find this file in the index"
  - "look up this file"
  - "check this file"
---

# _check-file-by-id

Find a specific file or URL in the agent's knowledge base and report its status. Returns the document ID (called "page ID" in the API) so you can reference it in `/_delete-file-by-id` or `/update-agent`.

## Rules

- ALWAYS read `.customgpt-meta.json` to get `agent_id`
- Match against BOTH `filename` (uploaded files) AND `page_url` (web sources)
- Use case-insensitive substring matching — the user may give a partial name or path segment
- Paginate until a match is found or all documents are checked

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

---

## Step 3 — Get Search Term

If the user provided a filename, path segment, or URL with the command, use it.

Otherwise ask:
> "Which file or URL do you want to look up? You can provide a partial name."

---

## Step 4 — Search by Paginating

Fetch documents 100 at a time, checking each document's `filename` and `page_url` for a case-insensitive substring match against the search term. Stop as soon as a match is found.

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

For each entry in `data.pages.data[]`, check `filename` and `page_url` (both lowercased) against the lowercased search term.

Continue through all results until a match is found or all documents have been checked.

---

## Step 5 — Report

**If found:**

> **Found:** `{filename or page_url}`
>
> | Field | Value |
> |---|---|
> | Document ID | {id} |
> | Type | {is_file ? "uploaded file" : "URL"} |
> | Added | {crawl_status — "ok" means successfully added} |
> | Processed | {index_status — "ok" means ready for queries} |
> | Created | {created_at} |

**Status guidance:**

| Added | Processed | Meaning |
|-------|-----------|---------|
| `queued` | any | Waiting to be added — still in the queue |
| `ok` | `queued` | Added successfully, waiting to be processed |
| `ok` | `ok` | Fully processed and available for queries |
| `limited` | any | Plan limit reached — content was partially processed |
| `failed` | any | Failed to add — see below |
| any | `failed` | Processing failed — see below |

If either status is `failed`:
> "This document failed to process. Try `/update-agent {filename}` to delete and re-upload it. If it continues to fail, check that the file format is supported and the file is not corrupted."

If status is `limited`:
> "Your plan limit was reached while processing this document. Upgrade your CustomGPT.ai plan at https://app.customgpt.ai/billing to process more content."

**If not found:**

> "No document matching '{search_term}' found in the knowledge base."
> "If you added this file recently, it may still be processing — run `/check-status` to see overall progress, or `/list-files` to browse all documents."
> "To add this file, run `/add-files {filename}`."

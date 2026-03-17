---
name: customgpt-ai-rag:_delete-file-by-id
description: Internal utility — permanently remove a document from the agent's knowledge base by its API page ID. Used by other skills that need to delete before re-uploading.
argument-hint: "[document ID or filename — run /list-files to find it]"
allowed-tools: Bash, Read
triggers:
  - "delete file"
  - "delete this file"
  - "delete document"
  - "remove file"
  - "remove document"
  - "remove from agent"
  - "delete from agent"
  - "delete from knowledge base"
  - "remove from knowledge base"
---

# _delete-file-by-id

Permanently remove a document from the agent's knowledge base by its document ID (called "page ID" in the API). The agent will no longer reference this content when answering questions. Cannot be undone — re-upload with `/add-files` to restore.

## Rules

- ALWAYS confirm before deleting — show the document ID and filename/URL to the user
- ALWAYS read `.customgpt-meta.json` to get `agent_id`
- If no document ID provided, look up the document first using the file list

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

---

## Step 3 — Resolve the Document ID

**If the user provided a numeric ID:** use it directly as `$PAGE_ID`.

**If the user provided a filename or URL instead of an ID:** search the file list to find the matching document. Fetch documents until a match is found:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Match `filename` or `page_url` (case-insensitive substring). If found, note the `id`, `filename` or `page_url`.

If not found after checking all documents:
> "No document matching '{term}' found. Run `/list-files` to browse all documents."

**If no identifier provided at all:**
> "Please provide a document ID or filename. Run `/list-files` to see all documents and their IDs."

---

## Step 4 — Confirm Deletion

Present the details and ask for confirmation:

> "About to permanently delete document **{PAGE_ID}** (`{filename or page_url}`) from the agent.
> This cannot be undone. The file must be re-uploaded to restore it.
>
> Continue? (yes/no)"

If not confirmed: "Cancelled. Document was not deleted."

---

## Step 5 — Delete the Document

```bash
curl -s --request DELETE \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${PAGE_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Read the response:
- HTTP 200 → deleted successfully
- HTTP 404 → document not found — it may have already been deleted
- HTTP 401 → invalid API key
- HTTP 400 or 500 → server error; show the response body

---

## Step 6 — Report

On success:
> "Document **{PAGE_ID}** (`{filename or page_url}`) deleted from the agent."
>
> "To restore it, run `/add-files {filename}`. To replace it with a new version, use `/update-agent {filename}`."

On failure:
> "Delete failed (HTTP {status}): {error message from response body}"

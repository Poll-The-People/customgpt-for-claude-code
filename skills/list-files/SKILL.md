---
name: customgpt-ai-rag:list-files
description: List all documents in the CustomGPT.ai agent with their filenames or URLs and processing status. Paginates automatically.
argument-hint: "[optional: filter by status — ok | failed | queued]"
allowed-tools: Bash, Read
triggers:
  - "list files"
  - "list documents"
  - "list added files"
  - "list added documents"
  - "show files"
  - "show added files"
  - "show added documents"
  - "show documents"
  - "what's added"
  - "what is added"
  - "what files are added"
  - "what files were added"
  - "list all files"
  - "show knowledge base"
  - "what's in the knowledge base"
  - "show all files"
---

# list-files

List every document in the agent's knowledge base — their filenames or URLs and processing status. Paginates automatically through all results.

## Rules

- ALWAYS read `.customgpt-meta.json` to get `agent_id`
- Paginate until all documents are collected — use `data.pages.last_page`
- Claude reads and parses the JSON — do not use shell scripts or jq

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

---

## Step 3 — Check for Status Filter

If the user specified a status filter (e.g., "list failed files", "show queued files"), note the filter value. Valid API values: `ok`, `failed`, `queued`, `limited`, `n/a`.

User-facing terms map to API parameters:
- "failed" → `&crawl_status=failed` or `&index_status=failed`
- "queued" / "pending" → `&crawl_status=queued` or `&index_status=queued`
- "ready" / "ok" → `&index_status=ok`

---

## Step 4 — Fetch All Documents

Fetch the first batch:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=1&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

From the response, read:
- `data.pages.data[]` — the array of document objects
- `data.pages.last_page` — total number of API pages
- `data.pages.total` — total document count

For each document, note: `id`, `filename`, `page_url`, `crawl_status`, `index_status`, `is_file`

If `last_page > 1`, repeat for pages 2 through `last_page`:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Collect all document objects across all batches.

---

## Step 5 — Display Results

Present a table. For `is_file: true` entries, show `filename` in the Name column; for URL entries, show `page_url`. Use user-facing terminology for status columns.

| Name | Added | Processed |
|------|-------|-----------|
| skills/ask-agent/SKILL.md | ok | ok |
| README.md                   | ok | failed |

Call out documents that need attention:
- Flag rows where status is `failed` with a note: "Run `/update-agent <filename>` to retry."
- Flag `queued` rows with: "Still being processed — check again with `/check-status`."
- Flag `limited` rows with: "Plan limit reached for this document."

Show totals:
> Total: **{N}** document(s) — {ok_count} ready, {queued_count} pending, {failed_count} failed

Suggest next steps:
> "If you'd like to remove a document or refresh it with a newer version, just let me know."

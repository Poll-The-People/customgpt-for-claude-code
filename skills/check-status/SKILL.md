---
name: customgpt-ai-rag:check-status
description: Show the current status of your CustomGPT.ai agent — how many documents have been added and processed, and whether the agent is ready for queries.
allowed-tools: Bash, Read
triggers:
  - "check status"
  - "agent status"
  - "check agent"
  - "how many files"
  - "how many files added"
  - "how many files processed"
  - "how many documents"
  - "is the agent ready"
  - "is it ready"
  - "is processing done"
  - "processing progress"
  - "status of agent"
  - "are my files processed"
  - "are my files ready"
---

# check-status

Show the current status of a CustomGPT.ai agent: how many documents have been added, how many are processed, and whether the agent is ready to query. Run this after `/create-agent` or `/update-agent` to know when you can start searching.

## Rules

- ALWAYS read `.customgpt-meta.json` to get `agent_id` — never hardcode it
- Present a clean summary — do NOT dump raw JSON

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id` and `indexed_folder`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

---

## Step 3 — Fetch Project Info

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Extract: `data.project_name`, `data.is_chat_active`, `data.created_at`

---

## Step 4 — Fetch Stats

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/stats" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Extract: `data.pages_found`, `data.pages_crawled`, `data.pages_indexed`, `data.total_words_indexed`

---

## Step 5 — Check Processing Status

Make four lightweight calls to get accurate status counts (using `limit=1` — we only need the `total` from pagination metadata):

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?crawl_status=queued&limit=1" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?index_status=queued&limit=1" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?crawl_status=failed&limit=1" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?index_status=failed&limit=1" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

From each response, extract `data.pages.total` to get:
- `queued_add` — documents waiting to be added
- `queued_process` — documents waiting to be processed
- `failed_add` — documents that failed to add
- `failed_process` — documents that failed to process

---

## Step 6 — Report

Present a clean summary:

> **Agent: `{project_name}`**
> Connected folder: `{indexed_folder}`
>
> | Metric | Count |
> |---|---|
> | Documents found | {pages_found} |
> | Documents added | {pages_crawled} |
> | Documents processed | {pages_indexed} |
> | Words processed | {total_words_indexed} |

**Status interpretation — add the relevant note:**

- If `queued_add == 0` and `queued_process == 0` and `failed_add == 0` and `failed_process == 0` and `pages_found > 0`:
  > "All documents processed. Agent is ready — use `/ask-agent` to start searching."

- If `queued_add > 0` or `queued_process > 0`:
  > "Still processing — {queued_add} waiting to be added, {queued_process} waiting to be processed. Check again in a moment."

- If `queued_add == 0` and `queued_process == 0` and (`failed_add > 0` or `failed_process > 0`):
  > "Processing is complete and agent is ready to chat — use `/ask-agent` to start searching. However, {failed_add + failed_process} document(s) failed to process. Run `/list-files` to see which ones."

- If `pages_found == 0`:
  > "No documents found. Run `/add-files` to upload files to this agent."

- If `is_chat_active` is false:
  > "Warning: Queries are disabled for this agent. The agent may still be setting up — check again shortly."

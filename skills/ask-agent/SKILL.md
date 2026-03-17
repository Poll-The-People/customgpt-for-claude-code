---
name: customgpt-ai-rag:ask-agent
description: Ask a plain-language question to your CustomGPT.ai agent and get an AI-generated answer with source citations. Warns if files have changed since last upload.
argument-hint: "[your question about the project]"
allowed-tools: Bash, Read
triggers:
  - "ask agent"
  - "ask the agent"
  - "ask customgpt"
  - "search the knowledge base"
  - "search my files"
  - "search my codebase"
  - "search my project"
  - "search my docs"
  - "what does the agent know"
  - "what do my files say about"
  - "find across my files"
  - "search the agent"
  - "question the agent"
---

# ask-agent

Ask a plain-language question about your project. Returns an AI-generated answer grounded in your files, with source citations.

## Rules

- ALWAYS read `.customgpt-meta.json` to get `agent_id` — never hardcode it
- Each call creates a fresh conversation session (stateless)
- ALWAYS present both the answer AND the sources — sources let the user verify accuracy
- If the agent has no processed documents yet, instruct the user to run `/check-status` first

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id`, `indexed_folder`, and `included_paths`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

---

## Step 3 — Freshness Warning (Optional)

Check whether any files in the included paths were modified after the meta file's timestamp. For each path in `included_paths` (resolved relative to `indexed_folder`):

```bash
find "${indexed_folder}/${path}" -newer "${META_FILE_PATH}" -type f -not -path "*/.git/*" -not -path "*/node_modules/*"
```

If `included_paths` is missing or `["."]`, check `indexed_folder` itself.

If any files are newer, inform the user before proceeding:
> "Note: {N} file(s) have changed since the last upload. Results may not reflect the latest changes. Run `/update-agent` to sync the changed files, or I'll continue with the current data."

Then continue regardless — do not stop.

---

## Step 4 — Get the Question

If the user provided a question with the command, use it directly.

Otherwise ask:
> "What would you like to search for?"

---

## Step 5 — Create a Conversation Session

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/conversations" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data "name=Claude+Code+Query"
```

Read the response and extract `data.session_id` as `$SESSION_ID`.

If the response indicates the agent has no content or is not ready, tell the user:
> "The agent has no processed content yet. Run `/check-status` to see processing progress."

---

## Step 6 — Send the Question

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/conversations/${SESSION_ID}/messages" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "prompt=${QUESTION}" \
  --data "stream=false"
```

Read the response and extract:
- `data.openai_response` — the AI-generated answer
- `data.citations` — array of source references (each has `title`, `url` or `filename`, and optionally `page_id`)

If `openai_response` is empty or null, the agent may still be processing. Tell the user:
> "The agent returned an empty response. Your files may still be processing — run `/check-status` to verify, then try again."

---

## Step 7 — Present Results

Format the output clearly:

---

**Answer**

{openai_response}

**Sources**

| # | Source | Details |
|---|--------|---------|
| 1 | {title or filename} | {url or "uploaded file"} |
| 2 | ... | ... |

---

If citations is empty or null, add:
> "No specific sources cited. The agent may be using general knowledge rather than your files — run `/check-status` to confirm processing is complete."

**After displaying results**, consider whether any cited files are relevant to the current task. If so, use the Read tool to open them for full context before making code changes.

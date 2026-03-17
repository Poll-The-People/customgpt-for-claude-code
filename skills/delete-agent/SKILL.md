---
name: customgpt-ai-rag:delete-agent
description: Permanently delete a CustomGPT.ai agent and all its processed content, then remove the local .customgpt-meta.json binding. Irreversible — requires confirmation.
argument-hint: "[optional: agent ID — defaults to current project's agent]"
allowed-tools: Bash, Read
triggers:
  - "delete agent"
  - "delete the agent"
  - "delete customgpt agent"
  - "remove agent"
  - "remove the agent"
  - "remove customgpt"
  - "destroy agent"
  - "destroy customgpt"
  - "teardown agent"
  - "unlink agent"
---

# delete-agent

Permanently delete a CustomGPT.ai agent, all its processed documents, conversations, and settings. Then remove the local `.customgpt-meta.json` binding.

## Rules

- ALWAYS confirm with the user before deleting — this is completely irreversible
- NEVER remove `.customgpt-meta.json` unless the API deletion succeeds
- If the API call fails, stop and show the error — do NOT remove the meta file

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id`, `agent_name`, and `indexed_folder`. Store the full path as `$META_FILE_PATH`.

If not found:
> "No agent found in this directory tree. Nothing to delete."

---

## Step 3 — Confirm with User

Clearly describe what will be deleted:

> "**Warning: This action is permanent and cannot be undone.**
>
> You are about to delete:
> - Agent: '{agent_name}' (ID: {agent_id})
> - All processed documents and knowledge base content
> - All conversation history
> - Local meta file: `{META_FILE_PATH}`
>
> Type **'delete {agent_name}'** to confirm, or anything else to cancel."

If the user does not type the exact confirmation phrase, stop:
> "Cancelled. Agent '{agent_name}' was not deleted."

---

## Step 4 — Delete the Agent

```bash
curl -s --request DELETE \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Read the response:
- HTTP 200 → deletion successful, proceed to Step 5
- HTTP 404 → agent not found on the server — it may have already been deleted; ask the user if they want to remove the local meta file anyway
- HTTP 401 → invalid API key (see `skills/_shared/api-key.md`); stop
- Any other error → show the full response and stop; do NOT remove the meta file

---

## Step 5 — Remove Meta File

Only execute this after confirmed API success:

```bash
rm "${META_FILE_PATH}"
```

---

## Step 6 — Report

> "Agent '{agent_name}' (ID: {agent_id}) has been permanently deleted."
> "Local meta file `{META_FILE_PATH}` removed."
>
> "Run `/create-agent` to create a new agent for this folder."

---
name: customgpt-ai-rag:create-agent
description: Create a new CustomGPT.ai agent for a local project and upload eligible files into its knowledge base. Saves agent binding to .customgpt-meta.json.
argument-hint: "[folder path — defaults to current directory]"
allowed-tools: Bash, Read
triggers:
  - "create agent"
  - "create a new agent"
  - "create customgpt agent"
  - "new agent"
  - "new customgpt"
  - "setup agent"
  - "set up agent"
  - "initialize agent"
  - "init agent"
  - "connect this repo"
  - "connect this project"
  - "connect everything"
  - "build a rag"
  - "create a rag"
  - "set up rag"
---

# create-agent

Create a CustomGPT.ai agent for a local project and upload eligible files into its knowledge base.

## Rules

- NEVER proceed without an API key
- ALWAYS save `.customgpt-meta.json` before reporting success
- NEVER upload `.git/`, `node_modules/`, or secrets (`.env`, `.env.*`)
- If `.customgpt-meta.json` already exists in the target folder, warn the user before overwriting — they may want `/rebuild-agent` instead

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Determine the Project Root

If the user specified a path, resolve it to an absolute path and verify it exists:

```bash
realpath "$USER_PATH"
```

If no path was given, use the Git repository root if available, otherwise `$PWD`:

```bash
git rev-parse --show-toplevel 2>/dev/null || echo "$PWD"
```

Store the result as `$FOLDER`.

Check whether `.customgpt-meta.json` already exists in `$FOLDER`. If it does, read it and ask:

> "An agent already exists for this folder: '{agent_name}' (ID: {agent_id}). Run `/rebuild-agent` to re-sync it, or type 'overwrite' to create a new agent and replace the meta file."

If they don't confirm overwrite, stop.

---

## Step 3 — Ask What to Add

Ask the user what they want to include:

> "Setting up agent for: `{FOLDER}`
>
> Would you like to add **all files and subfolders** in this project, or only specific folders/files?"

- If "all" / "everything" / default → set `$INCLUDED_PATHS` to `["."]` and use `$FOLDER` as the target for file collection
- If the user specifies paths (e.g., "just src/ and docs/") → set `$INCLUDED_PATHS` to the list they gave (e.g., `["src/", "docs/"]`) and collect files only from those paths relative to `$FOLDER`

---

## Step 4 — Create the Agent

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "project_name=$(basename $FOLDER)" \
  --form "file_data_retension=true"
```

Read the response. On success (HTTP 201), extract `data.id` as `$AGENT_ID` and `data.project_name`.

On failure, show the full response and stop. Common errors:
- HTTP 401 → invalid API key (see `skills/_shared/api-key.md`)
- HTTP 400 → check that `project_name` is not empty

---

## Step 5 — Set Agent Persona

Immediately after creation, configure the agent's persona and enable citations:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/settings" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "enable_citations=1" \
  --form "persona_instructions=You are AI Search Assistant to Claude Code Agent that is working on the project \"${PROJECT_NAME}\".

Claude Code Agent operates on a very big project and it's difficult for it to find files. You are there to help with your semantic search capabilities.

Claude Code Agent will ask you for information about some term or terms, and your job is to:
1. Provide short summary of what you have in your <CONTEXT> about that term(s).
2. Attach EXACT filename(s) from your <CONTEXT>, corresponding to pages you used to build your response. This will help Claude Code Agent locate the actual files.

Your response should contain nothing else."
```

Where `$PROJECT_NAME` is `$(basename $FOLDER)`.

If the call fails, warn the user but continue — persona is not required for the agent to function.

---

## Step 6 — Collect Eligible Files

**Important:** `find` is recursive — it automatically includes all subfolders. Do NOT manually iterate through subdirectories or call `find` per subfolder. One `find` call per included path is all that's needed.

For each path in `$INCLUDED_PATHS`, resolve it relative to `$FOLDER` and run:

```bash
find "$TARGET_PATH" -type f \
  -not -path "*/.git/*" \
  -not -path "*/node_modules/*" \
  -not -path "*/__pycache__/*" \
  -not -path "*/.next/*" \
  -not -path "*/dist/*" \
  -not -path "*/build/*" \
  -not -path "*/.cache/*" \
  -not -path "*/vendor/*" \
  -not -path "*/coverage/*" \
  -not -path "*/.venv/*" \
  -not -path "*/venv/*" \
  -not -path "*/target/*" \
  -not -path "*/.turbo/*" \
  -not -path "*/.parcel-cache/*" \
  -not -name ".env" \
  -not -name ".env.*"
```

CustomGPT.ai supports 1,400+ file types — do NOT filter by extension. If a file type is unsupported, the API will return an error for that file.

Combine results from all included paths. Report the count: "Found {N} eligible files. Uploading..."

---

## Step 7 — Upload Each File

For each file, compute `REL` as its path relative to `$FOLDER` (strip the leading `$FOLDER/` prefix). Upload:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_PATH};filename=${REL}"
```

HTTP 200 or 201 = success. Report progress as you go: ✓ `{REL}` or ✗ `{REL}` (HTTP {status}).

Collect totals: `$UPLOADED` (success) and `$FAILED` (non-200/201).

---

## Step 8 — Save Meta File

```bash
cat > "${FOLDER}/.customgpt-meta.json" << EOF
{
  "agent_id": ${AGENT_ID},
  "agent_name": "$(basename $FOLDER)",
  "indexed_folder": "${FOLDER}",
  "included_paths": ${INCLUDED_PATHS_JSON},
  "created_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF
```

Where `$INCLUDED_PATHS_JSON` is the JSON array, e.g. `["."]` or `["src/", "docs/"]`.

---

## Step 9 — Add Plugin Instructions to CLAUDE.md

**Always do this step.** Append the following block to `${FOLDER}/CLAUDE.md` (create the file if it doesn't exist). Do NOT modify any existing content in CLAUDE.md — only append.

```markdown

<!-- CUSTOMGPT-RAG-PLUGIN:START -->
## CustomGPT.ai RAG Search (plugin: customgpt-ai-rag)

This project has a CustomGPT.ai search agent connected (see `.customgpt-meta.json`). It provides semantic search across the entire codebase via the `customgpt-ai-rag` plugin.

### When to use it

Use `/ask-agent [question]` to search the codebase when:
- You need to find where something is implemented but there are many files to search through
- You're looking for code related to a concept, feature, or term (semantic search beats grep for this)
- The user asks about something and you're not sure which files are relevant
- Traditional search (Grep/Glob) would require many queries or guesswork across a large codebase
- You need to find files by content, not just by name

**Default behavior:** When this agent is available and you need to search across the project, prefer `/ask-agent` over exhaustive Grep/Glob searches. It's faster and finds semantically related code, not just exact matches. You can always follow up with Grep/Read on the specific files it points you to.

### Available commands

- `/ask-agent [question]` — Search the codebase. Returns an answer with source filenames you can then Read.
- `/update-agent` — Sync changed and new files with the agent.
- `/check-status` — Check if the agent is ready and see processing progress.
- `/add-files [path]` — Add specific files or folders.
- `/list-files` — See all documents in the knowledge base.
<!-- CUSTOMGPT-RAG-PLUGIN:END -->
```

---

## Step 10 — Offer Auto-Sync Reminders

Ask the user:
> "Would you like me to automatically remind you to sync changes with the CustomGPT agent after I make significant edits? (yes/no)"

If yes, append the following block to `${FOLDER}/CLAUDE.md` **after** the plugin instructions block:

```markdown
<!-- CUSTOMGPT-RAG-PLUGIN-AUTOSYNC:START -->
### Auto-sync reminders

After completing a task that modifies multiple files (3+ files created, edited, or deleted), ask the user:

> "I've modified several files. Want me to sync the changes with your CustomGPT search agent? (`/update-agent`)"

Guidelines:
- Ask once after a batch of work is done — NOT after every individual edit
- Good triggers: finished implementing a feature, completed a refactor, fixed a bug across multiple files
- If the user declines, do not ask again until the next significant batch of changes
- If the user says "stop asking" or "don't ask about this", stop asking for the rest of this session
<!-- CUSTOMGPT-RAG-PLUGIN-AUTOSYNC:END -->
```

If the user says no, skip this step.

---

## Step 11 — Report

> **Agent created successfully.**
>
> - Agent: `{agent_name}` (ID: `{agent_id}`)
> - Folder: `{folder}`
> - Scope: {included_paths description — "all files" or list of paths}
> - Uploaded: **{UPLOADED}** / {total} files{FAILED > 0 ? ", Failed: {FAILED}" : ""}
>
> CustomGPT.ai is now processing your files. Run `/check-status` in a moment to see progress, then `/ask-agent` to start searching.

If any uploads failed, add:
> "Some files failed to upload. Run `/add-files` to retry specific files, or `/rebuild-agent` to re-sync everything."

---
name: customgpt-ai-rag:rebuild-agent
description: Wipe all documents from the CustomGPT.ai agent and re-upload everything from scratch. Use only when the knowledge base is severely out of sync. Counts against monthly processing limits.
argument-hint: "[optional: folder path — defaults to connected folder from meta file]"
allowed-tools: Bash, Read
triggers:
  - "rebuild agent"
  - "rebuild the agent"
  - "rebuild everything"
  - "full rebuild"
  - "wipe and re-upload"
  - "rebuild knowledge base"
  - "start fresh"
  - "nuke agent"
  - "reset agent"
---

# rebuild-agent

Delete all documents from the agent and re-upload everything from scratch. This is the nuclear option — for incremental updates, use `/update-agent` instead.

## Rules

- ALWAYS read `.customgpt-meta.json` first — respect `included_paths` to know what's in scope
- ALWAYS confirm with the user before proceeding — deletion is irreversible
- ALWAYS warn that re-uploading all files will count against their monthly processing limits
- Paginate the document list — there may be more than 100 documents; use `data.pages.last_page`
- Collect ALL document IDs before starting any deletions
- Touch the meta file after upload to reset the freshness timestamp

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Read Meta File

Walk up from `$PWD` toward `/`, looking for `.customgpt-meta.json`. Extract `agent_id`, `agent_name`, `indexed_folder`, and `included_paths`. Store the full path as `$META_FILE_PATH`.

If `included_paths` is missing, default to `["."]`.

If not found:
> "No agent found in this directory tree. Run `/create-agent` to set one up first."

---

## Step 3 — Check for Changed Files

Before asking the user, give them context on what has changed. For each path in `included_paths` (resolved relative to `indexed_folder`):

```bash
find "${indexed_folder}/${path}" -newer "${META_FILE_PATH}" -type f \
  -not -path "*/.git/*" \
  -not -path "*/node_modules/*" \
  -not -path "*/__pycache__/*"
```

If `included_paths` is `["."]`, check `indexed_folder` itself.

Report the count. This is informational — a rebuild always re-uploads everything regardless.

---

## Step 4 — Confirm with User

> "**Warning:** This will delete ALL documents from agent '{agent_name}' and re-upload everything from scratch.
>
> **All re-uploaded files will count against your monthly processing limits on CustomGPT.ai.**
>
> If you only need to sync changed files, `/update-agent` is faster and cheaper.
>
> Continue with full rebuild? (yes/no)"

If the user does not confirm, stop: "Rebuild cancelled."

---

## Step 5 — Fetch All Document IDs

Fetch all document IDs before deleting anything. Start with page 1:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=1&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Read `data.pages.data[]` and collect all `id` values. Read `data.pages.last_page`. If greater than 1, repeat for each subsequent page:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Collect all IDs across all pages into a list. Tell the user: "Found {total} existing documents to remove."

---

## Step 6 — Delete All Documents

For each collected document ID:

```bash
curl -s --request DELETE \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${PAGE_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

HTTP 200 = deleted. HTTP 404 = already gone (count as success). Report progress periodically: "Removing... {D}/{total} done."

---

## Step 7 — Collect Eligible Files

**Important:** `find` is recursive — it automatically includes all subfolders. Do NOT manually iterate through subdirectories.

For each path in `included_paths`, resolve it relative to `indexed_folder` and run:

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
  -not -name ".env.*" \
  \( \
    -name "*.js" -o -name "*.ts" -o -name "*.tsx" -o -name "*.jsx" \
    -o -name "*.mjs" -o -name "*.cjs" \
    -o -name "*.py" -o -name "*.go" -o -name "*.rb" -o -name "*.java" \
    -o -name "*.cs" -o -name "*.cpp" -o -name "*.c" -o -name "*.h" -o -name "*.hpp" \
    -o -name "*.rs" -o -name "*.swift" -o -name "*.kt" -o -name "*.php" \
    -o -name "*.scala" -o -name "*.lua" -o -name "*.r" -o -name "*.R" \
    -o -name "*.sh" -o -name "*.bash" -o -name "*.zsh" -o -name "*.fish" -o -name "*.ps1" \
    -o -name "*.html" -o -name "*.htm" -o -name "*.css" -o -name "*.scss" \
    -o -name "*.sass" -o -name "*.less" -o -name "*.svelte" -o -name "*.vue" \
    -o -name "*.sql" -o -name "*.graphql" -o -name "*.proto" \
    -o -name "*.json" -o -name "*.jsonc" -o -name "*.yaml" -o -name "*.yml" \
    -o -name "*.toml" -o -name "*.xml" -o -name "*.ini" -o -name "*.cfg" \
    -o -name "*.conf" -o -name "*.env.example" \
    -o -name "*.md" -o -name "*.mdx" -o -name "*.rst" -o -name "*.txt" \
    -o -name "*.csv" -o -name "*.tsv" -o -name "*.pdf" -o -name "*.docx" \
    -o -name "*.doc" -o -name "*.odt" -o -name "*.pptx" -o -name "*.xlsx" \
    -o -name "*.jpg" -o -name "*.jpeg" -o -name "*.png" -o -name "*.webp" \
  \)
```

Combine results from all included paths. Tell the user the count: "Uploading {total} files..."

---

## Step 8 — Upload Each File

For each file, compute `REL` as its path relative to `indexed_folder`:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_PATH};filename=${REL}"
```

HTTP 200 or 201 = success. Report progress: ✓ `{REL}` or ✗ `{REL}` (HTTP {status}).

---

## Step 9 — Reset Freshness Timestamp

```bash
touch "${META_FILE_PATH}"
```

---

## Step 10 — Report

> **Agent '{agent_name}' rebuilt.**
>
> - Removed: {D} documents
> - Uploaded: {UPLOADED} / {total} files{FAILED > 0 ? ", Failed: {FAILED}" : ""}
>
> CustomGPT.ai is now processing your files. Run `/check-status` to monitor progress, then `/ask-agent` to search.

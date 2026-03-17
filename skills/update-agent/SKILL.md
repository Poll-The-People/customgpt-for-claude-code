---
name: customgpt-ai-rag:update-agent
description: Sync only the files that changed since the last upload. Deletes outdated versions and re-uploads them. Much faster and cheaper than a full rebuild.
allowed-tools: Bash, Read
triggers:
  - "update agent"
  - "update the agent"
  - "sync changes"
  - "sync agent"
  - "sync the agent"
  - "re-sync"
  - "resync"
  - "upload changes"
  - "upload changed files"
  - "update knowledge base"
  - "refresh changes"
  - "push changes"
  - "sync my changes"
---

# update-agent

Find files that changed since the last upload, delete their old versions from the agent, and re-upload the new versions. Also detects new files that were never added. Only changed/new files count against your processing limits — unchanged files are left alone.

## Rules

- ALWAYS read `.customgpt-meta.json` first — respect `included_paths` to know what's in scope
- Changed files: newer than the meta file timestamp → delete old version + re-upload
- New files: exist on disk within `included_paths` but not in the agent → ask user before adding
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

## Step 3 — Collect All Eligible Files on Disk

**Important:** `find` is recursive — it automatically includes all subfolders. Do NOT manually iterate through subdirectories.

For each path in `included_paths` (resolved relative to `indexed_folder`):

```bash
find "${indexed_folder}/${path}" -type f \
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

If `included_paths` is `["."]`, check `indexed_folder` itself.

Compute `REL` for each file (path relative to `indexed_folder`). Store as the full local file list.

---

## Step 4 — Fetch Document List from Agent

Fetch all documents currently in the agent:

```bash
curl -s --request GET \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages?page=${PAGE}&limit=100&order=asc" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

Paginate through all pages. Collect all `filename` values into a set of known documents.

---

## Step 5 — Categorize Files

Compare local files against the agent's document list:

- **Changed files**: exist in agent AND are newer than `$META_FILE_PATH` (use `find -newer` or compare timestamps)
- **New files**: exist on disk within `included_paths` but their `REL` path has no match in the agent's document list

For changed files, store the mapping: `REL → page_id`.

---

## Step 6 — Confirm with User

If no changed files AND no new files:
> "Everything is up to date — no changes detected."

Otherwise, present what was found:

> "Found:"
> - **{CHANGED_COUNT} changed file(s)** — will be re-uploaded (old version deleted first)
> {list of changed filenames}

If new files were found, also show:
> - **{NEW_COUNT} new file(s)** not yet in the agent:
> {list of new filenames}
> "Would you like to add these new files as well? (yes/no/skip)"

> "Changed and new files will count against your monthly processing limits. Continue? (yes/no)"

If not confirmed, stop.

---

## Step 7 — Delete Old Versions of Changed Files

For each changed file that has an existing document ID:

```bash
curl -s --request DELETE \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/pages/${PAGE_ID}" \
  --header "Authorization: Bearer ${API_KEY}" \
  --header "accept: application/json"
```

HTTP 200 = deleted. HTTP 404 = already gone (fine, continue).

---

## Step 8 — Upload Changed + New Files

Upload all changed files (new versions) and, if the user confirmed, new files too:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_PATH};filename=${REL}"
```

HTTP 200 or 201 = success. Report each result: ✓ `{REL}` or ✗ `{REL}` (HTTP {status}).

---

## Step 9 — Reset Freshness Timestamp

```bash
touch "${META_FILE_PATH}"
```

---

## Step 10 — Report

> **Agent '{agent_name}' updated.**
>
> - Changed files re-uploaded: {CHANGED_UPLOADED}
> - New files added: {NEW_UPLOADED}
> - Failed: {FAILED}{FAILED > 0 ? " — run `/update-agent` again to retry" : ""}
>
> CustomGPT.ai is now processing the updated files. Run `/check-status` to monitor progress, then `/ask-agent` to search.

---
name: customgpt-ai-rag:add-files
description: Upload specific files, a directory, or the current folder into an existing CustomGPT.ai agent. Auto-creates an agent if none exists. Supports AI Vision for images.
argument-hint: "[file, directory, or leave blank for current folder]"
allowed-tools: Bash, Read
triggers:
  - "add files"
  - "add this file"
  - "add these files"
  - "add this folder"
  - "add this directory"
  - "add this project"
  - "add this repo"
  - "add the files"
  - "add the folder"
  - "add the project"
  - "add the repo"
  - "add to the agent"
  - "add these files to the agent"
  - "add this file to the agent"
  - "upload to the agent"
  - "upload these files"
  - "upload to customgpt"
  - "upload file to agent"
  - "add files to rag"
  - "add to knowledge base"
---

# add-files

Upload specific files, a directory, or the current folder into a CustomGPT.ai agent's knowledge base. If no agent exists, one is created automatically. Existing documents are not affected — this only adds new ones.

## Rules

- THIS SKILL UPLOADS TO THE CUSTOMGPT.AI API via `curl` — it does NOT write to Claude's memory or local files
- NEVER upload `.env` or `.env.*` files
- Auto-create an agent if `.customgpt-meta.json` is not found — do NOT stop or ask the user to run another skill first
- If `.customgpt-meta.json` exists but the specified path is outside `indexed_folder`, still upload — users may intentionally add files from other locations

---

## Step 1 — Resolve API Key

Follow the lookup procedure in `skills/_shared/api-key.md`. Store the result as `$API_KEY`.

---

## Step 2 — Find or Create Agent

Walk up from `$PWD` toward `/`, checking each directory for `.customgpt-meta.json`. If found, extract `agent_id` and `indexed_folder`.

If not found, inform the user and auto-create:

> "No agent found for this project. Creating one now..."

Use the Git root (or `$PWD`) as the project folder:

```bash
PROJECT_FOLDER=$(git rev-parse --show-toplevel 2>/dev/null || echo "$PWD")
```

Create the agent:

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "project_name=$(basename $PROJECT_FOLDER)" \
  --form "file_data_retension=true"
```

Extract `data.id` as `$AGENT_ID`. Save `.customgpt-meta.json` to `$PROJECT_FOLDER`. Set `indexed_folder=$PROJECT_FOLDER`.

---

## Step 3 — Resolve What to Add

**If the user specified a single file:**

If it's just a filename (no path separators), search for it under `indexed_folder`:

```bash
find "$indexed_folder" -name "${FILENAME}" -type f
```

Use the first match. If not found, tell the user and stop.

If it's a full path, resolve to absolute and verify it exists.

**If the user specified a directory:** Collect eligible files recursively (Step 4 `find` command).

**If nothing specified:** Use `$PWD` as the directory.

CustomGPT.ai supports 1,400+ file types — do NOT filter by extension. If a file type is unsupported, the API will return an error for that file.

---

## Step 4 — Collect Files (Directory Mode)

```bash
find "$TARGET_DIR" -type f \
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

Tell the user the file count before uploading.

---

## Step 5 — Vision / OCR Check

Default: every file uploads with `is_ocr_enabled=0` (plain text extraction). Only ask about AI Vision (`is_ocr_enabled=2`) for file types where it actually matters: **images** and **PDFs**.

**If any files are images** (`.jpg`, `.jpeg`, `.png`, `.webp`), ask:

> "Image files detected ({N} images). Enable AI Vision processing for richer content extraction? (yes/no)"

If yes, also ask:
> "Compress images before vision processing? Reduces token usage. (yes/no)"

Store as `$USE_VISION_IMAGES` and `$COMPRESS_IMAGES`.

**If any files are PDFs** (`.pdf`), ask:

> "PDF files detected ({N} PDFs). Do any contain scanned pages, handwriting, or heavy imagery that text extraction would miss? Enable AI Vision for PDFs? (yes/no)"

Store as `$USE_VISION_PDFS`. If the user is unsure, default to **no** — AI Vision is slower and costs more tokens, so only opt in when the PDFs clearly need it.

The vision-enabled batch in Step 6 contains images when `$USE_VISION_IMAGES=yes` and PDFs when `$USE_VISION_PDFS=yes`. Everything else (including images/PDFs where the user said no) goes in the default batch with `is_ocr_enabled=0`.

---

## Step 6 — Upload Files (Batched via `files[]`)

The API accepts multiple files per request via the `files[]` multipart parameter. **Always batch** — do NOT issue one request per file. Limits per request:

- ≤ 50 files
- ≤ 100MB per file (skip larger files and report them)
- ≤ 1GB total batch size

For each eligible file, compute `REL` as its path relative to `indexed_folder`. If the file is outside `indexed_folder`, use its path relative to the file's own directory. Pass `REL` as the multipart `filename` so the server preserves directory structure.

Split files into two groups (vision params apply to the whole request, so they cannot be mixed):

1. **Default batch** — all files where vision was not opted in. Always sent with `is_ocr_enabled=0`.
2. **Vision batch** — images when `$USE_VISION_IMAGES=yes` **and** PDFs when `$USE_VISION_PDFS=yes`. Sent with `is_vision_enabled=true` and `is_ocr_enabled=2`.

Within each group, chunk into batches that fit the limits above, then send one `curl` per batch.

**Default batch (`is_ocr_enabled=0`):**

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "files[]=@${ABSOLUTE_PATH_1};filename=${REL_1}" \
  --form "files[]=@${ABSOLUTE_PATH_2};filename=${REL_2}" \
  --form "is_ocr_enabled=0"
  # ...up to 50 files[]= parts
```

**Vision batch (`is_ocr_enabled=2`) — images and/or PDFs the user opted in:**

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "files[]=@${VISION_PATH_1};filename=${VISION_REL_1}" \
  --form "files[]=@${VISION_PATH_2};filename=${VISION_REL_2}" \
  --form "is_vision_enabled=true" \
  --form "is_ocr_enabled=2" \
  --form "is_vision_compress_image=${COMPRESS_IMAGES}"
```

`is_vision_compress_image` is only meaningful when the batch contains images — include it whenever `$USE_VISION_IMAGES=yes`, otherwise omit it.

HTTP 200 or 201 = batch accepted. The response body lists per-file results; report each as ✓ `{REL}` or ✗ `{REL}` ({reason}). If a whole batch fails, report the HTTP status and retry that batch once.

---

## Step 7 — Update Freshness Timestamp

```bash
touch "${META_FILE_PATH}"
```

---

## Step 8 — Report

> **Added {UPLOADED} file(s) to agent `{agent_name}`.**{FAILED > 0 ? "\n> Failed: {FAILED} — run `/add-files` on specific files to retry." : ""}
>
> CustomGPT.ai is now processing your files. Run `/check-status` to monitor progress, then `/ask-agent` to start searching.

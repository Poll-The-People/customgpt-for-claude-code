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
- NEVER upload `.env`, `.env.*`, or binary files
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

If it's a full path, resolve to absolute and verify it exists. Check the extension against the whitelist below. If unsupported, tell the user: "Extension `.{ext}` is not supported. Supported types: [list]."

**If the user specified a directory:** Collect eligible files recursively (Step 4 `find` command).

**If nothing specified:** Use `$PWD` as the directory.

### Supported Extensions

**Code:** `.js` `.ts` `.tsx` `.jsx` `.mjs` `.cjs` `.py` `.go` `.rb` `.java` `.cs` `.cpp` `.c` `.h` `.hpp` `.rs` `.swift` `.kt` `.php` `.scala` `.lua` `.r` `.R` `.sh` `.bash` `.zsh` `.fish` `.ps1` `.html` `.htm` `.css` `.scss` `.sass` `.less` `.svelte` `.vue` `.sql` `.graphql` `.proto`

**Config / Data:** `.json` `.jsonc` `.yaml` `.yml` `.toml` `.xml` `.ini` `.cfg` `.conf` `.env.example`

**Docs:** `.md` `.mdx` `.rst` `.txt` `.csv` `.tsv` `.pdf` `.docx` `.doc` `.odt` `.pptx` `.xlsx`

**Images:** `.jpg` `.jpeg` `.png` `.webp`

### Excluded Directories

`.git` `node_modules` `__pycache__` `.next` `dist` `build` `.cache` `vendor` `coverage` `.venv` `venv` `target` `.turbo` `.parcel-cache`

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

Tell the user the file count before uploading.

---

## Step 5 — Vision Check (Images Only)

If any files to upload are images (`.jpg`, `.jpeg`, `.png`, `.webp`), ask before uploading:

> "Image files detected ({N} images). Enable AI Vision processing for richer content extraction? (yes/no)"

If yes, also ask:
> "Compress images before vision processing? Reduces token usage. (yes/no)"

Store these choices as `$USE_VISION` and `$COMPRESS_IMAGES` for use in Step 6.

---

## Step 6 — Upload Each File

For each eligible file, compute `REL` as its path relative to `indexed_folder`. If the file is outside `indexed_folder`, use its path relative to the file's own directory.

**Non-image files:**

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_PATH};filename=${REL}"
```

**Image files with AI Vision enabled:**

```bash
curl -s --request POST \
  --url "https://app.customgpt.ai/api/v1/projects/${AGENT_ID}/sources" \
  --header "Authorization: Bearer ${API_KEY}" \
  --form "file=@${ABSOLUTE_PATH};filename=${REL}" \
  --form "is_vision_enabled=true" \
  --form "ocr_mode=2" \
  --form "is_vision_compress_image=${COMPRESS_IMAGES}"
```

**Image files without AI Vision (or if user said no):** Use the non-image curl above (omit vision fields).

HTTP 200 or 201 = success. Report each result: ✓ `{REL}` or ✗ `{REL}` (HTTP {status}).

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

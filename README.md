# customgpt-ai-rag

A Claude Code plugin that gives you semantic search over your entire project using [CustomGPT.ai](https://app.customgpt.ai). Upload your codebase once, then ask questions across all files instantly.

**No dependencies. Just `curl`.**

## What is CustomGPT.ai?

[CustomGPT.ai](https://app.customgpt.ai) is a RAG (Retrieval-Augmented Generation) platform. It takes your files, chunks and embeds them, and lets you query across everything with AI-generated answers and source citations. This plugin connects it to Claude Code so you can search large projects without hitting context limits.

## Install

In Claude Code, run:

```
/plugin marketplace add adorosario/customgpt-skill-plugin
/plugin install customgpt-ai-rag@adorosario-customgpt-ai-rag
/reload-plugins
```

## Setup

1. Get a [CustomGPT.ai API key](https://app.customgpt.ai/profile#api-keys) (sign up at [app.customgpt.ai](https://app.customgpt.ai) if you don't have an account)
2. Run `/create-agent` — the plugin will ask for your key and save it

That's it. The plugin uploads your files and you're ready to search.

## Usage

```
/create-agent          → connect your project to a new agent
/ask-agent [question]  → search across your codebase
```

### Keeping files in sync

```
/update-agent          → sync changed + new files (fast, cheap)
/rebuild-agent         → wipe and re-upload everything (uses quota)
```

### Managing your agent

```
/check-status          → see if files are done processing
/add-files [path]      → add specific files or folders
/list-files            → see all documents in the knowledge base
/delete-agent          → permanently delete the agent
```

## API Key

The plugin checks these locations in order:

1. `$CUSTOMGPT_API_KEY` env var
2. `.env` file (walks up from current dir)
3. `~/.claude/customgpt-config.json`

If no key is found, the plugin prompts you and saves it automatically.

## Supported Files

**Code:** `.js` `.ts` `.py` `.go` `.rb` `.java` `.cs` `.cpp` `.rs` `.swift` `.kt` `.php` `.sh` `.html` `.css` `.vue` `.svelte` `.sql` and more

**Config:** `.json` `.yaml` `.toml` `.xml` `.ini` `.env.example`

**Docs:** `.md` `.txt` `.pdf` `.docx` `.csv` `.xlsx`

**Images:** `.jpg` `.png` `.webp` (with optional AI Vision)

**Auto-excluded:** `.git/` `node_modules/` `dist/` `build/` `.env` and other common junk directories.

## Privacy & Billing

Files are uploaded to CustomGPT.ai for processing. Uses your existing subscription quota. API key stored locally at `~/.claude/customgpt-config.json`.

## License

MIT

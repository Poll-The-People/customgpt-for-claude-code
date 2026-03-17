---
name: customgpt-ai-rag:help
description: Show available CustomGPT.ai RAG skills and how to get started with semantic search over your project.
allowed-tools: Read
triggers:
  - "customgpt help"
  - "rag help"
  - "how do I use customgpt"
  - "what customgpt skills are available"
  - "what can the customgpt plugin do"
  - "get started with rag"
  - "setup customgpt"
  - "customgpt getting started"
  - "help with customgpt"
  - "how does rag search work"
---

# CustomGPT.ai RAG Search

Search across your entire project instantly — thousands of files, no context overflow.

## Quick Start

```
1. /create-agent    → connect your project to a new CustomGPT.ai agent
2. /check-status    → wait until files are processed
3. /ask-agent       → ask questions across your codebase
```

## Prerequisites

| Requirement | Details |
|---|---|
| CustomGPT.ai account | https://app.customgpt.ai |
| API key | https://app.customgpt.ai/profile#api-keys |
| `curl` | Pre-installed on macOS, Linux, Windows 10+ |

## Skills

### Setup

| Skill | What it does |
|---|---|
| `/create-agent` | Create a new agent and upload your project files |
| `/add-files [path]` | Add specific files or a directory to an existing agent |

### Search

| Skill | What it does |
|---|---|
| `/ask-agent [question]` | Ask a question — get an answer with source citations |

### Sync

| Skill | What it does |
|---|---|
| `/update-agent` | Sync changed files + add new files |
| `/rebuild-agent` | Wipe everything and re-upload from scratch |

### Monitor

| Skill | What it does |
|---|---|
| `/check-status` | Processing progress and agent readiness |
| `/list-files` | All documents in the knowledge base with their status |
| `/delete-agent` | Permanently delete the agent |

## What should I use?

**Files changed** → `/update-agent`

**Want to start from scratch** → `/rebuild-agent`

**Just want to search** → `/ask-agent` (warns you if files changed)

**See what's in the knowledge base** → `/list-files`

**Start over with a different folder** → `/delete-agent` then `/create-agent`

## How it works

Your project maps to one CustomGPT.ai agent. The connection is stored in `.customgpt-meta.json` at the project root — all skills read this automatically.

After uploading, CustomGPT.ai processes files asynchronously. Use `/check-status` to monitor — queries work best once all files are processed.

## Privacy & billing

Files are uploaded to CustomGPT.ai for processing. The plugin uses your existing subscription quota. Your API key is stored locally in `~/.claude/customgpt-config.json` and only sent to the CustomGPT.ai API.

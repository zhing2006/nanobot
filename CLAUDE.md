# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

nanobot is an ultra-lightweight personal AI assistant framework (~3,400 lines core agent code). It connects to multiple chat platforms (Telegram, Discord, WhatsApp, Feishu) and multiple LLM providers via litellm. Built on Python 3.11+ with asyncio.

## Build & Development Commands

```bash
# Install for development
pip install -e ".[dev]"
# Or with uv (faster)
uv pip install -e ".[dev]"

# Run tests
pytest
pytest tests/test_tool_validation.py          # single test file
pytest tests/test_tool_validation.py::test_name  # single test

# Lint
ruff check nanobot/
ruff check --fix nanobot/                     # auto-fix

# Run the agent
nanobot agent -m "message"                    # one-shot
nanobot agent                                 # interactive
nanobot gateway                               # start multi-channel gateway

# Docker
docker build -t nanobot .
docker run -v ~/.nanobot:/root/.nanobot nanobot gateway
```

## Architecture

### Message Flow

```
Channel (Telegram/Discord/...) → InboundMessage → MessageBus (async queue)
  → AgentLoop → ContextBuilder → LLM Provider (via litellm)
  → Tool execution loop (max 20 iterations) → OutboundMessage → Channel
```

### Core Modules (`nanobot/`)

- **agent/loop.py** — Core processing engine. Receives messages from the bus, builds context, calls LLM, executes tools in a loop, sends responses back.
- **agent/context.py** — Builds the system prompt from workspace bootstrap files (AGENTS.md, SOUL.md, USER.md, TOOLS.md), memory, active/available skills, and conversation history.
- **agent/tools/** — Tool system. Each tool extends `Tool` (base.py), registered in `ToolRegistry`. Tools: `read_file`, `write_file`, `edit_file`, `list_dir`, `exec`, `web_search`, `web_fetch`, `message`, `spawn`, cron management.
- **agent/subagent.py** — Spawns background sub-agents for long-running tasks.
- **agent/memory.py** — Persistent memory management (workspace `memory/` directory).
- **agent/skills.py** — Skill loading from `nanobot/skills/` (each skill has a `SKILL.md`).
- **bus/** — Async message routing. `InboundMessage`/`OutboundMessage` dataclasses in events.py, `MessageBus` with async queues and subscriber pattern in queue.py.
- **channels/** — Chat platform integrations. All extend `BaseChannel` (base.py). Manager (manager.py) starts/stops channels. Each channel converts platform events to/from bus messages.
- **providers/** — LLM abstraction. `LLMProvider` base in base.py, `LiteLLMProvider` in litellm_provider.py wraps litellm for unified multi-provider access. Transcription support via Groq Whisper.
- **config/** — Pydantic-based config schema (schema.py) loaded from `~/.nanobot/config.json` by loader.py.
- **session/manager.py** — Conversation history persistence per `{channel}:{chat_id}` session key.
- **cron/** — Scheduled task service using croniter.
- **heartbeat/** — Periodic (30-min) proactive task execution from workspace `HEARTBEAT.md`.
- **cli/commands.py** — Typer CLI. Entry point: `nanobot.cli.commands:app`.

### WhatsApp Bridge (`bridge/`)

Node.js/TypeScript bridge using Baileys library. Communicates with the Python process via WebSocket. Built separately with `npm run build`.

### Workspace (`workspace/`)

Default templates copied to `~/.nanobot/workspace/` on `nanobot onboard`. Bootstrap files loaded into agent context:
- `AGENTS.md` — Agent behavior instructions
- `SOUL.md` — Personality definition
- `USER.md` — User context
- `TOOLS.md` — Tool documentation
- `HEARTBEAT.md` — Periodic task list

### Adding a New Tool

1. Create a class extending `Tool` in `nanobot/agent/tools/`
2. Implement `name`, `description`, `parameters` (JSON Schema), and `execute`
3. Register it in `AgentLoop._register_default_tools()` in loop.py

### Adding a New Channel

Extend `BaseChannel` from `nanobot/channels/base.py`, implement `start()` and `stop()`, handle conversion between platform messages and `InboundMessage`/`OutboundMessage`.

## Code Style

- Ruff linter with rules: E, F, I, N, W (E501 ignored)
- Line length: 100 characters
- Target: Python 3.11
- Async-first: all tool execution and channel handlers are async
- Config uses camelCase keys (JSON), Pydantic models use snake_case

## Test Configuration

- pytest with `asyncio_mode = "auto"` (no need for `@pytest.mark.asyncio`)
- Test directory: `tests/`

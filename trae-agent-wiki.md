# Trae Agent — Code Wiki

> **Source repository:** [commercegtm/trae-agent](https://github.com/commercegtm/trae-agent)  
> **License:** MIT — Copyright © 2025 ByteDance Ltd.  
> **Python:** ≥ 3.12  &nbsp;|&nbsp; **Package manager:** [uv](https://github.com/astral-sh/uv)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Directory Structure](#3-directory-structure)
4. [Module Responsibilities](#4-module-responsibilities)
   - 4.1 [trae_agent/cli.py — Entry Point](#41-trae_agentclipy--entry-point)
   - 4.2 [trae_agent/agent/ — Agent Core](#42-trae_agentagent--agent-core)
   - 4.3 [trae_agent/tools/ — Tool Ecosystem](#43-trae_agenttools--tool-ecosystem)
   - 4.4 [trae_agent/utils/ — Utilities](#44-trae_agentutils--utilities)
   - 4.5 [trae_agent/prompt/ — System Prompts](#45-trae_agentprompt--system-prompts)
   - 4.6 [server/ — HTTP Server (WIP)](#46-server--http-server-wip)
   - 4.7 [evaluation/ — Benchmarking](#47-evaluation--benchmarking)
5. [Key Classes and Functions](#5-key-classes-and-functions)
   - 5.1 [Agent Layer](#51-agent-layer)
   - 5.2 [Tool Layer](#52-tool-layer)
   - 5.3 [LLM Client Layer](#53-llm-client-layer)
   - 5.4 [Configuration](#54-configuration)
   - 5.5 [Observability](#55-observability)
   - 5.6 [CLI Console](#56-cli-console)
6. [Dependency Relationships](#6-dependency-relationships)
7. [Configuration Reference](#7-configuration-reference)
8. [Running the Project](#8-running-the-project)
9. [Development Guide](#9-development-guide)

---

## 1. Project Overview

**Trae Agent** is an LLM-powered command-line tool designed to autonomously solve software engineering tasks — particularly resolving GitHub issues. Given a natural-language task description and a code repository, the agent:

1. Explores the codebase to understand context.
2. Reproduces bugs and identifies root causes.
3. Implements and verifies code fixes.
4. Summarises actions via the **Lakeview** subsystem.

Key design goals:
- **Multi-provider LLM support** — OpenAI, Anthropic, Google Gemini, Azure, OpenRouter, Doubao, Ollama.
- **Transparent, modular architecture** — suitable for research and extension.
- **Docker isolation** — tasks can run inside containers with workspace mounting.
- **MCP (Model Context Protocol)** — dynamically discovers and calls external tool servers.
- **Full trajectory recording** — every LLM call and tool invocation is serialised to JSON for analysis and replay.

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          CLI  (trae-cli)                        │
│                      trae_agent/cli.py                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │ creates
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Agent Facade                             │
│                    trae_agent/agent/agent.py                    │
│   (selects AgentType, wires console + trajectory recorder)      │
└──────────────────────────┬──────────────────────────────────────┘
                           │ instantiates
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                        TraeAgent                                │
│                trae_agent/agent/trae_agent.py                   │
│        extends BaseAgent  (agent/base_agent.py)                 │
│   • registers tools          • MCP client management           │
│   • executes task loop       • git diff / patch handling        │
└───────┬──────────────────────────────────────────┬─────────────┘
        │ uses                                      │ uses
        ▼                                           ▼
┌──────────────────┐                   ┌────────────────────────┐
│  ToolExecutor /  │                   │      LLMClient         │
│  DockerToolExec  │                   │ (utils/llm_clients/)   │
│  (tools/)        │                   │  provider façade       │
└──────────────────┘                   └────────────────────────┘
        │                                           │
        │ wraps                                     │ calls
        ▼                                           ▼
┌──────────────────────────────┐   ┌──────────────────────────────┐
│  Individual Tools            │   │  Provider-specific clients   │
│  BashTool                    │   │  AnthropicClient             │
│  TextEditorTool              │   │  OpenAIClient                │
│  SequentialThinkingTool      │   │  GoogleClient  ...           │
│  CKGTool                     │   └──────────────────────────────┘
│  MCPTool  TaskDoneTool  ...  │
└──────────────────────────────┘

Shared cross-cutting concerns
────────────────────────────────────────────────────────────────
  TrajectoryRecorder  ←→  every LLM call & tool result
  LakeView            ←→  step summarisation via a secondary LLM
  CLIConsole          ←→  terminal rendering (Simple or Rich TUI)
  Config              ←→  YAML/JSON configuration tree
```

---

## 3. Directory Structure

```
trae-agent/
├── trae_agent/              # Main Python package
│   ├── __init__.py
│   ├── cli.py               # Click-based CLI entry point
│   ├── agent/
│   │   ├── agent.py         # Agent façade / factory
│   │   ├── agent_basics.py  # State enums & dataclasses
│   │   ├── base_agent.py    # Abstract execution loop
│   │   ├── docker_manager.py
│   │   └── trae_agent.py    # Concrete software-engineering agent
│   ├── tools/
│   │   ├── base.py          # Tool / ToolExecutor abstractions
│   │   ├── bash_tool.py
│   │   ├── edit_tool.py
│   │   ├── edit_tool_cli.py
│   │   ├── json_edit_tool.py
│   │   ├── json_edit_tool_cli.py
│   │   ├── sequential_thinking_tool.py
│   │   ├── task_done_tool.py
│   │   ├── ckg_tool.py
│   │   ├── mcp_tool.py
│   │   ├── docker_tool_executor.py
│   │   ├── run.py
│   │   └── ckg/             # Code Knowledge Graph SQLite DB helpers
│   ├── prompt/
│   │   └── agent_prompt.py  # System prompt for TraeAgent
│   └── utils/
│       ├── config.py        # Config dataclasses + YAML loader
│       ├── constants.py     # LOCAL_STORAGE_PATH
│       ├── lake_view.py     # Lakeview step summariser
│       ├── legacy_config.py # JSON→YAML migration
│       ├── mcp_client.py    # MCP server connection
│       ├── trajectory_recorder.py
│       ├── cli/             # Terminal console implementations
│       │   ├── cli_console.py
│       │   ├── console_factory.py
│       │   ├── rich_console.py
│       │   └── simple_console.py
│       └── llm_clients/
│           ├── llm_basics.py          # LLMMessage / LLMResponse / LLMUsage
│           ├── base_client.py         # BaseLLMClient ABC
│           ├── llm_client.py          # LLMClient façade + LLMProvider enum
│           ├── anthropic_client.py
│           ├── openai_client.py
│           ├── openai_compatible_base.py
│           ├── azure_client.py
│           ├── google_client.py
│           ├── ollama_client.py
│           ├── openrouter_client.py
│           ├── doubao_client.py
│           └── retry_utils.py
├── evaluation/              # SWE-bench evaluation harness
├── server/                  # FastAPI HTTP server (WIP)
├── tests/                   # pytest test suite
├── pyproject.toml
├── Makefile
├── trae_config.yaml.example
└── trae_config.json.example
```

---

## 4. Module Responsibilities

### 4.1 `trae_agent/cli.py` — Entry Point

Implements the `trae-cli` console command using **Click**. Exposes four sub-commands:

| Command | Purpose |
|---------|---------|
| `run` | Execute a single task (string or file). Supports LLM provider/model selection, Docker environment, working directory, trajectory output. |
| `interactive` | Launch a REPL session where the user submits tasks one after another. Console type (simple/rich) is selectable. |
| `show_config` | Pretty-print the resolved configuration (API keys partially masked). |
| `tools` | List all available tools and their descriptions. |

Responsibilities:
- Resolves and loads the config file (YAML preferred, legacy JSON supported).
- Validates the Docker daemon when Docker mode is requested.
- Builds PyInstaller binary tools when running from a bundled executable.
- Constructs an `Agent` instance and calls `agent.run()`.

---

### 4.2 `trae_agent/agent/` — Agent Core

#### `agent_basics.py` — State & Data Models

Pure data layer; no business logic.

| Symbol | Kind | Purpose |
|--------|------|---------|
| `AgentStepState` | Enum | Per-step states: `thinking`, `calling_tool`, `reflecting`, `completed`, `error` |
| `AgentState` | Enum | Lifecycle states: `idle`, `running`, `completed`, `error` |
| `AgentStep` | Dataclass | One iteration of the agent loop — thought, tool calls/results, reflection, token usage |
| `AgentExecution` | Dataclass | Full task run — task text, all steps, final output, cumulative tokens, elapsed time |
| `AgentError` | Exception | Base exception for agent failures |

#### `base_agent.py` — Abstract Execution Engine

`BaseAgent` owns the core **reason → act → observe** loop. Subclasses implement domain-specific behaviour.

Key responsibilities:
- Maintains the message history (`_messages`) sent to the LLM.
- Calls `_run_llm_step()` to obtain LLM responses; interprets `finish_reason` to decide between task completion, retry, or tool dispatch.
- Calls `_tool_call_handler()` which routes to sequential or parallel tool execution and optionally generates reflections on failures.
- Enforces `max_steps` limit.
- Ensures Docker container cleanup and MCP client teardown via `try/finally`.

Abstract methods subclasses must implement:
- `new_task(task, extra_args, tool_names)` — set up a new task context.
- `cleanup_mcp_clients()` — release MCP server connections.

#### `trae_agent.py` — Concrete Software-Engineering Agent

`TraeAgent(BaseAgent)` is the only shipping agent type. Additional responsibilities beyond `BaseAgent`:

- Registers the default tool set: `BashTool`, `TextEditorTool`, `JsonEditTool`, `SequentialThinkingTool`, `TaskDoneTool`.
- Initialises MCP clients asynchronously and exposes their tools as `MCPTool` wrappers.
- Implements `get_git_diff(base_commit)` to retrieve the patch produced by the agent.
- Implements `remove_patches_to_tests()` to strip test-file modifications from patches (required by SWE-bench rules).
- Overrides `llm_indicates_task_completed()` to validate that the LLM called `task_done` and that a non-empty code patch exists.

#### `agent.py` — Agent Factory / Façade

`Agent` is the public entry point used by `cli.py`.

- Reads `AgentType` (currently only `TraeAgent`).
- Creates the appropriate agent instance, wiring in a `TrajectoryRecorder` and a `CLIConsole`.
- `run(task, extra_args, tool_names)` — async method that calls `new_task()`, initialises MCP servers, then calls `execute_task()`.

#### `docker_manager.py` — Container Lifecycle

`DockerManager` wraps the Docker SDK and `pexpect` to provide a persistent shell inside a container.

| Method | Purpose |
|--------|---------|
| `start()` | Build/pull image → create container → mount workspace → copy agent tools → open persistent bash shell |
| `execute(cmd, timeout)` | Run a shell command inside the container; return stdout/stderr/exit-code |
| `stop()` | Close the persistent shell; remove managed containers |
| `_translate_path(path)` | Convert host paths to container-equivalent paths |

---

### 4.3 `trae_agent/tools/` — Tool Ecosystem

#### `base.py` — Abstractions

| Symbol | Kind | Purpose |
|--------|------|---------|
| `ToolError` | Exception | Raised by tool implementations |
| `ToolExecResult` | Dataclass | Raw output, error text, exit code |
| `ToolResult` | Dataclass | Final result linked to a `call_id` |
| `ToolCall` | Dataclass | Parsed invocation: name, id, arguments |
| `ToolParameter` | Dataclass | Parameter schema: name, type, description, required |
| `Tool` | Abstract class | All tools inherit this; must implement `get_name()`, `get_description()`, `get_parameters()`, `execute()` |
| `ToolExecutor` | Class | Registry + runner; supports `parallel_tool_call()` and `sequential_tool_call()` |

`Tool.get_tool_call_schema(provider)` generates JSON Schema compatible with OpenAI (strict mode: all params required, `additionalProperties: false`) and Gemini formats.

#### Individual Tools

| File | Class | Description |
|------|-------|-------------|
| `bash_tool.py` | `BashTool` | Persistent `asyncio` subprocess shell with 120 s timeout; supports session restart; cross-platform (Unix/Windows) |
| `edit_tool.py` | `TextEditorTool` | File operations: `view`, `create`, `str_replace`, `insert`; enforces absolute paths; requires unique string matches before replacement |
| `json_edit_tool.py` | `JsonEditTool` | JSON-specific editing using structured patch operations |
| `sequential_thinking_tool.py` | `SequentialThinkingTool` | Structured multi-step reasoning with branching and revision; maintains `thought_history` and `branches`; visual output using box-drawing characters |
| `task_done_tool.py` | `TaskDoneTool` | Signals task completion to the agent loop; must be called after verification |
| `ckg_tool.py` | `CKGTool` | Queries a SQLite Code Knowledge Graph (`search_function`, `search_class`, `search_class_method`); caches DB connections per codebase path |
| `mcp_tool.py` | `MCPTool` | Wraps an MCP server tool; converts MCP input schema to `ToolParameter` list; delegates `execute()` to `MCPClient.call_tool()` |
| `docker_tool_executor.py` | `DockerToolExecutor` | Routes tool calls: bash/edit/json tools execute inside Docker via `DockerManager`; all others execute locally |

---

### 4.4 `trae_agent/utils/` — Utilities

#### `config.py` — Configuration Tree

| Class | Purpose |
|-------|---------|
| `ModelProvider` | API credentials + base URL + API version |
| `ModelConfig` | Temperature, max tokens, top-p, parallel tools; `get_max_tokens_param()` handles `max_completion_tokens` vs `max_tokens` |
| `MCPServerConfig` | Transport type (stdio / SSE / HTTP / WebSocket), command, args, env |
| `AgentConfig` | Tool list, step limit, parallel execution flag |
| `TraeAgentConfig` | Extends `AgentConfig`; MCP server allowlist |
| `LakeviewConfig` | Model reference for the Lakeview summariser |
| `Config` | Root object; `Config.create(path)` parses YAML/JSON, validates all cross-references, raises `ConfigError` on inconsistencies |

Environment variable overrides are supported (e.g. `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`).

#### `lake_view.py` — Step Summariser

`LakeView` calls a secondary, typically smaller LLM to annotate each agent step with:
- A **task description** (≤10 words) via `EXTRACTOR_PROMPT`.
- An **action tag** from a fixed taxonomy via `TAGGER_PROMPT`.

Tags and their emoji:

| Tag | Emoji | Meaning |
|-----|-------|---------|
| `WRITE_TEST` | 🧪 | Writing tests |
| `VERIFY_TEST` | ✅ | Running tests |
| `EXAMINE_CODE` | 🔍 | Reading/exploring code |
| `WRITE_FIX` | 🔧 | Implementing the fix |
| `VERIFY_FIX` | 🏃 | Running the fixed code |
| `REPORT` | 📝 | Summarising findings |
| `THINK` | 💭 | Reasoning / planning |
| `OUTLIER` | ❓ | None of the above |

#### `trajectory_recorder.py` — Execution Log

`TrajectoryRecorder` serialises the complete agent run to a timestamped JSON file in `~/.trae-agent/trajectories/`.

Captured data:
- Task description, timestamps, provider + model.
- Each LLM message exchange (roles, content, tool calls, token usage including cache metrics).
- Each agent step (state, tool calls, tool results, reflection, errors).
- Lakeview summaries per step.

#### `mcp_client.py` — MCP Server Connection

`MCPClient` connects to an MCP server and exposes its tools.

| Method | Purpose |
|--------|---------|
| `connect_and_discover()` | Connect via stdio (WebSocket/HTTP not yet implemented), list all available tools |
| `call_tool(name, args)` | Invoke a remote tool; return raw MCP result |
| `cleanup()` | Release the `AsyncExitStack` and mark server disconnected |

Status tracked via `MCPServerStatus` (disconnected → connecting → connected) and `MCPDiscoveryState`.

#### `constants.py`

Defines `LOCAL_STORAGE_PATH = Path.home() / ".trae-agent"` — the single source of truth for local storage.

#### `llm_clients/llm_basics.py` — Shared LLM Data Models

| Class | Purpose |
|-------|---------|
| `LLMMessage` | Unified message: role, content, optional `tool_call` / `tool_result` |
| `LLMUsage` | Token counters: input, output, cache-creation, cache-read, reasoning; supports `+` aggregation |
| `LLMResponse` | LLM output: content, usage, model id, finish_reason, tool_calls |

#### `llm_clients/` — Provider Clients

All provider clients extend `BaseLLMClient`:

| Provider | Class | Notes |
|----------|-------|-------|
| Anthropic | `AnthropicClient` | Handles `str_replace_based_edit_tool` natively; cache-aware token tracking; retry decorator |
| OpenAI | `OpenAIClient` | Strict tool schema mode |
| Azure | `AzureClient` | Extends `OpenAICompatibleBase`; `max_completion_tokens` for newer models |
| Google Gemini | `GoogleClient` | Gemini-specific tool schema format |
| Ollama | `OllamaClient` | Local inference; extends `OpenAICompatibleBase` |
| OpenRouter | `OpenRouterClient` | Extends `OpenAICompatibleBase` |
| Doubao | `DoubaoClient` | ByteDance Doubao API; extends `OpenAICompatibleBase` |

`LLMClient` is the façade: it reads `LLMProvider` from config and lazily imports + instantiates the correct client.

#### `cli/` — Console Implementations

| File | Class | Purpose |
|------|-------|---------|
| `cli_console.py` | `CLIConsole` (ABC) | Defines the console contract; holds step history and optional LakeView reference |
| `simple_console.py` | `SimpleConsole` | Plain text output |
| `rich_console.py` | `RichConsole` | Full Textual TUI with live panels, step tables, emoji state indicators |
| `console_factory.py` | `ConsoleFactory` | Creates the right console based on `ConsoleType` enum |

State-to-emoji mapping used in step tables:

| State | Indicator |
|-------|-----------|
| `thinking` | 🤔 yellow |
| `calling_tool` | 🔧 blue |
| `reflecting` | 💭 purple |
| `completed` | ✅ green |
| `error` | ❌ red |

---

### 4.5 `trae_agent/prompt/` — System Prompts

`agent_prompt.py` contains the system prompt injected into every agent conversation. It instructs the agent to:

1. Understand the issue statement.
2. Explore and locate relevant files.
3. Reproduce the bug with test scripts.
4. Debug to identify root causes.
5. Implement a targeted fix.
6. Verify through testing and edge-case analysis.
7. Summarise the completed work.

Key constraints embedded in the prompt:
- All tool paths **must be absolute**.
- Think before acting — use `SequentialThinkingTool` with 5–25 thoughts.
- Act like a senior software engineer; prioritise correctness and safety.

---

### 4.6 `server/` — HTTP Server (WIP)

A planned **FastAPI** HTTP server that exposes agent functionality over HTTP. Design requirements:
- **Stateless** — no shared state between requests.
- **Concurrent** — handle simultaneous users.
- **JSON streaming** — all responses in JSON format, including streamed output.
- Replay from trajectory JSON files.

Currently under construction; not suitable for production use.

---

### 4.7 `evaluation/` — Benchmarking

Supports running Trae Agent against three benchmarks: **SWE-bench**, **SWE-bench-Live**, and **Multi-SWE-bench**.

| File | Purpose |
|------|---------|
| `run_evaluation.py` | Iterates benchmark instances, runs the agent per instance, collects patches |
| `setup.sh` | Automates Docker-based benchmark harness setup |
| `utils.py` | Shared helpers for evaluation tasks |
| `patch_selection/` | Utilities for selecting and comparing patches against ground truth |

---

## 5. Key Classes and Functions

### 5.1 Agent Layer

#### `BaseAgent.execute_task()`
```
execute_task() → AgentExecution
```
The main loop. Runs up to `max_steps` iterations:
1. Calls `_run_llm_step()` — sends current message history to the LLM.
2. Interprets `finish_reason`:
   - `stop` / `end_turn` → check if `llm_indicates_task_completed()` → break or retry.
   - `tool_calls` → call `_tool_call_handler()` → append tool results to history → continue.
3. Updates `AgentStep` state throughout.
4. Returns `AgentExecution` with full step history.

#### `BaseAgent._tool_call_handler(step, tool_calls)`
Dispatches tool calls to the `ToolExecutor` (sequential or parallel mode). Optionally generates **reflections** on tool failures by asking the LLM what went wrong.

#### `TraeAgent.get_git_diff(base_commit)`
Runs `git diff <base_commit>` in the working directory and returns the unified diff string. Used to extract the patch the agent produced.

#### `TraeAgent.remove_patches_to_tests(patch)`
Strips hunks from a unified diff that touch paths under `tests/` — required by SWE-bench evaluation rules.

---

### 5.2 Tool Layer

#### `ToolExecutor.parallel_tool_call(tool_calls)`
Uses `asyncio.gather` to run all tool calls concurrently. Returns a list of `ToolResult` in the same order.

#### `ToolExecutor.sequential_tool_call(tool_calls)`
Runs tool calls one at a time, propagating results between calls.

#### `BashTool.execute(command, restart)`
- Lazily starts a `_BashSession` subprocess.
- Writes the command plus a sentinel string; reads stdout until the sentinel is found.
- Extracts exit code via `echo $?`.
- Returns `ToolExecResult(output, error, exit_code)`.

#### `TextEditorTool.execute(command, path, ...)`
Routes to internal helpers:
- `_view(path)` — cat with line numbers or 2-level directory tree.
- `_create(path, content)` — write new file; fails if it exists.
- `_str_replace(path, old, new)` — uniqueness-checked replacement.
- `_insert(path, line, content)` — insert after specified line.

#### `SequentialThinkingTool.execute(thought, thought_number, total_thoughts, ...)`
Records a `ThoughtData` entry, optionally marks it as a revision or branch, and returns a formatted status summary with box-drawing visuals.

---

### 5.3 LLM Client Layer

#### `LLMClient.chat(messages, config, tools, reuse_history)`
Delegates to the active provider client's `chat()` method. Returns `LLMResponse`.

#### `BaseLLMClient.supports_tool_calling()`
Reads `ModelConfig.tool_calling` to determine whether the current model can invoke tools.

#### `AnthropicClient.chat(messages, config, tools, reuse_history)`
1. Converts `LLMMessage` list to Anthropic `MessageParam` objects.
2. Attaches tool schemas (with special handling for built-in Anthropic tools).
3. Calls `_create_anthropic_response()` (which has retry logic via `retry_with` decorator).
4. Parses response into `LLMResponse`; records to trajectory if configured.

#### `retry_utils.retry_with`
Decorator that retries a function call on rate-limit or transient errors with configurable backoff.

---

### 5.4 Configuration

#### `Config.create(path) → Config`
1. Reads YAML (preferred) or JSON file.
2. Instantiates `ModelProvider`, `ModelConfig`, `MCPServerConfig`, `AgentConfig`, `LakeviewConfig`.
3. Validates that every model reference resolves to a known provider + model.
4. Raises `ConfigError` on any inconsistency.

#### `ModelConfig.get_max_tokens_param() → dict`
Returns `{"max_completion_tokens": n}` for models that require it, or `{"max_tokens": n}` for others.

---

### 5.5 Observability

#### `TrajectoryRecorder.record_llm_interaction(messages, response, tools)`
Appends a serialised LLM exchange to `trajectory_data["interactions"]`.

#### `TrajectoryRecorder.record_agent_step(step)`
Appends a serialised `AgentStep` to `trajectory_data["steps"]`.

#### `TrajectoryRecorder.save_trajectory()`
Writes `trajectory_data` as indented JSON to the configured file path (default: `~/.trae-agent/trajectories/<timestamp>.json`).

#### `LakeView.create_lakeview_step(step, trajectory) → LakeViewStep`
Calls the LLM twice (extract task + assign tag) and returns a `LakeViewStep` with emoji label and descriptions.

---

### 5.6 CLI Console

#### `CLIConsole` (ABC)
Defines the interface all consoles must implement:
- `print_task_details(task)` — show task info at session start.
- `update_step(step)` — render or update the current step display.
- `get_user_input()` — prompt the user for the next task (interactive mode).
- `start() / stop()` — lifecycle management.

#### `RichConsole`
Full **Textual** TUI. Each agent step is rendered in a live panel with:
- State emoji + colour.
- LLM thought text.
- Tool call arguments (formatted JSON).
- Tool results.
- Reflection notes.
- Lakeview step summary.

---

## 6. Dependency Relationships

```
cli.py
  └── Agent (agent/agent.py)
        ├── TraeAgent (agent/trae_agent.py)
        │     ├── BaseAgent (agent/base_agent.py)
        │     │     ├── LLMClient  ──► provider clients (llm_clients/)
        │     │     └── ToolExecutor ──► Tool subclasses (tools/)
        │     ├── DockerManager (agent/docker_manager.py)
        │     └── MCPClient (utils/mcp_client.py)
        │           └── MCPTool (tools/mcp_tool.py)
        ├── TrajectoryRecorder (utils/trajectory_recorder.py)
        ├── LakeView (utils/lake_view.py)
        │     └── LLMClient  (separate lakeview model instance)
        └── CLIConsole (utils/cli/)
              └── ConsoleFactory

Config (utils/config.py)
  ├── ModelProvider
  ├── ModelConfig
  ├── MCPServerConfig
  ├── AgentConfig / TraeAgentConfig
  └── LakeviewConfig

LLMMessage / LLMResponse / LLMUsage (utils/llm_clients/llm_basics.py)
  └── used by all LLM clients and agent layer

AgentStep / AgentExecution (agent/agent_basics.py)
  └── used by BaseAgent, TrajectoryRecorder, CLIConsole
```

**External runtime dependencies** (from `pyproject.toml`):

| Package | Purpose |
|---------|---------|
| `anthropic` | Anthropic Claude API |
| `openai` | OpenAI / Azure / OpenRouter / Ollama API |
| `google-genai` | Google Gemini API |
| `click` | CLI framework |
| `pydantic` | Data validation |
| `rich` | Terminal formatting |
| `textual` | TUI framework (rich console) |
| `pyyaml` | Config file parsing |
| `docker` | Docker SDK |
| `pexpect` | Interactive shell management in Docker |
| `mcp` | Model Context Protocol client |

---

## 7. Configuration Reference

Configuration is loaded from `trae_config.yaml` (or `trae_config.json` for legacy setups). The example below reflects the canonical structure:

```yaml
# Which agent to use
agent: trae_agent

trae_agent:
  model: trae_agent_model      # reference to a model defined below
  max_steps: 200
  tools:
    - bash
    - str_replace_based_edit_tool
    - sequential_thinking
    - task_done
  lakeview:
    enabled: true
    model: lakeview_model

  # Optional: MCP servers to enable
  mcp_servers:
    - playwright

models:
  trae_agent_model:
    provider: anthropic           # LLMProvider value
    model: claude-4-sonnet
    max_tokens: 4096
    temperature: 0.5
    top_p: 1
    parallel_tool_calls: true

  lakeview_model:
    provider: anthropic
    model: claude-3.5-sonnet
    max_tokens: 4096
    temperature: 0.5
    top_p: 1

providers:
  anthropic:
    api_key: YOUR_API_KEY         # or set ANTHROPIC_API_KEY env var

# Optional: MCP server definitions
mcp_servers:
  playwright:
    type: stdio
    command: npx
    args: ["@playwright/mcp@0.0.27"]
```

**Supported providers:** `openai`, `anthropic`, `azure`, `ollama`, `openrouter`, `doubao`, `google`.

Environment variable overrides (take precedence over config file values):
- `OPENAI_API_KEY`
- `ANTHROPIC_API_KEY`
- `GOOGLE_API_KEY`
- `AZURE_OPENAI_API_KEY`

---

## 8. Running the Project

### Prerequisites

- Python ≥ 3.12
- [uv](https://github.com/astral-sh/uv) package manager
- An API key for at least one supported LLM provider
- (Optional) Docker, for isolated task execution

### Installation

```bash
# Clone the repository
git clone https://github.com/commercegtm/trae-agent.git
cd trae-agent

# Install all dependencies (including optional extras)
uv sync --all-extras

# Or use the Makefile shortcut
make install-dev
```

### Configuration

```bash
# Copy the example config
cp trae_config.yaml.example trae_config.yaml

# Edit and set your API key
# providers.anthropic.api_key: sk-ant-...
# or export ANTHROPIC_API_KEY=sk-ant-...
```

### Basic Usage

```bash
# Run a task
trae-cli run "Fix the bug in issue #42" --working-dir /path/to/repo

# Run a task and save the trajectory
trae-cli run "Add logging to the payment module" \
  --working-dir /path/to/repo \
  --trajectory-file my-run.json

# Run with a specific provider and model
trae-cli run "Refactor the data pipeline" \
  --provider openai --model gpt-4o \
  --working-dir /path/to/repo

# Read task description from a file
trae-cli run --task-file issue.txt --working-dir /path/to/repo

# Interactive mode
trae-cli interactive --working-dir /path/to/repo

# Interactive mode with rich TUI
trae-cli interactive --console rich --working-dir /path/to/repo
```

### Docker Mode

```bash
# Run inside a Docker image
trae-cli run "Fix memory leak" \
  --working-dir /path/to/repo \
  --docker-image python:3.12-slim

# Use an existing container
trae-cli run "Run the test suite" \
  --working-dir /path/to/repo \
  --docker-container my-container-id

# Build from a Dockerfile
trae-cli run "Implement feature X" \
  --working-dir /path/to/repo \
  --docker-file ./Dockerfile
```

### Utility Commands

```bash
# Display resolved configuration
trae-cli show_config

# List available tools
trae-cli tools
```

---

## 9. Development Guide

### Running Tests

```bash
# Run the full test suite (skips Ollama, OpenRouter, Google by default)
make test

# Or directly with pytest
uv run pytest

# Run only unit tests
uv run pytest -m unit

# Run with coverage
uv run pytest --cov=trae_agent
```

### Code Quality

```bash
# Format and auto-fix linting issues (Ruff)
make fix-format

# Install pre-commit hooks
make install-pre-commit

# Run pre-commit on all files
make pre-commit
```

### Evaluation Against SWE-bench

```bash
# Set up the evaluation environment
bash evaluation/setup.sh

# Run evaluation on a benchmark
uv run python evaluation/run_evaluation.py \
  --benchmark swe-bench \
  --output-dir ./results
```

### Project Conventions

| Convention | Detail |
|------------|--------|
| Line length | 100 characters (Ruff) |
| Linting rules | B, SIM, C4, E, F, I |
| Test markers | `slow`, `integration`, `unit` |
| Async test mode | `asyncio_mode = "auto"` |
| Storage root | `~/.trae-agent/` |
| Trajectory files | `~/.trae-agent/trajectories/<timestamp>.json` |
| Config file | `trae_config.yaml` (project root or `~/.trae-agent/`) |

### Adding a New LLM Provider

1. Create `trae_agent/utils/llm_clients/<provider>_client.py` extending `BaseLLMClient` (or `OpenAICompatibleBase` if OpenAI-API-compatible).
2. Add the provider name to the `LLMProvider` enum in `llm_client.py`.
3. Add a `case` branch in `LLMClient.__init__()` to instantiate the new client.
4. Add any provider-specific config fields to `ModelProvider` in `config.py`.

### Adding a New Tool

1. Create `trae_agent/tools/<my_tool>.py` with a class extending `Tool`.
2. Implement `get_name()`, `get_description()`, `get_parameters()`, and `execute()`.
3. Register the tool in `TraeAgent`'s default tool list (`trae_agent/agent/trae_agent.py`).
4. Add the tool name to the `tools` list in your `trae_config.yaml`.

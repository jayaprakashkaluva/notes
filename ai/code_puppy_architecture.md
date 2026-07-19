# Code Puppy — Architecture & Technical Implementation

> Analysis of the repository at `C:\workspace\opensource\code_puppy` (branch `main`, version `0.0.651`).
> Everything in this document is grounded in the source code; file paths are given so claims can be verified.

---

## 1. What it is

**Code Puppy** is an open-source (MIT), terminal-based AI coding agent — a CLI alternative to Windsurf/Cursor — published on PyPI as `code-puppy`. It is a pure-Python application (Python `>=3.11,<3.15`) built on **pydantic-ai** as the agent framework, with a rich interactive terminal UI, a plugin system, MCP (Model Context Protocol) support, multi-provider model management, and browser automation via Playwright.

- **Package:** `code_puppy/` (463 Python files)
- **Tests:** `tests/` (~486 Python files, pytest)
- **Entry points** (from `pyproject.toml`): `code-puppy` and `pup`, both mapping to `code_puppy.main:main_entry`
- **Build backend:** hatchling; dependency lockfile: `uv.lock` (uv)
- **Docs in repo:** `README.md`, `AGENTS.md`, `docs/` (`AGENT_SKILLS.md`, `CEREBRAS.md`, `FLUX.md`, `HOOKS.md`, `I18N.md`, `LEFTHOOK.md`)

---

## 2. High-level architecture

```
                         ┌─────────────────────────────────────────────┐
                         │  Entry: main.py → cli_runner.py (argparse)  │
                         │  pydantic_patches applied BEFORE any        │
                         │  pydantic-ai import                         │
                         └──────────────────┬──────────────────────────┘
                                            │
        ┌─────────────┬─────────────────────┼──────────────────┬────────────────┐
        ▼             ▼                     ▼                  ▼                ▼
┌──────────────┐ ┌───────────┐   ┌──────────────────┐  ┌─────────────┐  ┌────────────┐
│ command_line/ │ │ messaging/│   │     agents/      │  │  plugins/   │  │ hook_engine│
│ REPL commands │ │ MessageBus│   │ BaseAgent + JSON │  │ ~55 builtin │  │ user hooks │
│ /slash menus  │ │ TUI, line │   │ agents, runtime, │  │ + user +    │  │ (HOOKS.md) │
│ completion    │ │ editor    │   │ compaction       │  │ project     │  └────────────┘
└──────────────┘ └───────────┘   └────────┬─────────┘  └─────────────┘
                                          │ builds pydantic-ai Agent
                        ┌─────────────────┼───────────────────┐
                        ▼                 ▼                   ▼
                 ┌─────────────┐  ┌──────────────┐   ┌──────────────────┐
                 │   tools/    │  │model_factory │   │      mcp_/       │
                 │ TOOL_REGISTRY│ │ multi-provider│  │ MCPManager,      │
                 │ file/shell/ │  │ model builder │   │ health, circuit  │
                 │ browser/UC  │  │ models.json   │   │ breaker, registry│
                 └─────────────┘  └──────────────┘   └──────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────────┐
        │ config.py + session_storage.py            │
        │ ~/.code_puppy (XDG-aware): puppy.cfg,     │
        │ models.json, mcp_servers.json, agents/,   │
        │ autosaves (pickle), secret_store (keyring)│
        └───────────────────────────────────────────┘
```

### Package map

| Package / module | Responsibility |
|---|---|
| `code_puppy/main.py`, `cli_runner.py` | Entry point; arg parsing (argparse), interactive REPL loop, resume/quick-resume, turn-level exception rendering |
| `code_puppy/agents/` | Agent abstraction, built-in agents, JSON-defined agents, agent registry/manager, run orchestration, history compaction, streaming render |
| `code_puppy/tools/` | Tool registry and all built-in tool implementations (file, shell, browser, skills, agent invocation, Universal Constructor) |
| `code_puppy/model_factory.py` + friends | Constructs pydantic-ai `Model` objects for every supported provider from JSON model configs |
| `code_puppy/mcp_/` | MCP server lifecycle management (registry, health monitor, circuit breaker, retries, dashboard) |
| `code_puppy/messaging/` | Message bus (agent → UI), Rich rendering, custom raw-mode line editor / bottom bar TUI |
| `code_puppy/command_line/` | Interactive slash-commands, menus (model picker, `/add_model`, MCP wizard, onboarding), prompt-toolkit completion |
| `code_puppy/plugins/` | Built-in plugin suite + loader for user/project plugins |
| `code_puppy/callbacks.py` | Global phase-based callback (hook) system that plugins attach to |
| `code_puppy/hook_engine/` | User-configurable lifecycle hooks (registry/matcher/executor/validator) |
| `code_puppy/config.py` | XDG-aware config, paths, `puppy.cfg` accessors |
| `code_puppy/session_storage.py`, `session_lifecycle.py`, `session_migration.py` | Chat session persistence (pickle + JSON metadata), autosave, migration |
| `code_puppy/i18n/` | Stdlib-only internationalization framework |
| `code_puppy/secret_store.py`, `secret_store_backends.py` | Secrets via OS keyring |
| `code_puppy/hook_engine/`, `mcp_prompts/`, `error_logging.py`, `undo_manager.py`, `version_checker.py` | Supporting subsystems |

---

## 3. Startup & entry flow

`code_puppy/main.py` is a thin re-export; all logic lives in `code_puppy/cli_runner.py`:

1. **First lines of `cli_runner.py`** call `apply_all_patches()` from `code_puppy/pydantic_patches.py` *before any pydantic-ai import*. One notable patch: `patch_tool_call_json_repair()` intercepts tool calls and runs **json-repair** on malformed model-emitted JSON (missing quotes, trailing commas) instead of failing the turn.
2. `plugins.load_plugin_callbacks()` runs at module import, wiring all plugin callbacks before the app starts.
3. `main()` builds an `argparse.ArgumentParser` (note: **typer** is declared in `pyproject.toml` but no `import typer` exists anywhere in the package — argparse is the actual CLI parser).
4. Interactive mode enters the REPL; `_render_turn_exception()` classifies exceptions with the same transient-error classifier used by streaming auto-retry (`should_retry_streaming_exception`) — transient network/provider blips get a friendly one-liner, real bugs get a full traceback, and everything is persisted to `~/.code_puppy/logs/errors.log` via `error_logging.py`.
5. `--quick-resume [PATH]` (`apply_quick_resume`) resolves the most recent autosave scoped to the nearest git worktree root + branch and feeds it into the normal `--resume` machinery.
6. `pyfiglet` renders the ASCII-art intro banner (`cli_runner.py:245`, also used in onboarding slides).

---

## 4. Agent subsystem (`code_puppy/agents/`)

### 4.1 `BaseAgent` — a deliberately thin conductor

`agents/base_agent.py` is an ABC kept intentionally under ~300 lines; its docstring documents the decomposition:

- `_history.py` — token estimation, message hashing, orphan pruning
- `_compaction.py` — summarization/truncation + the history-processor factory
- `_builder.py` — pydantic-ai `Agent` construction + MCP wiring
- `_runtime.py` — `run_with_mcp` orchestration, cancellation, streaming retry classification
- `_key_listeners.py` — Ctrl+X / cancel-agent keyboard listener daemon threads

Abstract interface: `name`, `display_name`, `description`, `get_system_prompt()`, `get_available_tools()`. State includes message history, a set of compacted-message hashes, the built pydantic-ai agent, a cached "tool probe" agent (used to count tool token overhead before the real agent is built), and per-run model override support.

### 4.2 Built-in agents

Each is a `BaseAgent` subclass in `agents/`:

| Class | File |
|---|---|
| `CodePuppyAgent` (the default coding agent; system prompt personalized with puppy/owner name from config) | `agent_code_puppy.py` |
| `AgentCreatorAgent` | `agent_creator_agent.py` |
| `HeliosAgent` | `agent_helios.py` |
| `PlanningAgent` | `agent_planning.py` |
| `QualityAssuranceKittenAgent` | `agent_qa_kitten.py` |

### 4.3 JSON-defined agents

`agents/json_agent.py` defines `JSONAgent(BaseAgent)`: agents declared purely as JSON files with required fields `name`, `description`, `system_prompt` (string or list), `tools` (list), plus optional `mcp_servers` (shorthand list of names or full configs). `discover_json_agents()` scans the user agents directory (`~/.code_puppy/agents`, see `AGENTS_DIR` in `config.py`).

### 4.4 Agent manager

`agents/agent_manager.py` maintains `_AGENT_REGISTRY` (mixing Python classes and JSON file paths), per-agent message histories, and **per-terminal-session agent selection** keyed by parent process ID (`get_terminal_session_id()` → `session_{PPID}`), persisted in `STATE_DIR/terminal_sessions.json`. Discovery is serialized with an `RLock` because plugin callbacks fired during discovery may re-enter manager APIs. Plugins can contribute agents via the `register_agents` callback.

### 4.5 Agent construction (`_builder.py`)

`build_pydantic_agent()` is the single entry point that:
- instantiates `pydantic_ai.Agent` with the model from `ModelFactory`,
- loads "puppy rules" from repo instruction files (`AGENTS.md`, `AGENT.md`, `agents.md`, `agent.md` — `_AGENT_RULE_FILES`), size-capped via `get_agents_md_max_chars`,
- attaches history processors (`make_history_processor` for compaction, `make_steer_history_processor` for mid-run steering),
- wires the streaming `event_stream_handler`,
- attaches MCP toolsets from `MCPManager` (with per-agent bindings and tool filtering),
- lets plugins wrap the finished agent via the `wrap_pydantic_agent` callback.

### 4.6 Context compaction

`_compaction.py` + `summarization_agent.py`: when history grows past thresholds (protected token count from config), older messages are summarized by a dedicated pydantic-ai `Agent` built around a configurable summarization model (`get_summarization_model_name()`); the summarizer runs on a separate thread-pool event loop to avoid "event loop already running" issues. Compacted messages are tracked by hash so they are never re-summarized.

### 4.7 Sub-agents

`tools/subagent_invocation.py` + `agents/subagent_stream_handler.py` register `invoke_agent` / `invoke_agent_with_model` tools, letting one agent spawn another (optionally on a different model) with its own streamed output panel (`plugins/subagent_panel`).

---

## 5. Tool system (`code_puppy/tools/`)

`tools/__init__.py` defines a flat `TOOL_REGISTRY: dict[str, register_func]` — each tool has a `register_<name>(agent)` function that attaches it to the pydantic-ai agent. Agents request tools by name (`get_available_tools()`), and `register_tools_for_agent()` resolves the list.

### Tool inventory (registry keys, grounded in `tools/__init__.py`)

- **Agent tools:** `list_agents`, `invoke_agent`, `invoke_agent_with_model`, `list_available_models`
- **File operations** (`file_operations.py`): `list_files`, `read_file`, `grep` — the grep/list implementation shells out to a **ripgrep** binary (found on PATH or in the virtualenv; the pip package `ripgrep==14.1.0` guarantees availability), with a pluggable IO-backend abstraction (`io_backends.py`, `fs_access.py`)
- **File modifications** (`file_modifications.py`): `create_file`, `replace_in_file`, `delete_snippet`, `delete_file`; the legacy `edit_file` is in `TOOL_EXPANSIONS` and auto-expands into the three granular tools
- **Shell** (`command_runner.py`): `agent_run_shell_command`, `agent_share_your_reasoning`, with background-process support (`shell_backgrounding.py`)
- **User interaction:** `ask_user_question` (its own subpackage)
- **Browser automation** (~40 tools in `tools/browser/`, built on **Playwright** async API — `browser_manager.py` imports `playwright.async_api` and starts `async_playwright()`): lifecycle (`browser_initialize`/`close`/`status`/`new_page`/`list_pages`), navigation, element discovery (by role/text/label/placeholder/test-id/XPath), accessibility-first semantic interactions (`browser_page_snapshot`, `click_by_role`, …), low-level interactions (click/hover/set_text/check/…), scripting (`browser_execute_js`, scrolling, viewport, highlights), `browser_screenshot_analyze`, and saved workflows
- **Images:** `load_image_for_analysis` (`image_tools.py`; **Pillow** is used in clipboard/image handling under `command_line/`)
- **Skills:** `activate_skill`, `list_or_search_skills` (`skills_tools.py`; skill files live in `SKILLS_DIR`, documented in `docs/AGENT_SKILLS.md`)
- **Universal Constructor:** `universal_constructor` — user-defined Python tools registered at runtime; `_register_uc_tool_wrapper()` dynamically builds an async wrapper that *preserves the original function's signature and annotations* so pydantic-ai can generate a correct JSON schema. UC tools are requested with a `uc:` name prefix and gated by a config flag.

Other mechanics in `tools/__init__.py`:
- **Kill switch:** env var `CODE_PUPPY_NO_TOOLS` (set by the `no_tools` plugin for `--no-tools`) disables all tool registration — the model runs pure text-in/text-out and tool schemas are kept out of requests to save tokens.
- **Plugin tools:** the `register_tools` callback merges plugin tool definitions into `TOOL_REGISTRY`; `register_agent_tools(agent_name)` lets plugins add tools per-agent.
- **Extended thinking interplay:** `has_extended_thinking_active()` detects Anthropic models with extended thinking enabled/adaptive; the redundant `share_your_reasoning` tool is dropped and a prompt note (`EXTENDED_THINKING_PROMPT_NOTE`) encourages native thinking blocks instead.

---

## 6. Model layer

### 6.1 `model_factory.py`

`ModelFactory` turns JSON model configs into pydantic-ai `Model` instances. Supported constructions (all visible in imports/logic of `model_factory.py`):

- **Anthropic:** `pydantic_ai.models.anthropic.AnthropicModel` over `anthropic.AsyncAnthropic`, with a custom `ClaudeCacheAsyncClient` (`claude_cache_client.py`) that patches the messages endpoint for prompt caching. `_build_anthropic_beta_header()` assembles beta flags: `interleaved-thinking-2025-05-14`, `context-1m-2025-08-07` (when `context_length >= 1_000_000`), and a dormant `advisor-tool-2026-03-01` opt-in.
- **OpenAI-family:** `OpenAIChatModel` and `OpenAIResponsesModel`, including "custom OpenAI" endpoint types (`custom_openai`, `custom_openai_responses`) and `AsyncAzureOpenAI`.
- **Providers:** `CerebrasProvider`, `OpenRouterProvider` (from pydantic-ai), plus `provider_identity.py` helpers (`make_anthropic_provider`, `make_openai_provider`).
- **Gemini:** custom `GeminiModel` (`gemini_model.py`) and Gemini Code Assist support (`gemini_code_assist.py`).
- **`RoundRobinModel`** (`round_robin_model.py`): rotates across multiple underlying models.
- **Plugin providers:** `register_model_providers` callback lets plugins contribute model-provider classes (`_CUSTOM_MODEL_PROVIDERS`), used by e.g. the OAuth plugins.
- HTTP clients are built by `http_utils.create_async_client` (**httpx** with optional HTTP/2 and custom cert bundles); `reopenable_async_client.py` allows re-opening closed async clients.

### 6.2 Model catalogs

- `code_puppy/models.json` ships **empty by default** (`{}`) — models come from user config and OAuth/provider plugins.
- User-level files under the data dir (`config.py`): `models.json`, `extra_models.json`, plus per-OAuth-plugin catalogs (`gemini_models.json`, `chatgpt_models.json`, `claude_models.json`, `copilot_models.json`).
- **models.dev integration:** `/add_model` (`command_line/add_model_menu.py`) fetches the live models.dev catalog (65+ providers) and falls back to the bundled snapshot `code_puppy/models_dev_api.json`, parsed by `models_dev_parser.py`.
- `model_switching.py`, `model_utils.py`, `model_descriptions.py` handle runtime switching, per-agent pinned models (`get_agent_pinned_model`), and effective per-model settings (temperature, extended thinking, etc.).

### 6.3 Provider auth plugins

Under `plugins/`: `chatgpt_oauth`, `claude_code_oauth`, `copilot_auth`, `grok_oauth` (OAuth flows with `oauth_pasteback.py` / `oauth_puppy_html.py` helpers), `aws_bedrock` (**boto3**), `azure_foundry` (**azure-identity** token acquisition), `ollama` / `ollama_setup`. `chatgpt_codex_client.py` implements the ChatGPT Codex backend client.

---

## 7. MCP subsystem (`code_puppy/mcp_/`)

Built on pydantic-ai's MCP support (`MCPServerSSE`, `MCPServerStdio`, `MCPServerStreamableHTTP` from `pydantic_ai.mcp`), with substantial reliability engineering on top:

- **`manager.py` — `MCPManager`:** central coordinator; provides servers to agents, warns (once per `(server, agent)` pair, silenceable via `/mcp silence-warning`) when servers are registered in `mcp_servers.json` but not bound to the current agent.
- **`managed_server.py`:** `ManagedMCPServer` + `ServerConfig`/`ServerState` wrappers.
- **Reliability:** `health_monitor.py`, `circuit_breaker.py`, `retry_manager.py`, `error_isolation.py`, `status_tracker.py`, `async_lifecycle.py`, `blocking_startup.py`.
- **`captured_stdio_server.py`:** captures stdio-server output for the `/mcp logs` view (`mcp_logs.py`).
- **`server_registry_catalog.py`:** a catalog of known installable servers (plugins can extend it via `register_mcp_catalog_servers`).
- **`agent_bindings.py`:** binds specific servers to specific agents; `tool_arg_coercion.py` coerces mismatched tool-call argument types.
- **UI:** the whole `/mcp` command suite lives in `command_line/mcp/` (install wizard, catalog installer, custom server form, start/stop/restart/status/logs/trust/search commands), plus `dashboard.py`.

---

## 8. Messaging & terminal UI (`code_puppy/messaging/`)

Two coexisting APIs (documented in `messaging/__init__.py`):

- **Legacy API:** `emit_info/emit_warning/emit_error/...` + `MessageQueue`/`queue_console.py` — used throughout the codebase; a Rich-compatible console whose output flows through the queue so the TUI can interleave it safely.
- **Structured API:** `MessageBus` (`bus.py`) with Pydantic message models (`messages.py`: `TextMessage`, `DiffMessage`, …) for agent → UI, and command models (`commands.py`: `CancelAgentCommand`, `PauseAgentCommand`, `SteerAgentCommand`, `UserInputResponse`, …) for UI → agent. Rendering via `RichConsoleRenderer` (`rich_renderer.py`, `renderers.py`).

The interactive UI is **not** a full-screen TUI framework; it is a hybrid:
- **prompt-toolkit** powers input completion and interactive menus (used across `command_line/` — model picker, agent menu, autosave search, MCP forms, set-menu, onboarding wizard).
- A **custom raw-mode line editor** (`messaging/line_editor.py`, `RunningLineEditor`) implements the persistent bottom-bar prompt: a key-listener daemon thread (`agents/_key_listeners.py`) feeds raw characters; the editor handles a full Emacs-style keymap, bracketed paste, Ctrl+V smart paste (image or text), Ctrl+X chord registry, reverse search, multiline mode, and **live steering** — typing while the agent runs routes submissions into steer/slash-drain (`run_ui.py`, `run_ui_wiring.py`, `_steer_processor.py`).
- `bottom_bar.py`, `bar_painters.py`, `inline_bar.py`, `spinner/`, `pause_controller.py` render status; `markdown_patches.py` patches Rich's Markdown rendering (left-justified headers, unpadded code blocks).
- **termflow-md** renders streaming markdown output (`agents/event_stream_handler.py`, `agents/smooth_stream.py`); `smooth_stream.py` paces token output.

---

## 9. Plugin & callback architecture

### 9.1 Callback system (`callbacks.py`)

A global phase-based hook registry: `PhaseType` is a `Literal` of ~65 phases spanning the whole lifecycle — `startup`/`shutdown`, tool phases (`pre_tool_call`, `post_tool_call`, per-file-op phases), model phases (`load_model_config`, `register_model_providers`, `prepare_model_prompt`), agent-run phases (`agent_run_start/end/result/cancel`, `wrap_pydantic_agent`, `message_history_processor_start/end`), UI phases (`stream_event`, `termflow_style`, `prompt_toolkit_style`, `notification`), and session phases (`user_prompt_submit`, `pre_compact`, `session_end`, `post_autosave`). Plugins register callables per phase; core code invokes `on_<phase>()` at the right moments.

### 9.2 Plugin loader (`plugins/__init__.py`)

Three tiers, loaded once per process:
1. **Built-in** — every directory under `code_puppy/plugins/` with a `register_callbacks.py` (imported via `importlib`); `shell_safety` is conditionally skipped based on the configured safety permission level.
2. **User** — `~/.code_puppy/plugins/`.
3. **Project** — per-repo plugins with a **trust model** (`plugins/trust.py`, `trust_notice.py`): statuses `loaded | untrusted | changed | disabled | error` surfaced in the `/plugins` UI.

### 9.3 Notable built-in plugins (directory names under `code_puppy/plugins/`)

`acp` (Agent Client Protocol — see §11), `dbos_durable_exec` (optional DBOS durable execution, `pip install code-puppy[durable]`, toggled via `/dbos on|off`), `universal_constructor`, `claude_code_hooks` / `hook_manager` / `hook_creator`, safety guards (`destructive_command_guard`, `force_push_guard`, `shell_safety`, `file_permission_handler`), UX (`theme`, `statusline`, `puppy_spinner`, `subagent_panel`, `wide_completion_menu`, `context_indicator`, `prompt_newline`), workflow (`fork`, `quick_resume`, `switch_agent_resume`, `review_pr`, `prune`, `pop_command`, `steer_queue`), `flux_bootstrap` (installs the bundled Flux command suite into `~/.code_puppy`, see `docs/FLUX.md`), `frontend_emitter`, `token_ratio_learner`, `obsidian_agent`, `puppy_kennel`, `wiggum`, `yolo_cli`, and more (~55 total).

### 9.4 Hook engine (`hook_engine/`)

Separate from the plugin callback system: **user-configurable hooks** (documented in `docs/HOOKS.md`) with a `HookEngine` orchestrator, config `validator`, `registry` builder, event `matcher`, and sequential `executor` supporting blocking results (e.g., a hook can block a tool call). `mcp_prompts/hook_creator.py` and the `hook_creator` plugin assist in authoring hooks.

---

## 10. Configuration, persistence & security

### 10.1 Config (`config.py`)

- **XDG-aware with a legacy default:** `_get_xdg_dir()` uses `$XDG_CONFIG_HOME`/`$XDG_DATA_HOME`/`$XDG_CACHE_HOME`/`$XDG_STATE_HOME` *only if explicitly set*; otherwise everything lives in `~/.code_puppy`.
- Main config file: `puppy.cfg` (parsed with stdlib `configparser`); `python-dotenv` loads `.env` files.
- Key paths: `mcp_servers.json` (config dir); `models.json`, `extra_models.json`, `agents/`, `skills/`, `contexts/` (data dir); `terminal_sessions.json` (state dir).

### 10.2 Sessions (`session_storage.py`, `session_lifecycle.py`)

Chat sessions persist as **pickle** files plus JSON metadata (`SessionPaths`, `SessionMetadata` dataclasses); a legacy signed-header format (`CPSESSION\x01`) is still parsed for backward compatibility. Autosave writes into `AUTOSAVE_DIR`, and resume flows (`--resume`, `--quick-resume`, `/autosave` menus) restore full message histories. `undo_manager.py` supports undoing file changes.

### 10.3 Secrets (`secret_store.py`, `secret_store_backends.py`)

API keys and OAuth tokens are stored via the **keyring** library (OS credential store) with pluggable backends.

### 10.4 i18n (`code_puppy/i18n/`)

A deliberately **stdlib-only** localization framework: JSON message catalogs with a fallback chain (`catalog.py`), locale detection/override (`locale.py`), plural-aware translation (`t()` / `ngettext`, `plurals.py`), locale-aware number/date formatting (`formats.py`), **pseudolocalization** for coverage testing (`pseudo.py`), and a static-extraction audit tool (`audit.py`). Shipped locales: `en-US`, `es`, `es-AR`, `es-CL`, `es-CO`, `es-MX`, `fr-CA`.

---

## 11. ACP — editor integration (`plugins/acp/`)

Code Puppy can run as a native **Agent Client Protocol** agent (JSON-RPC 2.0 over stdio) inside an editor's agent panel, booted with `code-puppy --acp`. Implemented against the `agent-client-protocol` package (`from acp import Agent`, `acp.schema`), with an `EventBridge` translating Code Puppy's message bus into ACP session updates, plus modules for capabilities, permissions, MCP config passthrough, and session resume/fork.

---

## 12. Frameworks & libraries — what each is used for

All version pins are from `pyproject.toml`; usage locations verified in source.

### Runtime dependencies

| Library | Pin | How it is used |
|---|---|---|
| **pydantic-ai-slim** `[openai,anthropic,mcp]` | `==1.56.0` | The core agent framework: `pydantic_ai.Agent` is built per Code Puppy agent (`agents/_builder.py`); model classes (`AnthropicModel`, `OpenAIChatModel`, `OpenAIResponsesModel`), providers (Cerebras, OpenRouter), MCP server classes (`pydantic_ai.mcp.*`), message types (`pydantic_ai.messages.ModelMessage`), and `RunContext` for tools. Patched at startup by `pydantic_patches.py`. |
| **pydantic** | `>=2.4.0` | Data models throughout: structured UI messages (`messaging/messages.py`), UI commands, MCP configs, hook models (`hook_engine/models.py`). |
| **anthropic** | `==0.79.0` | `AsyncAnthropic` client for Anthropic models; wrapped by `ClaudeCacheAsyncClient` (`claude_cache_client.py`) for prompt caching; beta headers assembled in `model_factory.py`. |
| **openai** | `>=1.99.1` | OpenAI SDK; `AsyncAzureOpenAI` for Azure endpoints (`model_factory.py`); ChatGPT Codex client (`chatgpt_codex_client.py`). |
| **mcp** | `>=1.9.4` | Model Context Protocol SDK backing the `mcp_/` subsystem via pydantic-ai's MCP integration. |
| **httpx** `[http2]` | `>=0.24.1` | All async HTTP: custom clients with HTTP/2 and cert-bundle support (`http_utils.py`, `reopenable_async_client.py`). |
| **requests** | `>=2.28.0` | Synchronous HTTP (e.g., version checks, catalog fetches). |
| **rich** | `>=13.4.2` | All terminal rendering: consoles, markdown (patched in `messaging/markdown_patches.py`), panels, tracebacks (`RichConsoleRenderer`, `queue_console.py`). |
| **prompt-toolkit** | `>=3.0.52` | Input completion and interactive menus across `command_line/` (model picker, agent menu, MCP wizard, set-menu, onboarding) and key-listener integration. |
| **termflow-md** | `>=0.1.11` | Streaming markdown renderer for live agent output (`agents/event_stream_handler.py`, `agents/smooth_stream.py`, themable via the `theme` plugin). |
| **playwright** | `>=1.40.0` | Browser automation: `playwright.async_api` drives the ~40 `browser_*` tools (`tools/browser/browser_manager.py`). |
| **ripgrep** | `==14.1.0` | Ships the `rg` binary via pip; `tools/file_operations.py` shells out to it for `grep` and recursive listings. |
| **json-repair** | `>=0.46.2` | Repairs malformed JSON in model tool calls (`pydantic_patches.py: patch_tool_call_json_repair`). |
| **pyfiglet** | `>=0.8.post1` | ASCII-art startup banner (`cli_runner.py:245`) and onboarding slides. |
| **rapidfuzz** | `>=3.13.0` | Fuzzy matching (`tools/common.py`, list filtering/completion). |
| **Pillow** | `>=10.0.0` | Image handling for clipboard paste and image attachments (`command_line/image_utils.py`, `clipboard.py`, `attachments.py`). |
| **python-dotenv** | `>=1.0.0` | `.env` loading in `config.py`. |
| **keyring** | `>=24.0.0` | OS-credential-store secret storage (`secret_store.py`, `secret_store_backends.py`). |
| **azure-identity** | `>=1.15.0` | Azure AD token acquisition for the `azure_foundry` plugin. |
| **boto3** | `>=1.43.9` | AWS Bedrock model provider plugin (`plugins/aws_bedrock`); also an optional `bedrock` extra. |
| **agent-client-protocol** | `>=0.11.0,<0.12` | ACP (JSON-RPC over stdio) editor integration (`plugins/acp/`). |
| **typer** | `>=0.12.0` | **Declared but not imported anywhere in `code_puppy/`** — actual CLI parsing uses stdlib `argparse` (`cli_runner.py`). Effectively an unused/legacy dependency. |
| **dbos** (optional extra `durable`) | `>=2.11.0` | Durable execution / crash-resumable agent runs via the `dbos_durable_exec` plugin; off by default. |

### Development & build tooling

| Tool | Evidence | Role |
|---|---|---|
| **hatchling** | `[build-system]` in `pyproject.toml` | Build backend; ships `models.json` / `models_dev_api.json` as shared data and the bundled Flux suite in the sdist. |
| **uv** | `uv.lock` in repo root; README installs via `uvx code-puppy` | Dependency locking and recommended installer. |
| **pytest** + **pytest-cov** + **pytest-asyncio** | dev-dependencies; `tests/` (~486 files), `coverage.json` in root | Test suite with async tests and coverage. |
| **ruff** | dev-dependency; `[tool.ruff]` config with per-file ignores and a vendored-code exclusion (`plugins/flux_bootstrap/bundled`) | Linting/formatting. |
| **pexpect** | dev-dependency | Terminal/interactive-behavior tests. |
| **lefthook** | `lefthook.yml`, `docs/LEFTHOOK.md` | Git hooks runner. |
| **logfire** | `[tool.logfire] ignore_no_config = true` in `pyproject.toml` | Present as tool config so pydantic-ai's logfire integration stays silent without configuration. |

---

## 13. Cross-cutting implementation notes

- **Threading model:** the agent run is async (asyncio) while keyboard listening (`_key_listeners.py`), the line editor (`line_editor.py`), agent-manager session state, and the summarizer thread pool are thread-based; locks (`threading.Lock`/`RLock`) guard shared registries.
- **Resilience by design:** transient-error classification shared between streaming retry and REPL error rendering; MCP circuit breaker + health monitor + retry manager; JSON repair on tool calls; plugin failures are logged but never break core (`_load_plugin_tools` swallows exceptions deliberately).
- **Extensibility surface:** four distinct mechanisms — the phase callback system (`callbacks.py`), the plugin tiers (built-in/user/project with trust), user-configurable hook engine (`hook_engine/`), and runtime user-defined tools (Universal Constructor). Agents themselves are user-extensible via JSON files.
- **Token economy:** tool-probe agent measures tool schema overhead before building the real agent; `--no-tools` strips schemas entirely; compaction protects a configurable token budget; a `token_ratio_learner` plugin tunes estimation.

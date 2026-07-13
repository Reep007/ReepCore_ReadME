# Jarvis — ReepCore AI Assistant

Jarvis is the built-in AI assistant in the ReepCore TUI. It runs locally through [Ollama](https://ollama.com), understands your ReepCore system and repository, and can open files in **reepedit**, run commands in an embedded **zsh** terminal, and preview unified diffs before you apply edits.

Implementation lives in `src/tui_ai_assistant.rs`, `src/jarvis_pty.rs`, `src/jarvis_rag/`, `src/jarvis_patch.rs`, `src/jarvis_web.rs`, `src/jarvis_agent.rs`, `src/jarvis_tools.rs`, `src/jarvis_voice.rs`, `src/jarvis_planner.rs`, `src/jarvis_specialist.rs`, `src/jarvis_dispatch.rs`, `src/jarvis_models.rs`, `src/jarvis_sandbox.rs`, `src/jarvis_jetbrains.rs`, `src/jarvis_security_triage.rs`, `src/jarvis_session.rs`, `src/jarvis_chat_markdown.rs`, and the Jarvis view wiring in `src/tui.rs`.

---

## Prerequisites

1. **Ollama** running and reachable (default: `http://127.0.0.1:11434`).
2. A model pulled and available under the name you configure (see [Configuration](#configuration)).
3. Start the TUI: `reepcore` (or `reepcore tui`).

```bash
# Example setup (12 GB GPU — matches templates/reepcore-setup/ai.yaml)
# GPU daemon (:11434)
ollama pull qwen3.5:9b
ollama pull qwen3.5:4b
ollama pull qwen3.5:2b
ollama pull qwen2.5-coder:14b

mkdir -p ~/.config/ollama
cp REEPCORE/templates/reepcore-setup/config/ollama/*.Modelfile ~/.config/ollama/
# my-agent.Modelfile is user-maintained (not templated) — create on qwen3.5:9b with think enabled

ollama create my-agent -f ~/.config/ollama/my-agent.Modelfile
ollama create coding-agent -f ~/.config/ollama/coding-agent.Modelfile
ollama create planner-agent -f ~/.config/ollama/planner-agent.Modelfile
ollama create review-agent -f ~/.config/ollama/review-agent.Modelfile
ollama create friday-agent -f ~/.config/ollama/friday-agent.Modelfile

# CPU daemon (:11435) — embed + security triage
ollama-cpu.sh start
OLLAMA_HOST=http://127.0.0.1:11435 ollama pull qwen3.5:4b
OLLAMA_HOST=http://127.0.0.1:11435 ollama create security-agent -f ~/.config/ollama/security-agent.Modelfile
OLLAMA_HOST=http://127.0.0.1:11435 ollama pull qwen3-embedding:0.6b

reepcore init   # symlinks ~/.config/reepcore/ai.yaml → template when missing
reepcore
```

`my-agent:latest` and all `*-agent:latest` tags are **local** models (`ollama create` from Modelfiles), not registry pulls. After editing any Modelfile, recreate with the same `ollama create <name> -f …` command. Full Modelfile reference: [`templates/reepcore-setup/config/ollama/README.md`](../templates/reepcore-setup/config/ollama/README.md).

Or configure models in `~/.config/reepcore/ai.yaml` (see [AI model manifest](#ai-model-manifest)) — the shipped template only sets `auto_pull: true` on **`embed`**; every other role uses `auto_pull: false` so Jarvis never tries to pull local `*-agent` tags from the registry.

---

## Opening Jarvis

From the **Main Menu**, select **2. Jarvis** (or press `2` while the menu is focused).

The view has up to three panes:

| Pane | Purpose |
|------|---------|
| **Chat** (left) | Conversation with Jarvis; markdown rendering |
| **reepedit** (right, optional) | In-TUI editor for files Jarvis or you open |
| **Shell** (right/below, optional) | Interactive zsh login shell (`zsh -l`) |

Use **Tab** to cycle focus: Chat → Input → Editor → Terminal → Chat.

Press **Esc** to return to the Main Menu.

---

## How Jarvis knows your system

On every message you send, ReepCore builds a **context bundle** and injects it into the Ollama system prompt. Jarvis sees (summarized, size-capped):

- **System state** — kernel, installed kernels, generation, UUIDs, theme/wallpaper, enabled profiles, failed systemd units, bootloader info
- **TUI state** — current view, Main Menu catalog (keys 1–11), live CPU/RAM/load/network, top processes, package overview, pending official/flatpak updates (same panels as the home screen)
- **Sub-view state** — snapshots for loaded screens: Module Manager toggles, profile enablement, installed-package tabs/filters/selection, Intent Studio graph summary, theme selection, ReepCore Commands focus + embedded terminal output, App Launcher filter/selection, manual topic, File Browser tree/preview, Real Time Monitoring submenu, Network Dashboard (ntopng panels + triaged alerts), Security Monitoring (service actions + embedded terminal), Security Triage state (when enabled)
- **ReepCore repo** — repo root, `REEPCORE/tools/` index, tool README snippets, `dotfiles/config/` index
- **Runtime config** — `~/.config/reepcore/reepcore.yaml`, enabled profile YAMLs, generation manifests
- **File Browser** — current directory listing and selected file (content included when selected)
- **reepedit** — live buffer of the open file (authoritative over disk if modified). Large files may be **windowed** around the editor scroll position; Rust `.rs` files also get a **symbol outline** (`fn` / `struct` / `impl` / …) from the full buffer before windowing.
- **File tree (right panel)** — visible home-directory tree, highlighted selection, preview pane text, bookmarks (Ctrl+B toggles; Tab to focus tree)
- **Terminal** — recent output from the embedded shell pane when it is open
- **RAG (optional)** — when your message matches the code-context decider, relevant snippets from the indexed project are appended to the **user** prompt as `## RAG context (local codebase)` (not the system prompt). Skipped for issues-only turns, diff-only turns, and typical `@reepedit` single-file reviews when the buffer is already open

Repo root is resolved from, in order:

1. `$REEPCORE_REPO` (must contain a `REEPCORE/` directory)
2. Path inferred from the `reepcore` binary location
3. Common paths such as `~/ReepCore`

Jarvis is instructed to treat **`dotfiles/config/`** and **`REEPCORE/tools/`** as the source of truth for edits, not runtime copies under `~/.config` unless you explicitly ask otherwise.

---

## Chat input

Type in the input bar at the bottom and press **Enter** to send. Long prompts **wrap** inside the input panel; the panel **grows** (up to a cap) so text stays visible instead of scrolling off the right edge. **Ctrl+U** clears the input when the input pane is focused.

| Input | Effect |
|-------|--------|
| Plain question | Sent to Jarvis with full context |
| `@search <query>` | Web search (also `@web`, `@google`) — results injected before Jarvis replies |
| `@run <command>` | Opens the **shell pane** and runs `<command>` (also `@terminal`, `@exec`). Use `@run btop` — not `try @run` in a long chat sentence. Short lead-ins work: `try @run btop`, `please @run ls` |
| `@open <n\|name>` | Opens a TUI screen immediately (also `@navigate`, `@nav`, `@goto`) — e.g. `@open 5`, `@open installed packages` |
| `@launch <app>` | Launches a desktop app via `reep-launcher` (also `@app`, natural `launch Steam`, voice `at launch steam`) — no LLM round-trip |
| `@close <proc>` | Closes a running program via `pkill` (also `@kill`, natural `close btop`, voice `be top` / `p kill b top`) — no LLM round-trip |
| `@home` / `@menu` | Return to Main Menu (also natural `go to main menu`) |
| `go to <screen>` | Natural navigation — e.g. `go to theme manager`, `open file browser` |
| `@tree <path>` | Change the docked file tree root (also `@cd`) — e.g. `@tree ~/ReepCore` |
| `@reindex` | Rebuild local RAG index (also `@rag reindex`, `@rag index`) — no LLM call |
| `@doctor` / `@validate doctor` | Run `reepcore doctor` (agent tool; no LLM when agent off) |
| `open selected` / `review selected file` | Opens the highlighted tree row in reepedit |
| `@reepedit <path>` | Opens `<path>` in reepedit |
| `@/path/to/file` | Same as `@reepedit` for absolute/tilde/relative paths |
| “open …”, “review …”, “read …” | Resolves paths under home, repo, dotfiles, and tools; opens matching files in reepedit |

Append **`--explain`** to a prompt when you want a longer, more detailed answer (Jarvis stays concise by default).

While Jarvis is streaming a reply, press **Esc** to cancel.

When the model emits reasoning (thinking-capable models with `think: true` in `ai.yaml` or `OLLAMA_THINK=1`), reasoning is stored per assistant message and rendered as a **collapsible inline box** in chat (hidden by default). **Ctrl+Y** (or **F2** when reepedit is closed) toggles the latest message's reasoning box; click the **▸ Thinking** header to expand/collapse. Set **`JARVIS_THINKING_LEGACY_PANEL=1`** to restore the old fixed-height panel between the header and chat (**j/k** scroll when that panel is open). Disable reasoning for review/diff turns with **`OLLAMA_THINK=0`** when launching reepcore (not via chat text).

### Review loops & auto-recovery

Thinking models sometimes **self-debate** a review without ever writing a chat reply — flip-flopping between “this is a bug” and “this is correct” on the same lines, or restarting the analysis over and over. ReepCore handles this in layers:

- **Loop cutoff** — the repetition guard also aborts *verdict flip-flop* spirals (repeatedly reversing a conclusion, or piling up “wait…” / “actually…” restarts), not just repeated paragraphs.
- **Auto-finalize** — when a **review** turn’s reasoning loops or stalls, Jarvis fires **one** automatic continuation that asks for the final review only: one bullet per issue (line ref + one sentence + a verdict — *bug / not a bug / needs check*), with no re-analysis. This replaces the old behavior of dumping the raw reasoning trace into chat.
- **Cleaner fallback** — if reasoning must still be salvaged directly, deliberation lines (“Wait…”, “Actually…”, “Maybe the intention…”, anything ending in “?”) are dropped so only findings remain.
- **Reasoning-trim** — when a **review** turn *does* produce a chat reply but leaks a chain-of-thought preamble into it (“Thinking Process:”, “Analyze the request…”, “Let me analyze…”), the preamble is stripped and only the findings (bullets or a `Findings:` / `### Review` section) are kept. A clean reply is never modified.
- **Large-file chunking** — for a **review-only** turn on a big open file (default > 500 lines), the reepedit buffer is injected **windowed** (head + tail + Rust symbol outline) instead of dumped whole, with omitted ranges marked. This keeps weaker local models from free-associating over 1000+ lines. **Edit / diff** turns still get the full buffer so hunks match the exact text. Tune the threshold with `JARVIS_REVIEW_CHUNK_MAX_LINES` (`0` = always full).
- **JetBrains MCP availability** — *"do you see JetBrains MCP?"* gets an **instant** yes/no from reachability (no LLM). Short status questions also use a tighter thinking budget (`JARVIS_SIMPLE_THINK_MAX_BYTES`). Salvage rejects instruction-parroting ("Analyze the Request…") from loop dumps.

Review turns are also instructed up front to **commit to a verdict and not reconsider a conclusion more than once**. A byte budget caps runaway review reasoning (see `JARVIS_REVIEW_THINK_MAX_BYTES`); it is scoped to review-like prompts so ordinary long debugging reasoning is never cut short. Disable it with `JARVIS_REVIEW_THINK_BUDGET=0`.

### TUI navigation (Phase 3)

Navigation commands run **without** sending a message to the model — the TUI switches view immediately.

| Main menu key | Screen |
|---------------|--------|
| 1 | Theme Manager |
| 2 | Jarvis |
| 3 | Module Manager |
| 4 | File Browser |
| 5 | Installed Packages |
| 6 | Manage Profiles |
| 7 | ReepCore Commands |
| 8 | Real Time Monitoring |
| 9 | App Launcher |
| 10 | Intent Studio |
| 11 | ReepCore Manual |

**Real Time Monitoring submenu** (Main Menu → 8):

| Key | Screen |
|-----|--------|
| 1 | Network Dashboard (live ntopng panels + triaged alerts) |
| 2 | Security Monitoring (IDS, fail2ban, ntopng services) |
| 3 | Back → Main Menu |

Natural-language navigation also works without LLM round-trips: `@open network dashboard`, `@navigate security monitoring`, `go to real time monitoring`.

Jarvis can also drive navigation: if a reply ends with a lone line like `@navigate 5` or a fenced `navigate` block, ReepCore opens that screen when the reply finishes.

---

## Running shell commands

### Agent mode (Cursor-style inline commands)

Set **`JARVIS_AGENT=1`** (e.g. in `~/.zshenv`) to enable agent mode by default at startup. Toggle at runtime in the footer (**Agent** / **Chat**, click or **Ctrl+M**) without restarting.

1. Jarvis replies with a ` ```bash ` block.
2. ReepCore runs it via `sh -c`, captures stdout/stderr, and shows a **`run:`** block inline in chat (like Cursor Agent).
3. Jarvis automatically continues the turn with the output — no Ctrl+R, no “send again”.
4. Repeats until Jarvis answers without a shell block or **`JARVIS_AGENT_MAX_STEPS`** (default **8**) is reached.

**Confirmation:** If Jarvis ends with a question ("Want me to…?", "Want to…?", etc.), ReepCore **does not** auto-run any ```bash block in that reply — answer yes/no or give instructions first. Placeholder paths like `/path/to/wallpaper.jpg` and bare directory paths are also skipped.

**`@run <cmd>`** and **Ctrl+R** also use captured inline runs when agent mode is on (instead of the shell pane).

- **Auto-run** (after Jarvis replies): captured inline **`run:`** blocks — safe commands only.
- **Ctrl+R** / blocked **`@run`**: always opens the **interactive shell pane** (Ctrl+T) so `sudo`, `pacman -S`, etc. work — type your password there.
- **`@run`** with a safe command: captured inline run when agent mode is on.

| Variable | Default | Purpose |
|----------|---------|---------|
| `JARVIS_AGENT` | off | `1` / `true` enables agent mode |
| `JARVIS_AGENT_MAX_STEPS` | `8` | Max command runs per user message |
| `JARVIS_AGENT_CMD_TIMEOUT` | `120` | Seconds before killing a hung command |
| `JARVIS_AGENT_OUTPUT_MAX_BYTES` | `16384` | Cap on captured stdout per command |
| `JARVIS_AGENT_MAX_LINES` | `8` | Max lines in an auto-run shell block |

Still blocked in agent mode: `sudo`, `rm`, package installs, file writes (`>`), `curl`, `git push`, etc.

### Structured agent tools (Phase 1)

When agent mode is on, Jarvis can invoke **structured tools** (safer than widening the bash allowlist):

| Tool | Fence | Input shortcut | Runs |
|------|-------|----------------|------|
| **test** | ` ```tool:test` | `@test cargo check -p reepcore` | `cargo test/check/clippy/build` (repo-scoped); with reepedit open also `python3 -m py_compile …`, `python3 -c '…'` (cwd = editor dir) |
| **validate** | ` ```tool:validate` | `@validate doctor` / `@doctor` | `reepcore doctor`, `reepcore intent validate <path>` |
| **search** | ` ```tool:search` | `@search rg …` | `rg`, `fd`, `git grep` |
| **git** | ` ```tool:git` | `@git status` | `git status/diff/log/show` (read-only) |
| **jetbrains** | ` ```tool:jetbrains` | `@jetbrains {"tool":"open_files"}` | RustRover MCP read-only tools (when IDE MCP is reachable) |

Chat blocks are labeled **`test:`**, **`validate:`**, **`search:`**, **`git:`**, **`jetbrains:`** (or **`run:`** for plain bash). Test output includes a pass/fail **summary** line.

| Variable | Default | Purpose |
|----------|---------|---------|
| `JARVIS_AGENT_TOOLS` | `test,validate,search,git` | Comma list of enabled structured tools; add `jetbrains` for IDE MCP |
| `JARVIS_JETBRAINS_MCP_URL` | unset | Full MCP endpoint from RustRover **Settings → Tools → MCP Server** (HTTP Stream or SSE URL) |
| `JARVIS_JETBRAINS_PROJECT_PATH` | auto (`~/ReepCore`) | Override project root sent as `projectPath` on every IDE tool call |
| `JARVIS_AGENT_AUTO_CHECK` | off | After Ctrl+A patch apply, auto-run `cargo check -p reepcore` in sandbox/live cwd |
| `JARVIS_AGENT_AUTO_CHECK_LIVE` | off | After `@promote`, auto-run live `cargo check` |
| `JARVIS_AGENT_TOOL_CONTINUE` | off | Manual `@test` / `@validate` also auto-continues the agent LLM turn |

### JetBrains / RustRover MCP (read-only)

When RustRover is open on ReepCore with MCP enabled, Jarvis can call **read-only** IDE tools over localhost (no proxy container). Jarvis uses friendly JSON aliases and remaps them to the parameter names RustRover expects before each MCP call.

**RustRover setup (one-time):**

1. Open `~/ReepCore` in RustRover.
2. **Settings → Tools → MCP Server** → Enable MCP Server (note the port, e.g. `64342`).
3. **Settings → Build, Execution, Deployment → Debugger** → enable **Can accept external connections**.
4. **Settings → Tools → MCP Server → Exposed Tools** — enable these read-only tools (leave write/execute tools off unless you intend to grant them):

   | Toolset | Tool | Jarvis alias |
   |---------|------|--------------|
   | AnalysisToolset | `get_file_problems` | `diagnostics` |
   | CodeInsightToolset | `get_symbol_info` | `rust_resolve` (fallback backend) |
   | FileToolset | `get_all_open_file_paths` | `open_files` |
   | SearchToolset | `search_symbol` | `search_symbol` |
   | TextToolset | `get_file_text_by_path` | `read_file` |

   `rust_internal_multiresolve` is **not** present in current RustRover builds. Jarvis tries it first for `rust_resolve`, then falls back to `get_symbol_info` (Quick Documentation at a line/column).

   Safe to leave disabled: `apply_patch`, `replace_text_in_file`, `create_new_file`, `execute_terminal_command`, `execute_run_configuration`, database/VCS tools.

5. Copy the **HTTP Stream Config** or **SSE Config** URL into `JARVIS_JETBRAINS_MCP_URL`. Jarvis needs the plain URL only (not the full JSON block Cursor uses).
6. Add `jetbrains` to `JARVIS_AGENT_TOOLS`.

Example `~/.zshenv`:

```bash
export JARVIS_JETBRAINS_MCP_URL='http://127.0.0.1:64342/stream'   # paste from IDE
export JARVIS_AGENT_TOOLS='test,validate,search,git,jetbrains'
```

**Invocation (agent mode):**

````markdown
```tool:jetbrains
{"tool":"diagnostics","pathInProject":"REEPCORE/src/jarvis_tools.rs"}
```
````

Or from the input bar (works with Agent off):

```text
@jetbrains {"tool":"open_files"}
```

**Tool reference:**

| Jarvis alias | MCP tool(s) | Purpose |
|--------------|-------------|---------|
| `open_files` | `get_all_open_file_paths` | Files currently open in the IDE |
| `read_file` | `get_file_text_by_path` | File text via IDE index |
| `diagnostics` | `get_file_problems` | Compiler/inspection problems for a file |
| `search_symbol` | `search_symbol` | Semantic symbol search |
| `rust_resolve` | `rust_internal_multiresolve` → `get_symbol_info` | Symbol docs at line/column (fallback when multiresolve unavailable) |

**Jarvis JSON → MCP parameters** (remapped automatically; use either form):

| Jarvis field | MCP field | Used by |
|--------------|-----------|---------|
| `pathInProject` / `path` | `filePath` | `diagnostics`, `rust_resolve` |
| `pathInProject` | `pathInProject` | `read_file` |
| `query` | `q` | `search_symbol` |
| `projectPath` | `projectPath` | all tools (defaults to `~/ReepCore`) |
| `line`, `column` | `line`, `column` | `rust_resolve` (1-based) |

Omit `pathInProject` on `diagnostics` / `read_file` / `rust_resolve` to default to the file open in reepedit. **Sandbox caveat:** RustRover indexes the **live** project — sandbox worktree edits may not appear in IDE analysis until saved and visible to the IDE.

**Manual smoke tests** (RustRover open, reepcore rebuilt):

```text
@jetbrains {"tool":"open_files"}
@jetbrains {"tool":"read_file","pathInProject":"REEPCORE/src/backup.rs"}
@jetbrains {"tool":"diagnostics","pathInProject":"REEPCORE/src/jarvis_jetbrains.rs"}
@jetbrains {"tool":"search_symbol","query":"JetbrainsClient"}
@jetbrains {"tool":"rust_resolve","pathInProject":"REEPCORE/src/jarvis_jetbrains.rs","line":94,"column":12}
```

Cargo live test (requires IDE on localhost):

```bash
JARVIS_JETBRAINS_MCP_URL='http://127.0.0.1:64342/stream' \
JARVIS_AGENT_TOOLS='jetbrains' \
cargo test live_mcp_open_files -- --ignored --nocapture
```

When RustRover is closed or MCP is unreachable, Jarvis omits IDE tools from the system prompt and returns a clear error if the model still emits a `tool:jetbrains` fence.

### Sandbox mode (Phase 2–3 edit loop)

Isolated **git worktree** per Jarvis session under `~/.local/state/reepcore/jarvis/sandbox/<session-id>/reepcore/`. When sandbox is on, agent commands, tools, and **reepedit** target the worktree — live dotfiles are untouched until you **promote**.

| Control | Action |
|---------|--------|
| Footer **Sandbox** / **Live** | Click to toggle (or **Ctrl+Shift+S**) |
| Footer **`N changed`** | Click to open promote picker |
| `JARVIS_AGENT_SANDBOX=1` | Default sandbox on at startup |
| `@promote` | List changed files (status bar) |
| `@promote <path>` | Copy one sandbox file to the live repo |
| `@promote all` | Promote all changed files |
| **Ctrl+Shift+P** | Promote picker — j/k select, Enter one file, `a` all (confirm twice) |
| `JARVIS_AGENT_BWRAP=1` | Optional bubblewrap layer (`bwrap` must be installed); falls back to cwd guard |

**Edit loop (recommended):** enable Agent + Sandbox → Jarvis suggests a diff → **Ctrl+A** apply in reepedit (`[SANDBOX]` in title) → **Ctrl+S** save to worktree → `@test cargo check -p reepcore` or let `JARVIS_AGENT_AUTO_CHECK=1` run after Ctrl+A → `@promote` or picker when green → review live file, **Ctrl+S**.

Manual `@test` / `@validate` from the input bar runs the tool **without** an LLM follow-up (unless `JARVIS_AGENT_TOOL_CONTINUE=1`).

| Variable | Default | Purpose |
|----------|---------|---------|
| `JARVIS_AGENT_AUTO_CHECK` | off | After Ctrl+A patch apply, auto-run `cargo check -p reepcore` (agent step) |
| `JARVIS_AGENT_AUTO_CHECK_LIVE` | off | After promote, auto-run live `cargo check` |
| `JARVIS_AGENT_TOOL_CONTINUE` | off | Manual `@test` also triggers agent LLM continuation |

In chat, command output appears once under **`run:`** (command, `exit N · Xms`, stdout). Internal model continuations show as a one-line **`agent: continuing…`**. When Jarvis stops without another shell block, the status bar shows **`Agent finished — N of M step(s) used`** (8 is the max per message, not a target).

When agent mode is **off**, use the shell pane workflow below.

### Auto-run (read-only discovery)

When Jarvis’s reply ends with a fenced shell block that only contains safe discovery commands, ReepCore **runs it automatically** in the shell pane — no Ctrl+R:

- `find`, `fd`, `locate`, `ls`, `grep`, `rg`, `which`, `stat`, `cat`, `head`, `tail`, etc.
- Up to 3 lines; no pipes, redirects (except `2>/dev/null`), `sudo`, `-exec`, `-delete`, or destructive tools

Status: *“Discovery command auto-running in shell…”*. Send another message afterward so Jarvis sees the terminal output in context.

### Manual run (everything else)

When Jarvis suggests a script that is not auto-safe:

1. Press **Ctrl+R** to send the last script block to the shell pane, **or**
2. Copy the command and type it yourself in the shell pane.

The status line shows *“Reply ready — Ctrl+R run shell block in terminal pane”* when a runnable block is available.

### Direct run

- **Ctrl+T** — toggle the shell pane (interactive zsh)
- Type commands directly in the shell when the terminal pane is focused
- **`@run pacman -Q reepcore`** — run a one-liner from the input bar without waiting for a fenced block. Interactive TUIs (`btop`, `htop`, `vim`, …) always open the **PTY shell pane**, even in Agent mode — not a captured inline `run:` block
- **`@shell <command>` (Jarvis-emitted)** — when you ask Jarvis to **install** a package or run an interactive/sudo command, it ends its reply with a single line `@shell sudo pacman -S htop` (or `@shell paru -S …` for AUR). ReepCore opens the shell pane and runs it; type your sudo password in the pane. This is how Jarvis does one-shot installs ("install htop") — as opposed to "add htop to my profile", which edits the profile YAML as a ```diff. `@run` / `@exec` / `@terminal <cmd>` in a reply are accepted as aliases. **Routing:** `@shell` for a *read-only/safe* command (e.g. `pgrep -f rustrover`) is routed — in Agent mode — to a captured **`run:`** block instead of the live pane, so Jarvis auto-continues with a one-line summary. ReepCore rewrites `pgrep -fa`/`-a` to `pgrep -f` (PIDs only), auto-summarizes pgrep output, and strips Java classpaths from chat. The interactive PTY pane is reserved for `sudo`/installs/TUIs. For read-only checks Jarvis prefers a plain ```bash block (`pgrep -f …`, `pgrep -fc …`, `systemctl is-active …`) or `@jetbrains {"tool":"open_files"}` for MCP reachability.

### Shell pane details

- Spawns **`zsh -l`** (login shell, loads your `.zshrc`)
- Full PTY: tab completion, ncurses, sudo prompts (type your password in the pane)
- **Interactive TUIs** (`btop`, `htop`, `vim`, …) render in the shell pane at the pane’s real size (not a fixed 80×24). Use **`@run btop`** or **Ctrl+T** then type the command
- **Powerline / Starship / p10k prompts** — colored segments and Nerd Font icons render when your host terminal uses a Nerd Font (same as Kitty). The pane sets `REEPCORE_TUI_PTY=1` if you want to branch prompt config in `.zshrc`.
- **Ctrl+T** again closes the pane
- **Ctrl+W** closes the terminal pane when terminal focus is active

Jarvis receives terminal output in context on the next turn, so it can reason about command results you actually ran.

---

## reepedit and patch preview

### Opening files

- Ask Jarvis to review a file (it opens in reepedit instead of dumping the whole file in chat)
- **`@reepedit REEPCORE/src/tui.rs`**
- **Ctrl+E** — open the File Browser’s selected file in reepedit (File Browser must be navigated beforehand; Jarvis shares the same browser state)

### Creating files

Ask Jarvis to **create** a config, profile, or module and the generated file opens directly in reepedit as an unsaved draft — the YAML is not dumped into chat. Press **Ctrl+S** to save it (the parent directory is created automatically).

- “Create an NVIDIA GPU **module**” → `…/templates/reepcore-setup/modules/nvidia-gpu.yaml`
- “Make a **profile** called gaming” → `~/.config/reepcore/profiles/gaming.yaml`
- “Create a **config** file” → `~/.config/reepcore/reepcore.yaml`

The filename is inferred from the request (e.g. “called X”, “named X”, or the words before *module*/*profile*/*config*). Review the path shown in the status line before saving; nothing is written to disk until you press **Ctrl+S**.

### Applying Jarvis’s edits

When a file is open in reepedit and Jarvis’s last reply contains a unified diff:

1. **Ctrl+A** — open patch preview (diff overlay)
2. Review blocks — **a** accept / **r** reject per block (ReepCore **lints** patches: `cat >`/heredoc/sed -i anywhere in the reply (including ```bash fences), shell in added diff lines, shell tails after diff (`git apply`, `py_compile`), orphan `else:`, **dangling indented `main()` after removing `if __name__`**, Python `ast.parse` syntax check, `deck` used but not in scope, duplicate `@@` hunks, bulk blank-line removals, duplicate `-` lines, half-fixes like `Deck` + leftover `random.randint`, …). On first open, lint runs against the **full** patch so you see all problems upfront. After you **a**/**r** any block, lint **recomputes against accepted blocks only** — reject a broken hunk and the footer can clear from BLOCKED so **Y** applies the good blocks.
3. **Y** — apply accepted blocks into the editor buffer (blocked when lint reports errors on the accepted set)
4. **Ctrl+S** or **F2** — save to disk
5. **N** or **Esc** — reject the preview

**Apply-before-test:** When Jarvis proposes a patch, `@test` and agent test commands (`python3 …`, `cargo check`, …) are **blocked** until you **Ctrl+A → Y apply → Ctrl+S save**. Tests read files on disk; running before apply produces misleading failures. The gate also accepts **saved buffer changes** — if you edited in reepedit and **Ctrl+S** saved (even without pressing **Y**, or after sending a follow-up like “test it”), `@test` is allowed when the live buffer differs from the snapshot taken when Jarvis offered the patch. Re-echoed ```diff blocks in follow-up replies are ignored once the saved buffer already reflects your edits.

**Python smoke tests (reepedit open):** With a `.py` file open in reepedit, `@test` runs in that file's directory:

```text
@test python3 -m py_compile wakeup.py
@test python3 -c "import py_compile; py_compile.compile('wakeup.py')"
```

Agent ```tool:test``` fences use the same commands and inherit the reepedit cwd when the editor is open.

**Natural syntax check:** With a `.py` file open in reepedit and Agent mode on, phrases like *syntax check*, *smoke test*, or *check if it compiles* map to `@test python3 -m py_compile <open file>` (same as typing the `@test` line explicitly).

**Shell pane safety:** Ctrl+R, `@run`, and auto-discovery shell blocks **block** `cat >`, heredocs (`<< EOF`), `sed -i`, `tee`, and file redirects — use reepedit + **Ctrl+A** for edits instead.

**Diff-only turns:** When your prompt asks for a **diff** (`one small diff`, `create diff`, `make diff`, …), ReepCore injects a diff-only instruction into the model prompt, blocks **all** agent shell/tools for that turn (apply with **Ctrl+A**, not `git apply` / `sed`), and skips auto-running bash blocks after the reply. Patch preview lint also errors on shell tails (`git apply`, `py_compile`, ` ```bash` after the diff) and on runaway diffs (bulk blank-line removals, duplicate `-` lines).

**Issues-only / no-diff turns:** When your prompt asks for **issues only** or **no diff** (`list issues only`, `no diff blocks`, `no code fences`, `prose only`, …), ReepCore injects a bullets-only instruction, **skips RAG retrieve** (reepedit buffer is enough), does **not** track patch preview, and **blocks Ctrl+A** for that turn. If the model still emits a diff, the status line warns and preview stays disabled. Ask separately when you want a patch (`one small diff …`).

**Duplicated-function claims:** Prompts mentioning duplicated/duplicate function definitions inject a `def` line inventory from the reepedit buffer and require Jarvis to cite those line numbers before proposing a diff; if no two `def` names match, it should reply with no diff.

**Python edit guidance:** When a `.py` file is open in reepedit and you ask for an edit (review/edit/fix/clean-up/refactor phrasing, not an issues-only/no-diff turn), ReepCore injects **Python edit rules** into the prompt so Jarvis keeps valid indentation and block structure — the same things the patch-preview lint blocks. It detects the file's indent unit (e.g. *4 spaces*, *2 spaces*, *tabs*) and tells the model to match it exactly, make a small one-concern diff (no whole-file rewrites), copy context lines verbatim, never delete blank lines, never leave an empty block after `if`/`else`/`for`/`def` (use `pass`), and not strip an `if __name__` guard while leaving its body indented. This reduces the "Python syntax error (line N): expected an indented block" patches that get blocked at apply time.

**Blank-line auto-repair:** Before a diff is shown in patch preview, ReepCore converts any blank-line *removals* (`-` on an empty line) into context lines so the blank lines are kept instead of deleted. A diff that previously got blocked solely for "removes N blank lines — likely model loop" now applies cleanly; the preview summary notes `kept N blank line(s)`. Hunk `@@` headers are recomputed so the diff stays well-formed. Deletion-only diffs up to **64** removed lines are allowed (e.g. removing one duplicate function); above that still triggers "too large".

If the diff only partially matches, Jarvis shows a warning; **Y** applies a best-effort merge when lint passes.

**Ctrl+W** closes reepedit (with unsaved-changes protection).

---

## Keyboard reference

Global shortcuts work when not streaming and when patch preview is closed (unless noted).

| Key | Action |
|-----|--------|
| **Tab** | Cycle focus: Chat → Input → Editor → Terminal |
| **Enter** | Send message (when input focused) |
| **Ctrl+U** | Clear input (when input focused) |
| **j/k**, **PgUp/PgDn** | Scroll chat (chat focused) |
| **Ctrl+M** | Toggle **Agent** / **Chat** mode (footer, right of model name) |
| **Ctrl+Shift+L** | Ollama **chat model** picker (footer `Model:`) |
| **Ctrl+Shift+E** | Ollama **embedding model** picker (footer `embed:`; RAG must be on) |
| **Ctrl+Shift+G** | Toggle **RAG / embeddings** on or off (footer shows `embed: off` when disabled) |
| **Ctrl+Shift+S** | Toggle **Sandbox** / **Live** (agent commands in isolated worktree) |
| **Ctrl+T** | Toggle shell pane |
| **Ctrl+R** | Run last ```shell block from Jarvis reply |
| **Ctrl+N** | New Jarvis session |
| **Ctrl+O** | Sessions picker |
| **Ctrl+E** | Open File Browser selection in reepedit |
| **Ctrl+A** | Preview diff from last reply (reepedit open) |
| **Ctrl+S** / **F2** | Save reepedit buffer |
| **Ctrl+W** | Close terminal (terminal focus) or reepedit |
| **Ctrl+Y** / **F2** | Toggle inline **Thinking** box on latest assistant message (or legacy thinking panel when `JARVIS_THINKING_LEGACY_PANEL=1`; F2 only without reepedit) |
| **Esc** | Cancel streaming reply |
| **Esc** | Exit Jarvis → Main Menu |

When the **terminal pane** is focused, keystrokes go to zsh except Jarvis global keys (Tab, Ctrl+M/T/R/W/E/A/S, Esc, F2).

**Mouse wheel** over the shell pane scrolls terminal scrollback (up = older output, down = back toward live). Typing or sending a command snaps back to the live bottom.

**Select & copy from chat, thinking, shell, or reepedit:** click-and-drag with the left mouse button over the Chat pane, an expanded **Thinking** inline box (or legacy thinking panel when enabled), the **Shell — interactive** pane, or the reepedit editor pane to highlight text; releasing the button copies the selection to the system clipboard. In reepedit, drag over the file content (not the line-number gutter). Copying uses `wl-copy` (Wayland), then `xclip`/`xsel` (X11), falling back to an OSC 52 terminal escape (works over SSH). A status line confirms how many characters were copied. Click once (without dragging) to clear the selection.

Jarvis receives a **live shell snapshot** on every message you send (`## Embedded terminal pane` in its context — last ~200 lines plus the visible screen, capped at 12 KiB). That includes output from **Ctrl+R**, **`@run`**, auto-run discovery scripts, and commands you **type in the interactive shell**. Ask “check shell output” after a command finishes — Jarvis should summarize from that section, not claim it lacks access.

---

## Voice I/O (optional)

CPU-only voice is opt-in via **`JARVIS_VOICE=1`**. Missing mic, models, or binaries disable voice silently — text chat keeps working.

| Half | Behavior |
|------|----------|
| **STT** | While the Jarvis view is open, the mic listens continuously. Speak **"Jarvis, …"** as one phrase; whisper.cpp transcribes after you pause. Text lands in the input bar (press **Enter** to send, or set `JARVIS_VOICE_AUTO_SEND=1`). |
| **TTS** | At **turn completion only** — not during streaming — Jarvis asks Ollama for a 1–2 sentence spoken summary, then **piper-tts** reads it aloud. Full markdown detail stays in chat. |

Voice is paused while Jarvis streams a reply or speaks a summary. The footer shows **`voice: listen`** / **`rec`** / **`stt`** / **`speak`** when active.

### Setup (one-time)

**Piper TTS** — you already have `en_GB-alan-medium.onnx` in `~/`. Optional tidy path:

```bash
mkdir -p ~/.local/share/reepcore/jarvis/voice
mv ~/en_GB-alan-medium.onnx ~/en_GB-alan-medium.onnx.json \
   ~/.local/share/reepcore/jarvis/voice/
```

Smoke test:

```bash
echo "Jarvis online." | piper-tts -m ~/en_GB-alan-medium.onnx -f /tmp/jarvis.wav -q
pw-play /tmp/jarvis.wav
```

**Whisper STT** — download a ggml model (~140 MB):

```bash
mkdir -p ~/.local/share/reepcore/jarvis/voice
wget -O ~/.local/share/reepcore/jarvis/voice/ggml-base.en.bin \
  https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin
```

Build or point to **whisper-cli** (default: `~/whisper.cpp/build/bin/whisper-cli`).

**Enable in `~/.zshenv`:**

```bash
export JARVIS_VOICE=1
export JARVIS_PIPER_VOICE="$HOME/en_GB-alan-medium.onnx"
export JARVIS_WHISPER_BIN="$HOME/whisper.cpp/build/bin/whisper-cli"
```

Say **"Jarvis, what kernels are installed"** in the Jarvis view — transcribed text appears in the input bar.

---

## Configuration

### AI model manifest

Jarvis reads `~/.config/reepcore/ai.yaml` for named model aliases. `reepcore init` **symlinks** the shipped template (`REEPCORE/templates/reepcore-setup/ai.yaml`) when the file is missing; `reepcore init --sync-profiles` adds it when absent.

Each **role** (`chat`, `coding`, `review`, …) resolves to a **yaml alias key** under `models:` (e.g. role `chat` → alias `chat` → `models.chat.name`). The `defaults:` block maps roles to alias keys — the shipped template uses 1:1 names (`chat: chat`, `review: review`, …).

**Resolution order** (per role): TUI session picker → env var → `ai.yaml` alias → built-in fallback.

| Role | Env override | Default alias | Built-in fallback |
|------|-------------|---------------|-------------------|
| `chat` | `OLLAMA_MODEL` | `defaults.chat` | `my-agent:latest` |
| `coding` | — | `defaults.coding` | same as `chat` |
| `review` | — | `defaults.review` | `qwen3.5:4b` |
| `embed` | `JARVIS_RAG_EMBED_MODEL` | `defaults.embed` | `qwen3-embedding:4b` |
| `planner` | `JARVIS_PLANNER_MODEL` | `defaults.planner` | same as `chat` |
| `friday` | — | `defaults.friday` | same as `planner` (orchestrator peer review) |
| `security` | `JARVIS_SECURITY_MODEL` | `defaults.security` | `qwen3.5:4b` (CPU — ntopng alert triage) |

```yaml
defaults:
  chat: chat
  coding: coding
  review: review
  embed: embed
  planner: planner
  friday: friday
  security: security
```

**Per-turn auto-routing:** Each user message is classified (review/edit vs conversational). Jarvis picks the yaml role automatically — `review` for code-review/audit turns (open reepedit or path hints), `coding` for diff/fix/edit/run-compile turns, `chat` for hello/profiles/navigation, `planner` for **design checklists** (`create a plan`, `@plan …`) and orchestrator JSON, and per-agent models during orchestrator dispatch (`code_review` → `review`, `test_runner` → `coding`, `peer_review` → `friday`, `synthesize` → `chat`). If the chat model says it will fix or implement code (`let me fix…`, `let me implement these features…`, `I'll implement…`, etc.), Jarvis **switches to the coding model mid-turn** and continues with a minimal ```diff```. Run/compile prompts with an open code file also start on `coding`. The footer shows the **active** model for the turn: `Model: review (review-agent:latest) · think`. The model picker sets your **preferred chat** model only; the next message re-classifies.

| Task | Model role |
|------|------------|
| Conversational / profiles / navigate | `chat` |
| **Design plan / checklist** (`create a plan`, `@plan`) | `planner` |
| Reepedit review / audit / issues-only | `review` |
| Diff / fix / edit / run / compile | `coding` |
| Orchestrator planner JSON | `planner` |
| Orch `code_review` | `review` |
| Orch `test_runner` | `coding` |
| Orch `peer_review` (Friday) | `friday` |
| Orch `synthesize` | `chat` |
| ntopng alert triage (background) | `security` |

### Design planning mode

Ask for a **markdown checklist plan** (not code, not orchestrator dispatch). No `JARVIS_ORCHESTRATOR` required.

**Triggers:** phrase contains `create a plan`, `make a roadmap`, `create a checklist`, … or prefix **`@plan`**.

**Example:**

```text
@plan add a way to make a thinking model and a reasoning model work together
```

or

```text
Add dual-model routing to Jarvis — create a plan
```

**Behavior:** footer switches to **`Model: planner (…)`**; reply is markdown with **Goal**, **Checklist** (`- [ ]` items), **Files touched**, **Risks**. Steps are **not** auto-executed — follow up with `implement step 1` or `one small diff for step 2` on the **coding** model.

**Not planning mode:** `review blackjack.rs` with orchestrator on (uses multi-agent JSON planner instead). `@orchestrate` always wins over `@plan`.

### Recommended layout (12 GB GPU)

On a **12 GB** GPU, use **Modelfiles over shared base pulls** — not separate full-size models per role. The shipped template [`templates/reepcore-setup/ai.yaml`](../templates/reepcore-setup/ai.yaml) defines seven roles; five run on GPU (`11434`), two on CPU (`11435`):

| Role | Alias | Ollama name | Base | Host | `think` | Modelfile notes |
|------|-------|-------------|------|------|---------|-----------------|
| `chat` | `chat` | `my-agent:latest` | `qwen3.5:9b` | GPU | `true` | User-maintained Modelfile (not templated) |
| `planner` | `planner` | `planner-agent:latest` | `qwen3.5:9b` | GPU | `true` | `num_ctx 16384` — orchestrator JSON + `@plan` |
| `review` | `review` | `review-agent:latest` | `qwen3.5:4b` | GPU | `true` | `num_ctx 16384` — read-only audits |
| `coding` | `coding` | `coding-agent:latest` | `qwen2.5-coder:14b` | GPU | `false` | `num_ctx 16384` — writes/diffs only |
| `friday` | `friday` | `friday-agent:latest` | `qwen3.5:2b` | GPU | `false` | `num_ctx 8192` — orchestrator peer review |
| `security` | `security` | `security-agent:latest` | `qwen3.5:4b` | CPU | `false` | `num_ctx 8192` — ntopng alert triage |
| `embed` | `embed` | `qwen3-embedding:0.6b` | *(registry)* | CPU | — | `auto_pull: true` — RAG embeddings |

**Avoid** `deepseek-coder-v2:16b` without a custom Modelfile (`num_ctx`) — Jarvis's injected context exceeds the default window and returns HTTP 400. **Avoid** pulling a second full `qwen3.5:9b` per role — reuse Modelfiles on the same base; role switches still reload weights unless `OLLAMA_KEEP_ALIVE` is set (5–15s disk reload otherwise).

**VRAM:** raw `qwen2.5-coder:14b` at 32K KV cache OOMs on 12 GB (`cudaMalloc failed`). Use `coding-agent:latest` (local Modelfile, `num_ctx 16384`). If reviews still OOM, lower to `8192` in `coding-agent.Modelfile` and recreate. If you get HTTP 400 context overflow instead, raise toward `20480` cautiously.

**Reduce reload tax** between `chat` ↔ `coding` within a session — set on GPU `ollama.service`:

```bash
sudo cp REEPCORE/templates/reepcore-setup/config/ollama/systemd/ollama.service.d/override.conf \
  /etc/systemd/system/ollama.service.d/override.conf
sudo systemctl daemon-reload && sudo systemctl restart ollama
```

Sets `OLLAMA_KEEP_ALIVE=10m`. See [`templates/reepcore-setup/config/ollama/README.md`](../templates/reepcore-setup/config/ollama/README.md).

### Dual accelerators (CPU + GPU)

Jarvis can run **two Ollama daemons** so embeddings and security triage stay on CPU while chat/coding/planner/review/friday use GPU VRAM in parallel.

| Daemon | Port | Roles (`ai.yaml`) | `accelerator` |
|--------|------|-------------------|---------------|
| GPU (system `ollama.service`) | `11434` | `chat`, `coding`, `planner`, `review`, `friday` | `gpu` |
| CPU (`ollama-cpu.sh`) | `11435` | `embed`, `security` | `cpu` |

Minimal excerpt (matches shipped template):

```yaml
accelerators:
  cpu:
    host: http://127.0.0.1:11435
  gpu:
    host: http://127.0.0.1:11434

defaults:
  chat: chat
  coding: coding
  review: review
  embed: embed
  planner: planner
  friday: friday
  security: security

models:
  embed:
    accelerator: cpu
    name: qwen3-embedding:0.6b
    auto_pull: true
  chat:
    accelerator: gpu
    name: my-agent:latest
    think: true
    auto_pull: false
  # … see templates/reepcore-setup/ai.yaml for all seven roles
```

**Setup:**

```bash
# GPU daemon (default system service)
sudo systemctl start ollama
curl -s http://127.0.0.1:11434/api/tags | head

# CPU-only second daemon
ollama-cpu.sh start
ollama-cpu.sh pull qwen3-embedding:0.6b

# Ensure models on both hosts
reepcore jarvis models ensure --role chat
reepcore jarvis models ensure --role embed
reepcore jarvis models ensure --role security
```

Per-model `host:` overrides the accelerator URL. `OLLAMA_HOST` remains the fallback when an accelerator host is down (Jarvis logs a warning to stderr).

**Footer:** `Model: chat (my-agent:latest) · gpu · think` and `embed: embed (qwen3-embedding:0.6b) · cpu`. While streaming, status shows live `CPU …% · GPU …%`.

**Smoke test:**

1. Start both daemons; `reepcore jarvis models list` — both hosts show `✓`.
2. Enable RAG; send a chat message — `nvidia-smi` shows GPU use; embed hits `:11435`.
3. `@orchestrate review src/lib.rs` — specialists use GPU host without errors.
4. `ollama-cpu.sh stop` — embed falls back to `OLLAMA_HOST` with a warning; chat still works.

Example manifest (identical to [`templates/reepcore-setup/ai.yaml`](../templates/reepcore-setup/ai.yaml)):

```yaml
accelerators:
  cpu:
    host: http://127.0.0.1:11435
  gpu:
    host: http://127.0.0.1:11434

defaults:
  chat: chat
  embed: embed
  planner: planner
  coding: coding
  review: review
  friday: friday
  security: security

models:
  embed:
    accelerator: cpu
    source: ollama
    name: qwen3-embedding:0.6b
    auto_pull: true

  chat:
    accelerator: gpu
    source: ollama
    name: my-agent:latest
    think: true
    auto_pull: false

  coding:
    accelerator: gpu
    source: ollama
    name: coding-agent:latest
    think: false
    auto_pull: false

  planner:
    accelerator: gpu
    source: ollama
    name: planner-agent:latest
    think: true
    auto_pull: false

  friday:
    accelerator: gpu
    source: ollama
    name: friday-agent:latest
    think: false
    auto_pull: false

  security:
    accelerator: cpu
    source: ollama
    name: security-agent:latest
    think: false
    auto_pull: false

  review:
    accelerator: gpu
    source: ollama
    name: review-agent:latest
    think: true
    auto_pull: false
```

Create local models once (not `ollama pull` for `*-agent` tags):

```bash
ollama pull qwen3.5:9b
ollama pull qwen3.5:4b
ollama pull qwen3.5:2b
ollama pull qwen2.5-coder:14b

mkdir -p ~/.config/ollama
cp REEPCORE/templates/reepcore-setup/config/ollama/*.Modelfile ~/.config/ollama/

ollama create my-agent -f ~/.config/ollama/my-agent.Modelfile
ollama create coding-agent -f ~/.config/ollama/coding-agent.Modelfile
ollama create planner-agent -f ~/.config/ollama/planner-agent.Modelfile
ollama create review-agent -f ~/.config/ollama/review-agent.Modelfile
ollama create friday-agent -f ~/.config/ollama/friday-agent.Modelfile

ollama-cpu.sh start
OLLAMA_HOST=http://127.0.0.1:11435 ollama pull qwen3.5:4b
OLLAMA_HOST=http://127.0.0.1:11435 ollama create security-agent -f ~/.config/ollama/security-agent.Modelfile
OLLAMA_HOST=http://127.0.0.1:11435 ollama pull qwen3-embedding:0.6b
```

Legacy example (separate models per role — only if you have 24 GB+ VRAM):

```yaml
models:
  chat:
    accelerator: gpu
    name: my-agent:latest
    think: true
    auto_pull: false

  coding:
    accelerator: gpu
    source: ollama
    name: qwen2.5-coder:14b
    think: false
    auto_pull: false

  embed:
    accelerator: cpu
    source: ollama
    name: qwen3-embedding:0.6b
    auto_pull: false
```

Source forms:

- `id:` — use as-is (`hf.co/user/model:Q6_K`, `my-agent:latest`, `qwen2.5-coder:32b`)
- `source: huggingface` + `repo` + `quant` → `hf.co/{repo}:{quant}`
- `source: ollama` + `name` → Ollama model name
- `accelerator: cpu|gpu` — host from `accelerators.<key>.host` (dual-Ollama setup)
- `host:` — per-model Ollama URL (overrides `accelerator`)
- `think: true|false` — per-model Ollama reasoning (`think: true` on `/api/chat`). **Overrides `OLLAMA_THINK`** for that alias. Omit to fall back to the `OLLAMA_THINK` env var (default off). Set `think: false` on non-reasoning models (e.g. `qwen2.5-coder`); set `think: true` on thinking models (e.g. `my-agent`).

When `auto_pull: true`, Jarvis checks `GET /api/tags` on open (chat) or before RAG indexing (embed) and runs `POST /api/pull` if the model is missing. Set `auto_pull: false` for large models you prefer to pull manually. Pull progress appears in the Jarvis status bar during downloads.

When you switch chat models in the TUI picker, Jarvis sets your **preferred chat** model (status: `Preferred: chat (my-agent:latest)`). Per-turn routing may still switch to `coding` on code tasks. Reasoning follows the active model's `think` setting (footer chip: `· think` / `· no-think`).

**CLI:**

```bash
reepcore jarvis models list              # manifest aliases + install status (all roles)
reepcore jarvis models ensure --role chat
reepcore jarvis models ensure --role review
reepcore jarvis models ensure --role embed --pull
reepcore jarvis models ensure --role security
reepcore jarvis models pull coding       # pull one alias from ai.yaml
reepcore jarvis security watch           # background ntopng alert triage daemon
reepcore jarvis security watch --foreground
reepcore jarvis security voice-test      # Piper smoke test for alert notifications
```

**TUI:** Click `Model:` or `embed:` in the Jarvis footer (or **Ctrl+Shift+L** / **Ctrl+Shift+E**) to pick from `ai.yaml` aliases. Footer labels show `alias (ollama-id) · cpu|gpu · think|no-think` for the active turn model.

Env vars still override `ai.yaml` for quick experiments and are never auto-pulled.

### Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OLLAMA_HOST` | `http://127.0.0.1:11434` | Fallback Ollama API base URL when per-model `host` / `accelerator` hosts are unreachable |
| `OLLAMA_MODEL` | `my-agent:latest` | Model name for `/api/chat` (overrides `ai.yaml` chat role) |
| `OLLAMA_NUM_CTX` | *(unset)* | Override context window per `/api/chat` request (e.g. `16384` on 12 GB when `qwen2.5-coder:14b` OOMs) |
| `OLLAMA_KEEP_ALIVE` | `5m` (Ollama default) | Server env on `ollama.service` — how long loaded models stay in VRAM after a request. Template sets `10m` for Jarvis role switching (see [Recommended layout](#recommended-layout-12-gb-gpu)) |
| `OLLAMA_NUM_PREDICT` | `4096` | Max tokens per reply (caps runaway streams) |
| `OLLAMA_REPEAT_PENALTY` | `1.15` | Penalty for repeating tokens (raise if loops persist) |
| `OLLAMA_THINK` | `0` | Set to `1` to enable Ollama native reasoning when the active alias has no `think:` in `ai.yaml`. Per-model `think: true|false` in `ai.yaml` **overrides** this env var. When off (or model has `think: false`), output in Ollama's `thinking` field is routed into **chat** (not the inline Thinking box) so replies are not lost |
| `JARVIS_THINKING_PANEL` | `1` | When `JARVIS_THINKING_LEGACY_PANEL=1`, show the legacy pinned thinking strip when reasoning exists; set `0` to hide |
| `JARVIS_THINKING_LEGACY_PANEL` | `0` | Set to `1` to restore the old fixed-height thinking panel (default: inline collapsible boxes in chat) |
| `JARVIS_THINKING_MAX_BYTES` | `32768` | Max stored reasoning bytes per assistant reply |
| `JARVIS_THINKING_PARSE_TAGS` | `0` | Set to `1` to strip `` blocks from content into the thinking strip |
| `JARVIS_WEB_SEARCH` | `1` (on) | Set to `0` to disable `@search` / `@web` |
| `JARVIS_WEB_MAX_RESULTS` | `5` | Max hits injected per search (1–10) |
| `JARVIS_HISTORY_MAX_TURNS` | `8` | Max prior user+assistant turns included in model context |
| `JARVIS_HISTORY_MAX_BYTES` | `24576` | Byte cap for included chat history |
| `JARVIS_MEMORY_MAX_BYTES` | `4096` | Byte cap for long-term memory injection (`memory.md`) |
| `JARVIS_MAX_SOURCE_FILE_BYTES` | `24576` | Max bytes per injected source file (reepedit buffer, `@` path reads, tree selection; 8192–131072). Above this, reepedit injection **windowed** around scroll + cursor |
| `JARVIS_EDITOR_CONTEXT_WINDOW` | `120` | Lines in the scroll window when reepedit content is windowed (20–400) |
| `JARVIS_EDITOR_CONTEXT_HEAD` | `48` | Head lines (imports, module attrs) kept when windowing omits the start (0–200) |
| `JARVIS_REVIEW_CHUNK_MAX_LINES` | `500` | Line count above which a **review-only** turn injects a windowed (head + tail + symbol outline) reepedit buffer instead of the full file. `0` = always inject full; edit/diff turns are never chunked (120–20000) |
| `JARVIS_RAG` | `0` | Local semantic RAG (`1` / `true` / `on` enables at startup). Footer **embed:** toggle overrides per session |
| `JARVIS_RAG_EMBED_MODEL` | `qwen3-embedding:4b` | Ollama model for `/api/embeddings` (overrides `ai.yaml` embed role; `reepcore init` ships `qwen3-embedding:0.6b` in `ai.yaml`) |
| `JARVIS_RAG_RETRIEVE_TIMEOUT` | `15` | Seconds before abandoning query embed on the chat hot path (3–120) |
| `JARVIS_RAG_INDEX_EMBED_TIMEOUT` | `120` | Seconds per chunk during background indexing (10–600) |
| `JARVIS_RAG_TOP_K` | `5` | Max code chunks injected per retrieval (1–20) |
| `JARVIS_RAG_MAX_DISTANCE` | `0.6` | Cosine distance cutoff — higher = looser matches |
| `JARVIS_RAG_MAX_BYTES` | `8192` | Max bytes of RAG context appended to the user prompt |
| `JARVIS_RAG_MIN_PROMPT_LEN` | `20` | Decider skips RAG on shorter messages |
| `JARVIS_RAG_CHUNK_CHARS` | `1200` | Target chunk size when indexing |
| `JARVIS_RAG_CHUNK_OVERLAP` | `120` | Overlap between consecutive chunks |
| `JARVIS_REPETITION_GUARD` | `1` (on) | Set to `0` to disable stream anti-loop cutoff (applies to **chat and thinking** streams). Also aborts verdict flip-flop / self-debate spirals |
| `JARVIS_REPETITION_STRIKES` | `3` | Consecutive repetition signals required before abort (1–12) |
| `JARVIS_REVIEW_THINK_BUDGET` | `1` (on) | Cap runaway **review** reasoning and auto-finalize; set `0` to let review thinking run unbounded |
| `JARVIS_REVIEW_THINK_MAX_BYTES` | `16384` | Reasoning bytes a review turn may use before Jarvis stops and auto-finalizes into bullets (2048–131072) |
| `JARVIS_SIMPLE_THINK_MAX_BYTES` | `4096` | Tighter thinking cap for short yes/no questions (JetBrains MCP, capability checks; 1024–32768) |
| `JARVIS_VOICE` | off | Set to `1` to enable voice STT/TTS in Jarvis view |
| `JARVIS_VOICE_STT` | on when voice on | Wake-word speech input (`0` to disable) |
| `JARVIS_VOICE_TTS` | on when voice on | End-of-turn spoken summary (`0` to disable) |
| `JARVIS_WHISPER_BIN` | `~/whisper.cpp/build/bin/whisper-cli` | whisper.cpp CLI for STT |
| `JARVIS_WHISPER_MODEL` | `~/.local/share/reepcore/jarvis/voice/ggml-base.en.bin` | ggml whisper model |
| `JARVIS_WHISPER_THREADS` | `4` | CPU threads for whisper-cli (`-t`) |
| `JARVIS_PIPER_BIN` | `/usr/bin/piper-tts` | Piper TTS binary |
| `JARVIS_PIPER_VOICE` | `~/en_GB-alan-medium.onnx` | Piper `.onnx` voice model |
| `JARVIS_VOICE_WAKEWORD` | `jarvis` | Required prefix on spoken commands |
| `JARVIS_VOICE_SILENCE_MS` | `1200` | Silence after speech before STT runs |
| `JARVIS_VOICE_MAX_RECORD_SECS` | `15` | Max command recording length |
| `JARVIS_VOICE_AUTO_SEND` | off | Auto-submit after STT (default: review in input bar first) |
| `JARVIS_VOICE_SPEECH_THRESHOLD` | `0.012` | Mic RMS threshold for speech detection |
| `JARVIS_VOICE_INPUT` | *(auto)* | cpal input device name substring — auto-prefers `pipewire` / `pulse` over silent ALSA `default` on Linux |
| `JARVIS_VOICE_SINK` | *(system default)* | Optional PipeWire/PulseAudio sink override for TTS — when unset, playback uses your current default output device |
| `BRAVE_SEARCH_API_KEY` | *(unset)* | Optional — use Brave Search API instead of DuckDuckGo |
| `REEPCORE_REPO` | *(auto-detect)* | Path to the ReepCore git repo root |
| `SHELL` | *(zsh if installed)* | Used when `$SHELL` already points to zsh; otherwise `/usr/bin/zsh` or `/bin/zsh` |

Source files injected for review use **line-number prefixes** (` 105| code`) so Jarvis can cite real line numbers. Ask it to use those `N|` values when reviewing — they are **not** duplicate copies of the file. Windowed injections mark omitted ranges — do not ask Jarvis to cite lines in omitted sections.

The footer shows **`model:`** (chat, `OLLAMA_MODEL`, default **`my-agent:latest`**) and **`embed:`** (RAG embedding model, or **`off`** when RAG is disabled — **off by default**; set `JARVIS_RAG=1` to start with embed on). Click either name to open the picker, or use **Ctrl+Shift+L** / **Ctrl+Shift+E**. Click **`embed:`** or **`off`** (or **Ctrl+Shift+G**) to toggle RAG for the session without restarting.

**TUI footer defaults** when Jarvis opens: `model: my-agent:latest` · `embed: off` · **Chat** · **Live**. Opt in via env: `JARVIS_RAG=1` (embed on), `JARVIS_AGENT=1` (Agent), `JARVIS_AGENT_SANDBOX=1` (Sandbox).

---

## Persistence & memory

- **Chat sessions** persist under `~/.local/state/reepcore/jarvis/sessions/` and are managed with **Ctrl+N** (new) and **Ctrl+O** (picker).
- Assistant **reasoning** is saved in session JSON (collapsed in the UI by default) but is **not** sent back to Ollama on later turns — only `content` is included in chat history.
- **Chat history in the model:** Each turn sends prior user/Jarvis/**run** messages (capped by `JARVIS_HISTORY_MAX_TURNS` and `JARVIS_HISTORY_MAX_BYTES`). The first user message is always kept when trimming. Agent continuation prompts are UI-only and are not duplicated in the Ollama payload. The current prompt also includes a `## This Jarvis chat session` anchor with the first user message so meta-questions (“what did I ask first?”) stay grounded.
- In the sessions picker: **Enter** switches, **d** deletes (press twice to confirm), **e** exports the session to markdown at `~/.local/state/reepcore/jarvis/sessions/<id>.md` (export omits reasoning by default).
- **Long-term memory** is injected from `~/.local/state/reepcore/jarvis/memory.md` (capped by `JARVIS_MEMORY_MAX_BYTES`).
- **Shortcut:** `@remember <fact>` appends a line to `memory.md` without calling the model.

### Local RAG (semantic codebase search)

Jarvis can inject **relevant code snippets** from your project when a message looks like a code question. RAG is **Rust-native** (`src/jarvis_rag/`): Ollama embeddings + a SQLite vector index — no Python sidecar, no ChromaDB.

**When it runs:** a **rule-based decider** (zero extra LLM latency) — not every message. Typical triggers include phrases like *where is*, *find all*, *fix the*, *what does*, *review*, *across the project*, *duplicated function*, or mentions of `.py` / `.rs` / `@reepedit`.

**Skipped (no retrieve embed, chat starts immediately):**

| Case | Why |
|------|-----|
| Greetings, short messages | Below `JARVIS_RAG_MIN_PROMPT_LEN` |
| `@open` / `@home`, `@search` / `@web` | Navigation or web, not codebase RAG |
| **Diff-only** turns | User will Ctrl+A apply — buffer already targeted |
| **Issues-only / no-diff** turns | `no diff`, `issues only`, `no code fences`, … |
| **`@reepedit` + open file** | Full buffer already injected; no cross-file phrases (`across the project`, `where is`, `compare with other.rs`, …) |
| Footer **`embed: off`** / `JARVIS_RAG=0` | RAG disabled |

Background **indexing** may still run when RAG is on (first message, `@reindex`, Ctrl+S) — it uses the longer `JARVIS_RAG_INDEX_EMBED_TIMEOUT` and does not block the chat stream.

**Three search layers (different jobs):**

| Mechanism | What it searches | When |
|-----------|------------------|------|
| **RAG** | Indexed project source (semantic) | Decider fires on your message |
| **`@search` / `@web`** | DuckDuckGo / Brave (web) | You prefix the message |
| **`@search rg …`** (agent tool) | Exact text in repo | Agent mode / manual tool |

**Index & corpus**

| Item | Detail |
|------|--------|
| **Database** | `~/.local/state/reepcore/jarvis/rag/index.db` |
| **Embed model** | `JARVIS_RAG_EMBED_MODEL` (code fallback `qwen3-embedding:4b`; shipped `ai.yaml` uses `qwen3-embedding:0.6b` on CPU `:11435`); footer picker overrides per session |
| **Indexed roots** | Parent dir of open reepedit file → docked tree root (`@tree`) → ReepCore repo — **`$HOME` is never indexed**; use `@tree ~/Projects/foo` for a project |
| **File types** | `.py`, `.rs`, `.md`, `.yaml`, `.yml`, `.toml`, `.json`, `.js`, `.ts`, `.tsx`, `.jsx`, `.go` |
| **Skipped paths** | `.git`, `target/`, `node_modules/`, **Cursor/`.cursor` history**, jarvis sandboxes, Steam caches, etc. |

**Keeping the index fresh**

| Action | Effect |
|--------|--------|
| First message in a session | Background index of corpus roots (status: *RAG indexed N chunk(s)…* in footer) |
| **`@reindex`** | Full background reindex (status: *RAG reindex running…*) |
| **`@tree <path>`** | Background reindex of new tree root |
| **reepedit Ctrl+S** | Re-indexes that saved file only |

**Deduping:** chunks from files already in the reepedit buffer, tree selection, or explicitly read via `augment_user_prompt` this turn are excluded from RAG hits.

**Agent mode:** multi-step agent runs still run RAG against your **original** user prompt (not the internal continuation text).

**Output:** up to `JARVIS_RAG_TOP_K` chunks (default 5), cosine-filtered, capped at `JARVIS_RAG_MAX_BYTES` (default 8 KiB), injected as `## RAG context (local codebase)` before Ollama `/api/chat`. Query embed uses a **short timeout** (`JARVIS_RAG_RETRIEVE_TIMEOUT`, default 15s) so chat does not stall behind a large embed model. If retrieve is skipped or embed fails, you may see `[RAG skipped: …]` in the prompt (harmless for single-file `@reepedit` reviews).

Disable entirely with **`JARVIS_RAG=0`** (default) or footer **Ctrl+Shift+G** (`embed: off`). Enable at startup with **`JARVIS_RAG=1`**.

**Future (not in v1):** Jarvis session history and docs-only passes as additional corpora.

---

## Tips

- **One script per reply** — Jarvis is prompted to give a single concise fenced block for commands; use **Ctrl+R** once rather than re-pasting.
- **Verify in the pane** — Jarvis is told not to claim a command ran unless output appears in the terminal context.
- **Prefer diffs for edits** — For files open in reepedit, Jarvis should suggest `diff` hunks; use **Ctrl+A** instead of hand-copying.
- **Review without diffs** — For audits, use `Review @reepedit only list issues, no diff blocks` (or `issues only`). RAG retrieve is skipped; Ctrl+A is disabled for that turn.
- **Code review diffs** — Jarvis must diff against the file content in context only, keep hunks small, and say **no diff needed** when the file is already correct. It should not re-add existing `match` arms or emit broken mega-diffs; run `cargo check` mentally before suggesting Rust changes.
- **Visibility vs review** — *“Can you see the file open in reepedit?”* gets a one-sentence yes/no from injected context, not a full code review.
- **Destructive commands** — Jarvis is instructed to confirm before rm, force push, format, etc.; still review before **Ctrl+R**.
- **File Browser context** — Navigate the File Browser (Main Menu → 4) before asking Jarvis about a directory listing; selected files are injected automatically.
- **Installed Packages** — Open Main Menu → 5 (or `@open 5`) before asking about installed/selected packages; Jarvis receives tab counts, the visible list (`>` = selection), and pacman/flatpak info for the highlighted row each message you send from Jarvis.
- **Semantic codebase search** — For cross-file questions (*where is X defined?*), let RAG run (decider fires automatically) after `@reindex` once; use `@search rg foo` when you know the exact symbol string. For a single open file in reepedit, RAG retrieve is usually skipped — leave **embed** on for indexing or turn it off (**Ctrl+Shift+G**) if Ollama is GPU-bound.

---

## Troubleshooting

| Symptom | Things to check |
|---------|-----------------|
| “Error: …” in chat | Ollama running? Model name correct? `curl $OLLAMA_HOST/api/tags` |
| HTTP 500 — `cudaMalloc failed` / KV cache OOM | **coding** on 12 GB with raw `qwen2.5-coder:14b` — create `coding-agent` (`num_ctx 16384`). Run `ollama ps`; if two models show, wait for keep-alive expiry or `ollama stop <name>` |
| HTTP 400 — prompt longer than context length | Footer shows which role loaded. **coding** needs `coding-agent` Modelfile or lower Jarvis injection. **chat** — raise `num_ctx` in `my-agent` Modelfile |
| Empty or stuck stream | **Esc** to cancel; check Ollama logs |
| No thinking visible | Default UI uses **inline Thinking boxes** — press **Ctrl+Y** on a reply from a thinking model. Legacy panel needs `JARVIS_THINKING_LEGACY_PANEL=1`. Model must support reasoning (`think: true` in `ai.yaml` or `OLLAMA_THINK=1`) |
| Ctrl+R does nothing | Last assistant message needs a ```bash/zsh/sh fenced block |
| Model repeats endlessly | Jarvis stops true loops after several strikes; raise `OLLAMA_REPEAT_PENALTY` or use a stronger model |
| “Stopped — model was repeating” on good replies | Guard was too eager (fixed in recent builds); set `JARVIS_REPETITION_STRIKES=5` or `JARVIS_REPETITION_GUARD=0` |
| Reasoning loops forever, chat empty | Model stuck in `OLLAMA_THINK` reasoning; guard aborts loops. Recovers ```diff or **review bullets** into chat when possible. **Review** turns auto-finalize into a clean bullet list (tune `JARVIS_REVIEW_THINK_MAX_BYTES`, disable with `JARVIS_REVIEW_THINK_BUDGET=0`). For reviews you can also use **`OLLAMA_THINK=0`** or `issues only — no diff` |
| Review keeps re-debating itself | The oscillation guard + auto-finalize handle this; if it persists, lower `JARVIS_REVIEW_THINK_MAX_BYTES` (e.g. `8192`) or use `OLLAMA_THINK=0` for that turn |
| `@open` on a directory — "file did not open" | `@open` requires a **file** (e.g. `base.yaml`), not a folder. For profiles/modules overview, ask in chat — Jarvis already has profile YAML in context. Use `@navigate 6` (Profiles) or `@navigate 3` (Module Manager) to browse the TUI |
| Directory review dumps files | Ask per-file (`review jarvis.md`) or use `@reepedit`; bulk `review docs/` gets listings only |
| Review claims file is "truncated" past line ~300 | Raise `JARVIS_MAX_SOURCE_FILE_BYTES` (default 24576) or scroll reepedit — large files use a **window** (`JARVIS_EDITOR_CONTEXT_WINDOW` / `_HEAD`) |
| Wrong repo context | Set `REEPCORE_REPO=/path/to/ReepCore` |
| Jarvis says it can't see shell output | Wait for the command to finish, then send another message — output is injected as `## Embedded terminal pane` each turn |
| Shell is not zsh | Install zsh; or set `$SHELL` to your zsh binary |
| sudo hangs | Type password in the shell pane, not a separate command bar |
| `[RAG skipped: index empty]` | Run `@reindex` or open a project under `@tree ~/Projects`; ensure embed model is pulled (`reepcore jarvis models ensure --role embed`) and Ollama is running |
| `[RAG skipped: embed …]` / embed timeout | Retrieve uses 15s timeout — often harmless on `@reepedit` reviews (retrieve skipped). For cross-file RAG, check `OLLAMA_HOST`, try `nomic-embed-text`, or raise `JARVIS_RAG_RETRIEVE_TIMEOUT` |
| RAG never triggers | Message too short, `@open`, diff-only / issues-only / `@reepedit`+open-file skip — use cross-file phrasing (*where is total defined across the project?*) |
| Ctrl+A blocked on review | Issues-only turn — intentional. Follow up with `one small diff …` when you want a patch |
| RAG hits wrong files | `@tree` to your project root first, then `@reindex`; narrow with explicit `@reepedit path` |
| `btop` / `htop` looks garbled or doubled in shell pane | Rebuild — pane now sizes the PTY to the split rect and paints an opaque background. Close pane (**Ctrl+T**) and `@run btop` again |
| `@reepedit main.rs` opens wrong file | Use a full or repo-relative path (`@reepedit REEPCORE/src/main.rs`) — bare filenames match the first hit under corpus roots |
| Voice STT off / no mic | Set `JARVIS_VOICE=1`; download `ggml-base.en.bin`; check `JARVIS_WHISPER_BIN` exists. Status bar shows the reason at startup |
| Voice never transcribes | Start phrase with **"Jarvis"** (e.g. *Jarvis, list kernels*). Background speech without the wake word is ignored. On Linux, Jarvis auto-selects the `pipewire`/`pulse` input — ALSA `default` is often silent; override with `JARVIS_VOICE_INPUT=pipewire` or your headset name |
| No spoken summary | TTS fires at **turn end** (normal chat stream or **orchestrator** finish — not mid-run). Needs `JARVIS_VOICE_TTS=1`, piper voice + binary, and a non-empty assistant reply (synthesis for orchestrator). Status should show **Speaking summary…** |
| Hear nothing but TTS ran | TTS uses your **current default output** (`wpctl status` / `pactl get-default-sink`). Set the OS default to your headphones, or override with `JARVIS_VOICE_SINK` from `pactl list short sinks` |
| Piper silent / error | Run smoke test above; confirm `JARVIS_PIPER_VOICE` points at your `.onnx` + sibling `.json` |

---

## Agent loop harness (testing)

Integration tests in [`tests/agent_loop_harness.rs`](../tests/agent_loop_harness.rs) drive `AiAssistantState` through the same `poll_stream` / `poll_agent_command` cycle the TUI uses each frame. They require the `test-harness` feature because integration tests do not enable `#[cfg(test)]` on the library crate.

**Pure tests (CI-safe, no Ollama):**

```bash
cd REEPCORE
cargo test --test agent_loop_harness --features test-harness
```

**Live-model tests (dev machine, `#[ignore]` by default):**

```bash
cd REEPCORE
OLLAMA_HOST=http://127.0.0.1:11434 OLLAMA_MODEL=my-agent:latest \
REEPCORE_REPO=/path/to/ReepCore \
cargo test --test agent_loop_harness --features test-harness -- --ignored --nocapture
```

| Test | Needs model | What it checks |
|------|-------------|----------------|
| `gating_predicates_agree_with_known_examples` | No | Shell-script allow/block predicates |
| `apply_before_test_gate_blocks_until_applied_and_saved` | No | Patch must be applied + saved before `@test` (gate function in isolation) |
| `agent_blocks_shell_test_when_patch_pending` | No | `@test` shell dispatch blocked via `block_agent_command` when patch unapplied |
| `agent_blocks_tool_test_when_patch_pending` | No | `tool:test` / user `@test` blocked via `block_agent_command` (Agent + User origin) |
| `agent_allows_test_after_patch_saved` | No | Gate clears after apply+save; trivial `python3 -c pass` runs |
| `shell_gating_blocks_destructive_commands` | Yes | Destructive shell blocks never execute |
| `tool_fence_dispatch_respects_step_limit_and_dedup` | Yes | Tool fences respect step limit and dedup |
| `sandbox_apply_then_promote_produces_changed_file` | Yes | Sandbox patch → save → promote round-trip |
| `patch_gate_blocks_agent_test_before_apply` | Yes | Live diff from model → `@test` blocked before Ctrl+A apply |

**Orchestrator tests (pure, CI-safe):**

| Test | What it checks |
|------|----------------|
| `planner_validate_accepts_good_plan` | Valid JSON plan parses and validates |
| `planner_validate_rejects_synthesize_in_parallel` | `synthesize` cannot run in `parallel` |
| `planner_validate_rejects_missing_goals` | Every referenced agent needs a `tasks` goal |
| `planner_validate_rejects_invalid_then` | `then` must be exactly `["synthesize"]` |
| `should_orchestrate_heuristic_matches_review_and_test` | Heuristic + `@orchestrate` prefix detection |
| `dispatch_runs_parallel_mock_specialists` | Mock runner: parallel specialists + synthesize chain |
| `specialist_tool_gate_code_review_blocks_test` | CodeReview profile rejects `@test` |
| `specialist_tool_gate_test_runner_allows_test_only` | TestRunner allows `@test` only |

**Orchestrator live test (`#[ignore]`):**

| Test | Needs model | What it checks |
|------|-------------|----------------|
| `orchestrator_review_and_test_parallel` | Yes | Full planner → CodeReview + TestRunner parallel → synthesize via TUI state |

**Isolation:** live scenarios redirect `$HOME` to a temp directory so Jarvis sessions, sandboxes, and memory files do not touch your real `~/.local/state/reepcore`. `REEPCORE_REPO` must be an **absolute** path to your checkout.

**Sandbox scenario:** edits `.jarvis-harness-scratch.md` at repo root and restores the original file on exit (even on panic). The live promote step briefly writes that file back to your checkout before revert.

---

## Multi-agent orchestrator

When `JARVIS_ORCHESTRATOR=1` and agent mode is on, Jarvis can route qualifying prompts through a **planner → dispatch → specialists** pipeline instead of the single-agent ReAct loop.

### Architecture

1. **Planner** — one Ollama call, JSON-only output (`parallel`, `then`, `tasks`). Validated in Rust before any specialist runs.
2. **Dispatch** — pure Rust: runs `code_review`, optional `peer_review` (Friday), and `test_runner` in parallel, then `synthesize` sequentially.
3. **Specialists** — isolated mini-loops with restricted tool subsets:
   - `code_review` — Jarvis read-only review: `@search` (`rg`, `cat`, `head`, `tail`, `ls`), `@git` only
   - `peer_review` — **Friday** skeptical second opinion (same read-only tools; challenges assumptions and risks)
   - `test_runner` — `@test` only (respects patch apply-before-test gate)
   - `synthesize` — no tools; merges parallel outputs (includes **Peer review (Friday)** section when Friday ran)

If the planner fails validation or dispatch errors, Jarvis **falls back** to the normal agent loop automatically.

**Read-only tools use the live checkout:** orchestrator specialists (and agent `@test` when not editing in sandbox) run `cargo check` / `rg` against your live `~/ReepCore/REEPCORE` tree, not the session sandbox worktree. Sandboxes are git worktrees frozen at creation and do not include uncommitted files — using them for compile checks produces false "module not found" errors.

**Orchestrator sub-calls ignore `OLLAMA_THINK`:** planner, specialists, and synthesizer always use `think: false` so JSON plans, tool fences, and the **Synthesis** section stay in chat — not in a reasoning trace. Main Jarvis chat streaming respects per-model `think:` in `ai.yaml`, then `OLLAMA_THINK`.

**Voice (TTS):** spoken summary runs when orchestrator dispatch **completes** (same as normal chat turn end), using the **Synthesis** text plus orchestrator tool titles for context.

**Large-file review:** CodeReview is prompted to use `head -n 80`, targeted `rg`, and `tail -n 80` instead of full `cat` on big sources. Raise `JARVIS_ORCHESTRATOR_READ_MAX_BYTES` if tool output is still truncated.

**Collapsible chat output:** Orchestrator plan and tool runs render as **folded boxes** (4 preview lines; `cargo check`/`test` show the **last** lines when collapsed). **Click** the `▸` header (or the `╰─` footer) to expand or collapse. On **all-green** runs, specialist summaries are **hidden** (synthesis only); they appear when any check fails or a summary flags issues. **Synthesis** is always shown.

**Agent `run:` blocks:** long `cat`/`ls`/`head` results also render as folded boxes. Multi-line ```bash``` command queues in assistant replies collapse to _▸ Queued reads (N commands — see **run:** blocks below)_ once the agent executes them.

### Environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `JARVIS_ORCHESTRATOR` | off | Master switch (`1` / `true` / `yes` / `on`) |
| `JARVIS_PEER_REVIEW` | off | Enable Friday `peer_review` specialist in orchestrator plans (`1` enables up to 3 parallel agents) |
| `JARVIS_ORCHESTRATOR_MAX_PARALLEL` | `2` or `3` | Override max parallel specialists (`3` when peer review on) |
| `JARVIS_PLANNER_MODEL` | same as `OLLAMA_MODEL` / `ai.yaml` planner | Model for the planner JSON call |
| `JARVIS_ORCHESTRATOR_TIMEOUT` | `180` | Total wall-clock seconds for one orchestration run |
| `JARVIS_ORCHESTRATOR_READ_MAX_BYTES` | `65536` | Tool stdout cap for CodeReview read tools (`cat`/`head`/`tail`/`rg`) |
| `JARVIS_ORCHESTRATOR_HEAD_MAX_LINES` | `120` | Max `-n` for orchestrator `head` / `tail` |
| `JARVIS_READ_CAT_MAX_BYTES` | `524288` | Max file size for read-only `cat` in `@search` |

Requires `JARVIS_AGENT=1` (agent mode) — orchestrator only activates when agent mode is on.

### When it triggers

Orchestrator runs when **all** of:

- `JARVIS_ORCHESTRATOR=1`
- Agent mode enabled
- Prompt matches heuristic **or** starts with `@orchestrate`

**Heuristic** (any of):

- Contains both review/audit and test/check/compile/cargo language
- **Rust source review** — e.g. `review lib.rs`, `review blackjack.rs`, `review src/foo.rs` (runs code_review + test_runner)
- Contains a file path hint (`.rs`, `src/`, `/`, `reepcore`) plus check/lint/clippy (without review)

Basenames like `blackjack.rs` are resolved under the REEPCORE package (e.g. `BlackJack_rust/blackjack.rs`). **test_runner** runs the matching verify command deterministically (`cargo check -p <crate>` when a `Cargo.toml` exists nearby, else `rustc --check <path>`) — not `cargo check -p reepcore` unless the target is under `src/`.

**Not orchestrator** (normal chat / patch agent): issues-only reviews, reepedit/open-file review, Python game test phrases, **fix/edit/remove prompts**. When orchestrator is off but the prompt is review+verify, agent mode receives injected target paths and the correct ```tool:test``` command.

**Agent shell:** piped `cargo check|test|build` in ```bash``` blocks (misleading exit 0 from `head`); use ```tool:test``` instead.

**Override:** `@orchestrate review src/foo.rs and cargo check -p reepcore` always triggers when enabled.

### Friday (peer review)

When `JARVIS_PEER_REVIEW=1`, the planner may add `peer_review` to the parallel phase. **Friday** uses the `friday` role from `ai.yaml` (`friday-agent:latest` on `qwen3.5:2b`, `think: false`, `num_ctx 8192`). Friday runs **read-only** tools only — no diffs, no `@test` — and produces a second-opinion critique merged into **Synthesis** under `### Peer review (Friday)`.

Typical parallel plan for review+verify:

```json
{"parallel":["code_review","peer_review","test_runner"],"then":["synthesize"],...}
```

**Smoke test:**

```bash
JARVIS_AGENT=1 JARVIS_ORCHESTRATOR=1 JARVIS_PEER_REVIEW=1 reepcore tui
# @orchestrate review src/lib.rs and cargo check -p reepcore
```

**Footer:** when `JARVIS_ORCHESTRATOR=1` and Agent mode is on, footer shows **Orch** next to Agent.

### Manual smoke test

```bash
cd ~/ReepCore/REEPCORE
cargo build --release
JARVIS_AGENT=1 JARVIS_ORCHESTRATOR=1 reepcore tui
# In Jarvis chat:
@orchestrate review REEPCORE/src/lib.rs and cargo check -p reepcore
```

Live harness:

```bash
OLLAMA_HOST=http://127.0.0.1:11434 OLLAMA_MODEL=my-agent:latest \
REEPCORE_REPO=/home/neo/ReepCore \
cargo test --test agent_loop_harness --features test-harness orchestrator_review -- --ignored --nocapture
```

### Source modules

- [`src/jarvis_planner.rs`](../src/jarvis_planner.rs) — schema, validation, planner LLM call
- [`src/jarvis_specialist.rs`](../src/jarvis_specialist.rs) — specialist profiles and mini-loops
- [`src/jarvis_dispatch.rs`](../src/jarvis_dispatch.rs) — parallel execution and synthesize chain
- [`src/jarvis_models.rs`](../src/jarvis_models.rs) — `ai.yaml` manifest, role routing, model pickers
- [`src/jarvis_security_triage.rs`](../src/jarvis_security_triage.rs) — ntopng alert triage + voice notify
- [`src/jarvis_sandbox.rs`](../src/jarvis_sandbox.rs) — git worktree sandbox + optional bwrap
- [`src/jarvis_jetbrains.rs`](../src/jarvis_jetbrains.rs) — RustRover MCP client
- [`src/tui_ai_assistant.rs`](../src/tui_ai_assistant.rs) — TUI hook (`submit_prompt`, `poll_orchestrator`)

---

## Security alert triage (optional)

When `~/.config/reepcore/security-triage.toml` has `enabled = true`, ReepCore polls ntopng in the background (TUI and `reepcore jarvis security watch`) and classifies new alerts with the **`security`** model role from `ai.yaml` (typically `security-agent:latest` on the CPU Ollama daemon).

**Pipeline:**

1. **CPU first-pass** — `security-agent` (`security` role, CPU `:11435`) returns `suppress`, `notify`, or `escalate`.
2. **Escalation** — serious alerts can invoke the **planner** role (`planner-agent:latest` on GPU) for a richer summary.
3. **Notify** — desktop notification and/or Piper TTS (`voice_notify` in config; does **not** require `JARVIS_VOICE=1`).

**Setup:**

```bash
cp REEPCORE/docs/config-examples/security-triage.toml ~/.config/reepcore/
# Requires ntopng.toml, ai.yaml security role, ollama-cpu.sh on :11435
ollama-cpu.sh start
reepcore jarvis models ensure --role security
reepcore jarvis security voice-test   # optional Piper check
```

**TUI:** open **Real Time Monitoring → Network Dashboard** to see live ntopng panels and a triaged-alerts strip. Jarvis receives triage state in context when enabled.

**Daemon (24/7 outside TUI):**

```bash
reepcore jarvis security watch --foreground
# Or install docs/config-examples/reepcore-security-watch.service → systemd --user
```

State persists at `~/.local/share/reepcore/security-triage-state.json`. Full setup (Suricata, fail2ban, ntopng profile): [SECURITY_MONITORING.md](SECURITY_MONITORING.md).

# OpenCode v1 architecture

Reading notes so a v2 distill can be compared against the live v1 line. Not a product pitch, not a backlog, not a port plan.

The tree this was read from is `origin/dev` at `dc4449df0d52199704ea4989a5a993ebbc605612` (`chore: update nix node_modules hashes`, 2026-08-29). That is this fork's v1-shaped line. Confirmed on that ref:

- `packages/opencode` is the product binary and the live session loop
- `SessionV2.prompt` exists in `packages/core/src/session.ts` and admits `session_input`
- `session_input` is a real SQLite table in `packages/core/src/session/sql.ts`

Do not read `origin/v2` for this document. That branch deleted `packages/opencode`, renamed the inbox to `session_inbox`, and retired "provider turn" language. Compare against [PR #1](https://github.com/johnnyeric/opencode/pull/1) (`docs/v2-architecture.md`) after reading this file.

## Overview

OpenCode v1 on this tree is still a coding-agent monolith with a split TUI. `packages/opencode` owns the yargs CLI, the Worker-hosted server, Effect HttpApi (Hono is gone), tools, agents, plugins, and the in-memory session loop. TUI, desktop, ACP, and `opencode run` are HTTP/SSE (or Worker-RPC-as-fetch) clients of that server.

The product idea that actually runs when you type a prompt is not SessionV2. `SessionPrompt.prompt` writes a user message into `message` / `part` immediately, then `SessionPrompt.loop` drives an in-memory while-loop. AI SDK `tool({ execute })` runs local tools during `streamText`. The loop reloads Parts and calls the model again until the last assistant finishes without pending tools.

A second stack is already growing in the same process. `packages/core` owns SQLite, EventV2, Location, catalog, and `SessionV2`. `POST /api/session/:id/prompt` admits `session_input` and wakes `SessionExecution`. The TUI's main send is `POST /session/:id/message` (`session.prompt`); `prompt_async` exists for a few other UI flows. Both APIs are mounted on one server.

If you are porting Kilo, do not start from this tree's `KiloPlugin`. `packages/core/src/plugin/provider/kilo.ts` only stamps `HTTP-Referer` and `X-Title` onto `https://api.kilo.ai/api/gateway`. It is not a port. This document does not port anything.

## What to ignore on Monday

Skip these unless you are specifically shipping that product:

- `packages/console`, `enterprise`, `function`, `stats`, `containers`. Hosted billing, cloud functions, CI images.
- `packages/web`. Docs/marketing site (Astro). There is no `packages/www` on this tree.
- `packages/identity`. Tiny asset package.
- `packages/storybook`. Component gallery.
- `packages/slack`. Slack bot host.
- `sdks/vscode`. Opens a terminal and talks to the CLI. Not an execution engine.
- `packages/cli` (`lildax`). Parallel Effect CLI over `packages/server`. Not the `opencode` binary users run.
- `packages/sdk-next`. In-process compose of Client + Core + Server. The published `@opencode-ai/sdk` still spawns the `opencode` binary.
- `specs/v2/*` and `packages/opencode/specs/v2/*`. Contracts and sketches for the emerging core. Several are ahead of, or beside, the live `SessionPrompt` loop. If they disagree with `packages/opencode/src/session/prompt.ts`, trust the code.
- `packages/opencode/src/storage`. JSON files under `~/.local/share/opencode/storage`. Sessions are SQLite. The module is still wired into revert; it is not the transcript.
- `packages/opencode/src/temporary.ts`. Leftover yargs scratch.
- User-facing docs that imply network auth is always on. `opencode serve` warns and listens unsecured when `OPENCODE_SERVER_PASSWORD` is unset. Username defaults to `opencode` and is overridable via `OPENCODE_SERVER_USERNAME`.

Do not skip `packages/core`. It is live SQLite, EventV2, models.dev, and the SessionV2 stack, not a graveyard. Do not skip `packages/core/src/v1`. It is the v1 session/config/permission schema the monolith still uses.

## Package map

Runtime dependency direction, from `AGENTS.md` on this tree:

```text
schema
  -> protocol, core
       protocol -> client
       core + protocol -> server
         client + core + server -> sdk-next
```

That is the emerging split. The product binary does not follow it cleanly. `packages/opencode` imports Core, Protocol, Server, Client, Plugin, LLM, TUI, and Schema, then hosts the live loop itself.

| Package | Job |
| --- | --- |
| `opencode` | Product binary. Yargs CLI, Worker TUI, Effect HttpApi server, `SessionPrompt.loop`, v1 tools/agents/plugins/config |
| `core` | SQLite, EventV2, Location, catalog, SessionV2 inbox/runner, v1 schemas, models.dev |
| `llm` | Provider-neutral stream, protocols, native provider packages. Default TUI path still uses Vercel AI SDK |
| `protocol` | Effect `HttpApi` contracts for the `/api/...` surface (`v2.session.prompt`, etc.) |
| `server` | Binds Protocol to Core SessionV2. Mounted *inside* the opencode server as a second API |
| `client` | Generated Promise/Effect clients for Protocol |
| `sdk` (`packages/sdk/js`) | Published SDK. Spawns `opencode serve`. Hand-written OpenAPI client plus a `/v2` generation |
| `sdk-next` | Embeds Server+Core in-process. Not what TUI uses |
| `plugin` | Public v1 Promise plugin SDK, plus a `src/v2` Effect/Promise SDK the core catalog uses |
| `schema` | Branded IDs, durable event payloads, prompt/input types |
| `cli` | Effect CLI named `lildax`. Not `opencode` |
| `tui` | Terminal UI. HTTP/SSE or Worker-fetch client. Imports Core for frontend bits |
| `app` / `session-ui` / `ui` | Solid web/desktop UI |
| `desktop` | Electron shell. Forks a sidecar `opencode` server via `utilityProcess` |
| `codemode` | JS sandbox used when the experimental code-mode flag is on |

`packages/ai` does not exist here. Native protocols live in `packages/llm`. `packages/opencode` is not gone.

## Key concepts

**Instance / directory.** V1's cache key is a project directory, not a Location object. `InstanceState` (`packages/opencode/src/effect/instance-state.ts`) is a `ScopedCache` keyed by directory. Agents, tools, config, MCP, plugins, and `SessionRunState` are per directory. The server loads an instance per request from `x-opencode-directory` / workspace routing. Omitted workspace means the local filesystem.

**Location.** `{ directory, workspaceID? }` already exists in Core. SessionV2, catalog, and Core tools are Location-scoped through `LocationServiceMap`. The TUI loop mostly ignores this and uses `InstanceRef`.

**Two transcripts.** Live TUI history is `message` + `part` (JSON columns, mutable Parts). SessionV2 history is `session_message` (projected from events) plus pending work in `session_input`. They share the `session` row.

**Two loops.** `SessionPrompt.loop` is the product loop: prompt is visible first, then an in-memory while-loop. `SessionV2.prompt` is the emerging loop: admit first, promote later, one `llm.stream` per provider turn. `AGENTS.md` describes the second. The TUI runs the first.

**Provider turn.** On this tree that phrase is live. One provider turn is one model call inside `SessionRunnerLLM.runTurnAttempt` (`packages/core/src/session/runner/llm.ts`). The v1 loop does not use that name; it increments `step` per while-iteration and may run several AI SDK streams, including title and compaction, as side jobs.

**Steer vs queue.** SessionV2 delivery modes. Default is steer. The TUI path has no inbox, so it has no steer/queue. A second prompt while busy is either joined onto the same `SessionRunState` runner or rejected as busy, depending on the endpoint.

**Busy.** V1 busy is process-local `SessionRunState` plus `SessionStatus`. It is not `session.time_suspended` and is not a restart claim. Kill the process and the in-memory runner is gone. Parts that were already written stay.

**Bus.** Two buses. `EventV2` is process-global, transactional, and always inserts durable payloads into `event`. `GlobalBus` is an in-process `EventEmitter` that the Worker forwards to the TUI. `EventV2Bridge` copies Core events onto `GlobalBus` and stamps directory/workspace.

## Process model

### How a user actually starts this

`packages/opencode/src/index.ts` is the binary (`bin.opencode`). Root command is the full TUI. Yargs, not Effect CLI.

1. Default TUI. Parent process starts a `Worker` (`packages/opencode/src/cli/tui/worker.ts`). The worker holds `Server.Default().app.fetch`. The TUI calls it through JSON RPC disguised as `fetch` (`createWorkerFetch`) plus `global.event` RPC. URL is the fake `http://opencode.internal`. Network listen is off unless you pass `--port` / `--hostname` / `--mdns`.
2. `opencode serve`. Same process becomes a network server (`Server.listen`). Default bind `127.0.0.1` with port `0` (tries 4096, then any free port). Password optional. Missing password prints a warning and stays unsecured.
3. `opencode attach <url>`. TUI as a real HTTP/SSE client. Basic auth from `--username` / `--password` or env.
4. `opencode run` / `--mini`. HTTP client of a local or attached server. Mini is a smaller TUI over the same session API.
5. Desktop. Electron main forks `sidecar.js` as a `utilityProcess` that runs the opencode server. Renderer talks HTTP to that sidecar. WSL is a separate `opencode serve` inside the distro.
6. `opencode acp`. Starts `Server.listen`, then speaks Agent Client Protocol ndjson on the CLI process stdin/stdout using `@opencode-ai/sdk/v2`.
7. `@opencode-ai/sdk` `createOpencodeServer`. Spawns the `opencode` binary with `serve`. Not in-process.

The only in-process full server the TUI uses is the Worker. `--standalone` is a v2 CLI idea and is not how this TUI works.

### Server boot

`packages/opencode/src/server/server.ts` `Default()` builds a web handler from `HttpApiApp`. `listen` binds Node HTTP via Effect `HttpRouter.serve`.

One process, one SQLite file, many directories. Default DB is `~/.local/share/opencode/opencode.db` (`OPENCODE_DB` overrides; non-latest channels use `opencode-<channel>.db`). WAL, 5s busy timeout.

`createRoutes` in `packages/opencode/src/server/routes/instance/httpapi/server.ts` mounts **both** APIs:

- Instance/root HttpApi: `/session/...`, `/event`, `/global/event`, TUI, MCP, config. Handlers live under `packages/opencode/src/server/routes/instance/httpapi/handlers`.
- Protocol HttpApi (`@opencode-ai/server/api`): `/api/session/...`. Handlers live under `packages/server/src/handlers`.

Auth is Basic. Production username defaults to `opencode` but `OPENCODE_SERVER_USERNAME` still works. The v2 line later hardcoded the username.

### Request path for a prompt (what the TUI actually does)

```text
TUI
  -> SDK session.prompt
    -> POST /session/:id/message
      -> InstanceContextMiddleware (directory header)
        -> SessionPrompt.prompt
          -> write user message + parts
          -> SessionPrompt.loop          (waits for idle)
        <- streamed JSON SessionV1.WithParts
```

`POST /session/:id/prompt_async` runs the same `SessionPrompt.prompt` forked and returns 204. The TUI uses it for some secondary flows (session move, workspace create), not the main composer.

### Request path for SessionV2 (mounted, not the TUI default)

```text
generated client POST /api/session/:id/prompt
  -> Protocol endpoint + SessionLocationMiddleware
    -> SQLite session row -> Location.Ref
      -> LocationServiceMap.get(ref)
        -> SessionV2.prompt
          -> SessionInput.admit
          -> SessionExecution.wake   (unless resume:false)
        <- SessionInput.Admitted     (does not wait for the model)
```

SessionV2 routes do not trust the request's current directory. Middleware reads the session's recorded Location. V1 `/session/...` routes *do* use the request directory to load `InstanceState`.

### Events to the UI

V1 session facts publish through `EventV2Bridge` onto `GlobalBus`. `/event` and `/global/event` are SSE. Worker TUI skips HTTP SSE and subscribes to `global.event` RPC.

SSE is volatile. Clients rehydrate with ordinary API reads. Payload history for durable EventV2 types is also in the `event` table on this tree. That is not how the TUI rebuilds a session. The TUI reads `message` / `part`.

Desktop Electron main talks to the renderer over IPC for native bits. Ordinary OpenCode API traffic is HTTP from the renderer to the sidecar.

## A prompt from click to idle

This is the part to internalize. It is the v1 loop.

### 1. Write the user message

`SessionPrompt.prompt` in `packages/opencode/src/session/prompt.ts`:

- Load the session. `SessionRevert.cleanup` commits a staged revert if needed.
- `createUserMessage` materializes parts (text, files, agents, subtasks), runs `chat.message` plugin hooks, writes `message` + `part` immediately.
- Optional per-prompt `tools` booleans become session permission rules.
- `noReply: true` returns without calling the model.
- Otherwise `loop({ sessionID })`.

There is no inbox. The prompt is already in the transcript before the model runs. The TUI composer waits for `SessionPrompt.loop` to finish. `prompt_async` returns 204 only because the handler forks that same loop, not because admission is separate.

Identity: reusing a session ID loads the existing session. Reusing a message ID is not an idempotent admit. The v1 prompt path generates new message IDs.

### 2. Take the session runner

`SessionPrompt.loop` is `SessionRunState.ensureRunning(sessionID, lastAssistant, runLoop)`.

`SessionRunState` (`packages/opencode/src/session/run-state.ts`) is directory-scoped. One in-memory `Runner` per session ID:

- A second `prompt` on a busy session joins that runner (or `assertNotBusy` fails, used by revert/summarize).
- `cancel` interrupts the fiber and background jobs.
- Idle clears the map and publishes status idle.
- Different sessions in the same directory run concurrently.
- Different directories are different `InstanceState` caches.

This is not `SessionExecution`. Killing the process drops the runner. There is no write-ahead claim and no startup recovery of in-flight v1 loops.

### 3. The while loop

`SessionPrompt.run` (`Effect.fn("SessionPrompt.run")`):

1. Mark busy.
2. Reload compacted history (`MessageV2.filterCompactedEffect`).
3. If the last assistant finished (not `tool-calls` / `unknown`), has no live local tool parts, and points at the last user, break.
4. `step++`. On step 1, fork title generation.
5. Resolve the model from the last user message.
6. If a subtask part is pending, `handleSubtask` (task tool / child session) and `continue`.
7. If a compaction part is pending, `compaction.process` and `continue`.
8. If the last finished assistant overflowed context, enqueue compaction and `continue`.
9. Create a new assistant row. `SessionProcessor.create` then `handle.process(streamInput)`.
10. On `stop`, break. On `compact`, enqueue compaction. Otherwise continue.

At `step >= agent.steps`, tools stay advertised (prefix caching) and `MAX_STEPS_PROMPT` is appended. Structured output injects a `StructuredOutput` tool and a system reminder.

### 4. One processor pass, tools inside the stream

`SessionProcessor.process` consumes `LLM.Service.stream`. Default runtime is Vercel AI SDK `streamText` with `tool({ execute })` (`packages/opencode/src/session/tools.ts`). Local tools therefore run **during** the provider stream, inside AI SDK's in-memory tool loop. The processor's job is to fold `LLMEvent`s into Parts, not to own tool dispatch.

That is the loop v2 deleted. Comments in `packages/core/src/session/runner/llm.ts` say so explicitly: do not rebuild the `SessionPrompt` monolith; one `llm.stream` per provider turn; record the tool call before side effects.

Native LLM (`OPENCODE_EXPERIMENTAL_NATIVE_LLM` or umbrella `OPENCODE_EXPERIMENTAL`) is an opt-in adapter over `@opencode-ai/llm`. Unsupported providers fall back to AI SDK. Title and compaction have their own streams.

### 5. Idle

When the while-loop exits, `SessionRunState` marks idle. Compaction prune may fork in the background. The last assistant message is the return value of the synchronous prompt endpoint.

A process-local busy period is one `ensureRunning` occupancy. Queued v1 prompts do not sit in `session_input`. They either join this runner or wait on it.

## SessionV2 on this same tree

Already wired. Do not confuse it with origin/v2.

`SessionV2.prompt` (`packages/core/src/session.ts`):

- Load session. Generate `msg_...` if needed.
- `SessionInput.admit` publishes `PromptAdmitted`. Projector inserts `session_input` (`admitted_seq`, `delivery`, `promoted_seq` null).
- Reusing an ID is idempotent when session, prompt, and delivery match. Conflict raises `PromptConflictError`.
- `resume !== false` calls `SessionExecution.wake`.
- Returns the admitted row. The model has not run.

`SessionExecution` is process-global and keyed by session ID. Drain start reloads the session, `LocationServiceMap.get(session.location)`, then that Location's `SessionRunner`.

`SessionRunCoordinator` joins explicit resumes, coalesces wakes, concurrent across sessions. Interrupt cancels the owner fiber. Inbox rows stay.

At a boundary (`packages/core/src/session/runner/llm.ts`): pending steers promote first; at idle, one queued item then new steers. Promotion publishes a prompted event; projector sets `promoted_seq` and inserts the visible `session_message`. Then one provider turn: exactly one `llm.stream`, record `session.tool.called` before local execute, join tools, reload projected history, continue if needed.

Shell and skill on SessionV2 currently raise `OperationUnavailableError`. Compaction there is also unavailable. The live compaction path is still v1 `SessionCompaction` + `SessionPrompt.loop`.

`AGENTS.md` "V2 Session Core" describes this stack and says not to bridge through `SessionPrompt.loop`. Production TUI still bridges through `SessionPrompt.loop`. Trust the HTTP handlers.

## Streaming

V1 processor (`packages/opencode/src/session/processor.ts`) maps `LLMEvent` to mutable Parts:

- Text and reasoning: create/update the Part, then `Session.updatePartDelta` (`message.part.delta`) for live SSE.
- Tool calls: Part status `pending` → `running` → completed/error. AI SDK `execute` is what actually runs the tool.
- Errors: `session.error`.
- Doom-loop: three identical recent tool calls ask permission `doom_loop`.

Deltas are live events. The Part row is also updated, so reconnect can read the truncated-so-far text from SQLite. That is coarser than v2's "ephemeral deltas never touch SQLite."

SessionV2 publisher (`packages/core/src/session/runner/publish-llm-event.ts`) is the other mapping: durable full text/reasoning/tool input, ephemeral deltas. It exists on this tree for the `/api` runner, not for the TUI processor.

Tool-call IDs are per assistant message. Fold by message ID plus call ID.

## Agents

`Agent.Service` is directory-scoped (`packages/opencode/src/agent/agent.ts`). Default ID is `build` (config `default_agent`, else first visible primary).

Built-ins:

- `build`. Visible primary. Default. Question and plan_enter allowed.
- `plan`. Visible primary. Denies most edits; allows plan files under `.opencode/plans` and `~/.local/share/opencode/plans`. Denies `task.general`.
- `general`. Subagent. Denies `todowrite`.
- `explore`. Subagent. Read/search/bash/webfetch/websearch. Prompt from `prompt/explore.txt`.
- `compaction`, `title`, `summary`. Hidden primaries with deny-all tools.

`mode` is still a first-class field: `primary` | `subagent` | `all`. Config `mode` tables fold into `agent` at load. Hidden only affects picker fallback.

Tool access is a permission ruleset, last match wins, plus a legacy `tools` boolean map that is rewritten into permission at config load (`write`/`edit`/`patch` → `edit`).

`agent.prompt` *replaces* the default provider system prompt (`LLMRequestPrep.prepare`). It does not append. Environment, AGENTS.md, MCP, and skills still layer after it.

Session stores `agent` and `model` on the user message more than on the session row. Switching agent in the UI without sending a new user model can keep the old model. Subagent task tool does `cmd.model ?? agent.model ?? parent.model` itself.

`temperature` / `topP` on the agent are real on this tree. They go into AI SDK / native request prep. That is a difference vs the v2 distill, which called per-agent sampling largely inert.

Core also has `packages/core/src/plugin/agent.ts` for the SessionV2 Location-scoped agent service. TUI does not use it.

## Providers and models

Catalog for the TUI is `packages/opencode/src/provider/provider.ts` on top of Core `ModelsDev` (`https://models.opencode.ai`, bundled snapshot, refresh). AI SDK packages are listed in `packages/opencode/package.json`.

Three questions, less formally separated than v2:

1. Catalog. models.dev plus config provider overlays.
2. Auth. `~/.local/share/opencode/auth.json` (oauth / api / wellknown). Env keys. Plugin auth hooks (Copilot, Codex, Azure, ...).
3. Enable/disable. `enabled_providers` / `disabled_providers`. Not the later user-global-outranks-repo policy engine, though Core `Policy` already exists for SessionV2.

`Provider.getLanguage` returns an AI SDK language model. Native runtime is gated. `aisdk` catalog entries on the Core side are a SessionV2/catalog concern; the TUI path is "whatever `Provider` constructed."

Durable v1 model identity on a user message is `{ providerID, modelID, variant? }`. Persist that, not the AI SDK model's live id.

`small_model` is still a config field. Title/summary use the small model when set. Core v1→v2 config migration later turns that into a hidden `title` agent.

## Tools, permissions, MCP

`ToolRegistry` is directory-scoped (`packages/opencode/src/tool/registry.ts`). Built-ins: read, write, edit, apply_patch, glob, grep, bash (tool id stays `"bash"` on this tree; v2 later calls it `shell`), webfetch, websearch, question, skill, task, todowrite, lsp, plan_enter/plan_exit, invalid. Experimental `code-mode` (`execute`) is off unless flagged. Question is gated to app/cli/desktop (or a flag).

A tool is omitted from the model snapshot when `Permission.disabled` sees a deny on pattern `*`. Resource-specific denies still advertise the tool. Execution does the real `Permission.ask`.

Hook order on the TUI path: `tool.execute.before` → `item.execute` → `tool.execute.after` → image normalize → truncation (2,000 lines or 50 KiB, prefix kept, rest in `~/.local/share/opencode/tool-output/`, 7-day retention). `tool_output` config can override limits. Truncate can keep head or tail.

MCP tools register after discovery. Permission action is the tool name. Extra MCP resource tools: `list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource`.

Permissions (`packages/opencode/src/permission/index.ts`):

- Last matching rule wins. No match means `ask`.
- `ask` publishes `permission.asked` and blocks on a Deferred. UI replies `once` / `always` / `reject`.
- `always` is **in-memory** on this process's `InstanceState.approved`. It is not the SQLite `permission` table the v2 line uses for saved always. Restart forgets it.
- `reject` without feedback is `RejectedError`, rejects every other pending permission in that session, interrupts.
- `reject` with feedback is `CorrectedError`. The model sees a tool failure and may continue.

`OPENCODE_PERMISSION` JSON merges into config permission.

### Subagents

The `task` tool (`packages/opencode/src/tool/task.ts`), not `subagent`. Creates or resumes a child session. Child gets `parentID`. Foreground waits on nested `SessionPrompt.prompt`. Background (`background=true`) needs `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS`, returns immediately, `BackgroundJob`, completion later injected onto the parent.

Parent and child have different session IDs and different `SessionRunState` runners, so they are concurrent. A fork copies messages and remains a top-level session.

SessionV2's `subagent` tool is a Core registry item for the other loop.

## Plugins

Public v1 API: `packages/plugin/src/index.ts` `Hooks`. Loaded by `packages/opencode/src/plugin/index.ts`.

Activation: internal auth plugins (Codex, Copilot, Modal, GitLab, Poe, Cloudflare, Azure, DigitalOcean, xAI, Cerebras, Snowflake) then config `plugin` specs and `.opencode/plugin(s)` files. TUI also has a separate plugin host (`createLegacyTuiPluginHost`) that is UI-only.

Useful v1 hooks:

| Hook | When |
| --- | --- |
| `auth` | Provider login flows |
| `provider.models` | Overlay models.dev |
| `chat.message` | Mutate user message/parts before save |
| `chat.params` / `chat.headers` | Sampling and HTTP headers |
| `tool.execute.before` / `.after` | Mutate or observe a local tool call |
| `command.execute.before` | Slash commands |
| `shell.env` | Extra env for bash |
| `experimental.chat.messages.transform` | Rewrite history before the model |
| `experimental.chat.system.transform` | Rewrite system prompt |
| `config` | Mutate loaded config |
| `event` | Observe GlobalBus-shaped events |

`permission.ask` is declared on the v1 Hooks type and is **not** triggered. Runtime `Permission.ask` publishes `permission.asked` events instead.

Core's Effect plugin SDK (`packages/plugin/src/v2`, `packages/core/src/plugin`) is what `KiloPlugin` and SessionV2 catalog transforms use. V1 config plugins do not automatically become those hooks.

## Storage

One SQLite database per process graph, shared by all projects. Schema lives in `packages/core/src/**/*.sql.ts`. Migrations are applied by Core.

Global paths (`packages/core/src/global.ts`): data `~/.local/share/opencode`, config `~/.config/opencode`, state `~/.local/state/opencode`, cache `~/.cache/opencode`.

Important tables:

- `session`. Identity, directory, workspace, parent, agent/model, tokens, revert, permission JSON. Shared by both loops.
- `message` / `part`. V1 transcript. JSON columns. Mutable Parts.
- `session_message`. V2 projected transcript.
- `session_input`. V2 pending work. Columns: `id`, `session_id`, `prompt`, `delivery`, `admitted_seq`, `promoted_seq`, `time_created`. Pending means `promoted_seq IS NULL`.
- `session_context_epoch`. V2 instruction snapshot.
- `todo`.
- `event` / `event_sequence`. Durable EventV2 payloads **always** persist on this tree. There is no `Bus.configured({ persist: false })` flag here.
- `project`, `workspace`, `worktree`, `credential` (Core; TUI auth is still `auth.json`).

This is not "rebuild the world from the log" for the TUI. The TUI reads `message` / `part`. EventV2 projectors update SessionV2 rows at commit.

JSON files that are still live: `auth.json`, `mcp-auth.json`. Snapshots are Git trees under `<data>/snapshot/<project-id>/`. Shell/tool truncation under `<data>/tool-output/`. Plans under `<data>/plans/`. SDK `createOpencodeServer` uses the CLI disk DB, not `:memory:`.

`packages/opencode/src/storage` JSON layout is leftover for transcripts. Revert still writes `session_diff` files there. Do not look there for sessions.

## Config

Two systems, both live.

**Opencode config** (`opencode.json` / `opencode.jsonc`), directory-scoped, `packages/opencode/src/config/config.ts`. Lowest to highest in `loadInstanceState`:

1. Well-known remote configs from `auth.json` entries of type `wellknown`
2. Global `~/.config/opencode/{config.json,opencode.json,opencode.jsonc}` (legacy TOML `config` migrates into `config.json`)
3. `OPENCODE_CONFIG`
4. Ancestor `opencode.json(c)`, farthest to nearest (unless `OPENCODE_DISABLE_PROJECT_CONFIG`)
5. `.opencode/` directories (global config dir, ancestors, `OPENCODE_CONFIG_DIR`): nested `opencode.json(c)`, commands, agents, modes, plugin files
6. `OPENCODE_CONFIG_CONTENT`
7. Active console org config, if logged in
8. Managed dir `opencode.json(c)` (`ConfigManaged`)
9. macOS MDM preferences (highest document merge)
10. `mode` tables folded into `agent`
11. `OPENCODE_PERMISSION` JSON
12. `tools` booleans rewritten into `permission`

`{env:VAR}` and `{file:path}` substitute before parse. `ConfigV2Compat` accepts some v2-shaped fields and reports diagnostics; the running TUI still thinks in v1 names (`plugin` tuples, `tools`, `small_model`, `mode`, `attachment`, `snapshot`).

Core `packages/core/src/config.ts` is the Location-scoped v2 parser (native v2 fields, `ConfigMigrateV1` in memory). SessionV2 uses that. The TUI uses `packages/opencode/src/config`.

**TUI config** (`packages/opencode/src/config/tui.ts`, migrated from `tui.json` / keybinds / theme). UI preferences. The server does not need it to run a prompt.

## Instructions

Not the same thing as `agent.prompt`.

Request assembly (`LLMRequestPrep.prepare`):

1. `agent.prompt` or `SystemPrompt.provider(model)` (per-family default txt)
2. Environment block (cwd, worktree, git, platform, date, references)
3. `Instruction.system()` — AGENTS.md / CLAUDE.md / CONTEXT.md, global and project
4. MCP server instructions
5. Skill catalog text
6. Optional user.system on the message
7. Plugin `experimental.chat.system.transform`

`Instruction` (`packages/opencode/src/session/instruction.ts`) reads:

- Global `~/.config/opencode/AGENTS.md` and optional `~/.claude/CLAUDE.md`
- Project `AGENTS.md`, `CLAUDE.md`, deprecated `CONTEXT.md`
- Config `instructions` globs/URLs (this field **is** loaded on the TUI path)
- After a successful `read`, nested AGENTS.md along loaded paths can attach to the current assistant message (claimed per message ID so it does not spam)

There is no content-addressed `instruction_blob` on the TUI path. That algebra is SessionV2 (`packages/core/src/system-context`, `session_context_epoch`).

`CLAUDE.md` fallback **is** implemented here unless `disableClaudeCodePrompt` is set. The v2 distill says the opposite for origin/v2.

## How the pieces connect

```text
CLI / Desktop / TUI / App / ACP / SDK
        |              |           |
        | Worker-RPC   | HTTP+SSE  | ndjson
        | or HTTP      |           |
        v              v           v
     packages/opencode Server
        |
        +-- /session/:id/prompt[_async]     +-- /api/session/:id/prompt
        |   InstanceState (per directory)   |   LocationServiceMap
        |   SessionPrompt.loop              |   SessionV2.prompt
        |   SessionProcessor + AI SDK       |   SessionInput.admit
        |   opencode tools/agents/plugins   |   SessionExecution.wake
        |                                   |   SessionRunner llm.stream
        |
        +-- SQLite (everyone)
              session, message, part,
              session_input, session_message,
              event, event_sequence, project
```

A Kilo product that wants today's OpenCode UX should treat `packages/opencode` as the host and `SessionPrompt.loop` as the loop. A Kilo that wants the inbox/runner should read origin/v2, not this file's SessionV2 sidebar — this tree's SessionV2 is mid-migration (shell/skill/compact still `OperationUnavailableError`). A Kilo that only needs a provider should use a v1 `provider.models` / auth plugin, or the Core catalog transform, not a new process.

## What changed vs origin/v2 (visible from this tree)

The v2 distill already lists the inverse. From this side:

**Process.** TUI Worker + JSON RPC is the default. Network `serve` is the special case. Yargs. Monolith `packages/opencode` still exists and still owns the loop. `packages/cli` / `sdk-next` are present but not the product path. Desktop sidecar is a forked opencode server, not a shared authenticated daemon election.

**Session loop.** Prompts are visible first. In-memory `SessionPrompt.loop`. AI SDK executes tools during `streamText`. Coarse `message.part.delta` / `part.updated`. SessionV2 inbox exists in-process on `/api` but is not what you get from the TUI.

**Storage.** Sessions are already SQLite, not JSON files. Dual transcript tables. Event payloads always persist. Auth is still `auth.json`. Saved "always" permissions are in-memory.

**Config.** `tools` booleans, `plugin` tuples, `small_model`, `mode`, `attachment`, `snapshot` are live names. Core can migrate them for SessionV2. TUI still loads v1.

**HTTP.** Effect HttpApi replaced Hono *inside* `packages/opencode`, and a second Protocol API is mounted at `/api`. Generated v2 clients exist. TUI `session.prompt` still hits `/session/:id/message`.

**Plugins.** V1 Promise hooks still run. Core Effect plugins also run for catalog/SessionV2. Two SDKs in one repo.

## Gotchas

- `AGENTS.md` on this tree describes SessionV2 as the session core. The TUI does not use it. Trust `SessionHttpApi.prompt` / `promptAsync`.
- `/session/:id/message` (and `prompt_async`) vs `/api/session/:id/prompt` are different loops that share a `session` row. Do not mix clients casually.
- Worker TUI is not an in-process function call. It is RPC-to-`fetch` into the same Effect handler graph.
- `opencode serve` without a password is unsecured. Username is not hardcoded.
- `always` permission does not survive restart on the TUI path.
- `SessionRunState.assertNotBusy` is why revert/summarize fail during a run. Prompt itself joins the runner.
- Native LLM is opt-in. Default is AI SDK, including tool execution inside the stream.
- `packages/cli` is not `opencode`. `packages/sdk-next` is not `@opencode-ai/sdk`.
- Core `KiloPlugin` is a header stamp. Ignore it for a real port.
- `oh-my-opencode` is an external ecosystem name, not code in this repo.
- `mode` vs `agent`: both exist. `mode` config merges into agents.
- `task` is the v1 subagent tool. `subagent` is the v2 name. The shell tool id is still `bash`.
- The TUI composer posts `/session/:id/message`, not `/session/:id/prompt` and not `/api/session/:id/prompt`.
- Config `instructions` reach the TUI model. Nested AGENTS.md via `read` also exists here.
- `CLAUDE.md` is loaded unless disabled.
- EventV2 `owner_id` is replay ownership, not session execution claims. V1 has no execution claim.
- Separate processes can open the same channel DB. WAL plus a 5s busy timeout make that survivable. It is not clearly a supported workflow.

## Where to start reading

In this order:

1. This file, then `docs/v2-architecture.md` on PR #1, section for section.
2. `packages/opencode/src/cli/cmd/tui.ts` and `cli/tui/worker.ts`. Process topology.
3. `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts`. HTTP boundary for the TUI.
4. `packages/opencode/src/session/prompt.ts` (`prompt`, `run`, `loop`). The live loop.
5. `packages/opencode/src/session/processor.ts` and `session/tools.ts`. Streaming and in-stream tools.
6. `packages/opencode/src/session/llm.ts` plus `session/llm/AGENTS.md`. AI SDK vs native.
7. `packages/opencode/src/agent/agent.ts`, `permission/index.ts`, `plugin/index.ts`, `config/config.ts`.
8. Only then `packages/core/src/session.ts` → `session/input.ts` → `execution/local.ts` → `runner/llm.ts`, to see the inbox already growing beside the loop you just read.

If `AGENTS.md` "V2 Session Core" or `specs/v2/session.md` disagree with `SessionPrompt.loop`, the loop is what users hit on this ref.

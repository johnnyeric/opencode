# OpenCode v2 architecture

Reading notes for porting another coding agent onto this tree. Not a product pitch, not a backlog.

The tree this was read from is `origin/v2` at `e70d667a9fe3e84cc071a5596aa522c142c525b7` (`fix(ai): preserve Anthropic finish across usage deltas`, 2026-08-29). Fetched from this fork's origin. That is the live v2 line.

This fork's `dev` is a diverged older snapshot. It still has `packages/opencode`, talks about `SessionV2.prompt` and `session_input`, and uses "provider turn" language the v2 branch explicitly retired. Do not read `dev` for v2.

## Overview

OpenCode v2 is a split-process coding agent. A long-lived server process owns SQLite, sessions, tools, and model streaming. TUI, desktop, and the web app are HTTP/SSE clients of that server. The SDK can embed the same server graph in-process behind a fake `fetch`.

The product idea that actually changed from v1 is the session loop. A prompt is admitted to a durable inbox first. A process-local runner later delivers it into visible history, calls the model once per physical attempt, records tool calls before running them, and reloads projected history before the next step. There is no in-memory tool loop.

If you are porting Kilo, the reusable core is `packages/core` plus `protocol` / `server` / `client`. The existing `KiloPlugin` in `packages/core/src/plugin/provider/kilo.ts` only stamps `HTTP-Referer` and `X-Title` onto a matching gateway URL. It is not a port.

## What to ignore on Monday

Skip these unless you are specifically shipping that product:

- `packages/console`, `enterprise`, `function`, `stats`, `updates`, `containers`. Hosted billing, cloud functions, CI images.
- `packages/www` and `packages/web`. Docs and marketing sites. `www` is the current one. `web` is leftover translations.
- `packages/posts`, `packages/identity`. Blog and two SVGs.
- `sdks/vscode`. Opens a terminal and posts `/tui/append-prompt`. Not an execution engine.
- Tauri leftovers. Desktop is Electron now.
- `packages/simulation`. Replay harness. Server can install it through `ServerOptions.simulation`. Normal runs do not.
- `packages/merman`, `latex`, `theme`. TUI renderer details. Read them only if you are keeping the TUI.
- `packages/core/src/session/sql.ts` `session_pending`. Declared, unused. Inbox is `session_inbox`.
- `specs/v2/catalog-config-plugin-lifecycle.md`. Historical option comparison. Option B shipped. The sketched APIs did not.
- User-facing docs that mention `OPENCODE_SERVER_USERNAME`. Production auth hardcodes username `opencode`.

Do not skip `packages/core/src/v1`. It is live config compatibility, not a graveyard.

## Package map

Runtime dependency direction, from `AGENTS.md`:

```text
schema
  -> ai, protocol
       protocol -> client
       ai + plugin + schema -> core
         core + protocol -> server
           client + core + server -> sdk
           server + client + tui -> cli
```

Client runtime code may depend on Schema and Protocol. It must not depend on Core or Server. `sdk` is allowed to compose all three.

| Package | Job |
| --- | --- |
| `schema` | Domain types, branded IDs, durable event payloads |
| `ai` | Provider-neutral LLM stream, protocols, native provider packages |
| `protocol` | Effect `HttpApi` contracts, paths, errors, SSE event union |
| `client` | Generated Promise and Effect HTTP clients, daemon discovery, Solid SSE |
| `plugin` | Public Effect and Promise plugin SDKs |
| `core` | Sessions, inbox, runner, tools, catalog, config, SQLite, plugins |
| `server` | Binds Protocol to Core, Node listener or `fetch` handler |
| `sdk` | Embeds Server+Core in-process, in-memory `fetch` |
| `cli` | Effect CLI. Spawns or attaches to the server. Not the monolith. |
| `tui` | Terminal UI over HTTP/SSE. Imports Core for frontend bits, not for the runner. |
| `app` / `session-ui` / `ui` | Solid web/desktop UI |
| `desktop` | Electron shell. Discovers the same daemon. Not an HTTP proxy. |

`packages/opencode` is gone. So is `packages/sdk-next`. Their replacements are the rows above.

## Key concepts

**Location.** `{ directory, workspaceID? }`. Cache key for one runtime instance. Omitted `workspaceID` means the local filesystem. Explicit workspace identity is reserved for remote drivers.

**Process-global vs Location-scoped.** One server process has one SQLite database, one `Bus`, one `SessionExecution`, one `LocationServiceMap`. Agents, catalog, tools, permissions, plugins, filesystem, and `SessionRunner` are built per Location and cached.

**Inbox vs transcript.** `session_inbox` holds unconsumed work. Visible conversation lives in `session_message`. Delivery deletes the inbox row and inserts the message in one transaction.

**Step vs physical attempt vs turn.** One physical attempt is one `llm.stream(...)` call. A logical step is one model-visible call and may retry or rebuild without incrementing the step number. "Turn" is reserved for a future unit covering every step from prompt promotion until idle. Do not write "provider turn."

**Steer vs queue.** Steer (default) delivers at the next safe step boundary while work continues. Queue waits until the session would otherwise go idle, then delivers one item. Steers always win at an idle boundary.

**Claim.** `session_v2.time_suspended` marks a process-local busy period for restart recovery. It is not clustered ownership, fencing, or proof the process is alive.

**Bus.** Process-global publication. Projectors run inside the SQLite transaction. Live SSE is notified after commit. Event *payload* retention is optional and off by default on the local CLI.

## Process model

### How a user actually starts this

`packages/cli/src/index.ts` is the binary. Root command is the full TUI. `ServerConnection.resolve` in `packages/cli/src/services/server-connection.ts` picks one of three topologies:

1. `--server URL`. Attach. Basic auth user is always `opencode`. Password from `OPENCODE_PASSWORD`, with `OPENCODE_SERVER_PASSWORD` as fallback. Version mismatch warns and continues.
2. `--standalone`. Private child process, `serve --stdio --port 0`. Parent owns the child by holding stdin. This is not an in-process server.
3. Default. Managed daemon. Registration file plus authenticated `/api/health`. `Service.ensure` elects or starts `opencode2 serve --service`. Desktop uses this same path.

`serve` turns the current process into the server. `--service` registers as the daemon. `--stdio` dies when stdin closes.

The only normal in-process full server is `@opencode-ai/sdk` or `ServerFetch`. The TUI is an HTTP client of the runner.

### Server boot

`packages/cli/src/server-process.ts` builds `ServerOptions` (database filename, config sources, catalog, watcher, PTY handoff) and calls `ServerProcess.start`.

The server binds the socket *before* the Core graph finishes. Authenticated `/api/health` can return 503 during boot. Password is required for the network server. CLI generates one if needed.

`createRoutes` in `packages/server/src/routes.ts` builds the process-global graph, then `HttpApiBuilder` binds Protocol endpoints to `packages/server/src/handlers.ts`.

One daemon hosts many directories through `LocationServiceMap` and one SQLite file. Default DB is `~/.local/share/opencode/opencode.db` (`OPENCODE_DB` overrides; custom channels use `opencode-<channel>.db`).

### Request path for a prompt

```text
TUI/app/desktop
  -> generated client POST /api/session/:id/prompt
    -> Protocol endpoint + SessionLocationMiddleware
      -> SQLite session row -> Location.Ref
        -> LocationServiceMap.get(ref)
          -> Session.Service.prompt
            -> SessionInbox.admit
            -> SessionExecution.wake   (unless resume:false)
          <- SessionInbox.User (does not wait for the model)
```

Session routes do not trust the request's current directory. Middleware reads the session's recorded Location from SQLite. `session.create` is different. It is global, takes Location in the payload, and defaults to server cwd because no row exists yet.

### Events to the UI

Core publishes on `Bus`. `EventFeed` listens once, JSON-encodes, and fans out to per-connection dropping queues of 4096 frames. `/api/event` prepends `server.connected`, then heartbeats every 15 seconds.

SSE is volatile. No replay. Clients rehydrate with ordinary API reads after reconnect. `/api/event` is server-global. UIs filter on `event.location`.

Desktop Electron main talks to the renderer over Effect RPC for native bits and credentials. Ordinary OpenCode API traffic goes HTTP from the renderer to the daemon. WSL is a separate `opencode serve` inside the distro, treated as another sidecar URL.

ACP (`opencode acp`) starts a standalone child server and speaks newline JSON on the CLI process stdin/stdout.

## A prompt from click to idle

This is the part to internalize.

### 1. Admit

`Session.prompt` in `packages/core/src/session/session.ts`:

- Load the session. Generate `msg_...` if no ID given.
- `SessionInbox.reconcile` first. An already-admitted ID skips hooks and file reads.
- For new input, Location-scoped `SessionPrompt.prepare` runs `session.prompt` hooks, materializes attachments, resolves skills.
- A staged revert commits only after preparation succeeds.
- `SessionInbox.admit` publishes `session.inbox.enqueued`. Projector inserts `session_inbox`.
- `SessionExecution.wake` unless `resume === false`.
- Return the inbox item. HTTP returns here. The model has not run.

Identity: reusing a session ID adopts the session. Reusing a user/synthetic inbox ID is idempotent when session and type match. First payload wins. Cross-session or cross-type reuse fails. `resume` is not part of identity, so a retry can wake without inserting another row.

### 2. Wake and drain

`SessionExecution` is process-global and keyed only by session ID. At drain start it reloads the session, resolves `LocationServiceMap.get(session.location)`, and calls that Location's `SessionRunner`.

`SessionRunCoordinator` keeps one process-local execution object per session:

- `run` (explicit resume) joins or starts with `force=true`.
- `wake` starts with `force=false`, or doorbells an active run. `"input"` subsumes `"steer"`.
- Different sessions run concurrently.
- Interrupt clears recorded wakes and cancels the owner fiber. Inbox rows stay.

A write-ahead claim is set on `session.execution.started`. Success, failure, and user interrupt release it. Shutdown interrupt and process death keep it. Startup recovery (`session/execution/restart.ts`) resumes claimed *top-level* sessions, appends a synthetic "server restarted" message, and caps at 10 attempts per claim. Recovery is at-least-once. Stale in-flight tools are failed, never replayed.

### 3. Boundary processing

`SessionRunnerLLM.drain` in `packages/core/src/session/runner/llm.ts`:

1. Flush plugins. Fail tools still projected as running from a dead process.
2. Peek the next eligible inbox item under the inbox lock.
3. Compaction and move controls go first. They are barriers. A steer batch stops before them.
4. Reset the logical step count when new queued work becomes the intent.
5. `SessionContext.select` resolves agent, tools, instruction sources. Persist instruction deltas. If the *initial* instruction baseline cannot be observed, stop with the prompt still pending.
6. Deliver eligible inbox items. Delivery publishes `session.inbox.delivered`. Projector deletes the inbox row and inserts the visible message.
7. `SessionContext.load` reloads history so the new messages are included.
8. Run a logical step.

Steer-vs-queue at the boundary: while the drain still needs continuation, only steers deliver. At idle, all current steers deliver, else exactly one queued item, then any steers that arrived during that delivery. Then reevaluate before another queue item.

### 4. One step, one or more physical attempts

`runStep` owns a logical step. `SessionStep.attempt` owns a physical attempt. Most steps are one attempt. Extra attempts without consuming another step allowance:

- generic retry before durable output (jittered 2s exponential, max 4)
- one full-context rebuild after continuation-state rejection
- incomplete-stream continuation after partial output (fails the partial assistant, appends a synthetic continue instruction, new assistant ID, same step)
- one overflow-compaction rebuild (second overflow is terminal)

Each physical attempt contains exactly one `llm.stream(...)` in `packages/core/src/session/runner/step.ts`. Title generation and compaction have their own streams. Those are not extra calls inside the session attempt.

At `step >= agent.steps`, the runner keeps tool definitions for prefix caching, sets `toolChoice: "none"`, and appends `MAX_STEPS_PROMPT`.

### 5. Tools during the stream

For each non-hosted `tool-call` event:

1. Publish durable `session.tool.called` first.
2. Fork local execution immediately. Provider streaming continues.
3. Multiple local tools may run concurrently.
4. After the provider stream ends, publish `session.step.streamed` (stream ended, tools may still be running).
5. Join all local tools. Fail missing hosted results.
6. Only then publish the step's terminal ended/failed event.

`session.step.streamed` does not mean the assistant is complete.

If any local tool ran, `needsContinuation` is true. The next step reloads projected history. The runner does not keep an in-memory transcript of tool results.

Hosted tools (`providerExecuted: true`) never enter `Tool.Service` or local permissions. OpenAI Responses and Anthropic server tools emit call and result from the protocol layer.

### 6. Idle

When `needsContinuation` is false and no eligible inbox remains, drain returns complete. If the doorbell is quiet, execution publishes `session.execution.succeeded`, releases the claim, and the projector stamps the idle watermark.

A process-local busy period can absorb several prompt intents if queued work keeps landing at idle boundaries. One `session.execution.started` / terminal pair covers that whole busy period.

## Streaming

`createLLMEventPublisher` in `packages/core/src/session/runner/publish-llm-event.ts` maps `LLMEvent` to session events.

Durable: step started/streamed/ended/failed, full text, full reasoning, full tool input, tool called, tool success/failure, retry scheduled, compaction started/ended/failed.

Ephemeral (SSE only, not SQLite): text delta, reasoning delta, tool progress, compaction delta, usage-updated. Text and reasoning deltas batch at 100ms. First delta starts the interval. End flushes then publishes the durable full value.

Tool-input fragments accumulate in memory. The publisher emits `Started` and full `Ended` only. `session.tool.input.delta` exists in the schema and is not produced.

`SessionMessageUpdater` folds durable events into one assistant row. Ephemeral deltas never touch SQLite. Orderly interrupt flushes fragments into durable `Ended` events. A hard crash can lose unflushed live text.

Tool-call IDs are unique only within a step. Every durable tool event also carries `assistantMessageID`. Fold by assistant ID, tool ID, and content ordinal. Do not assume a global interleaving of provider vs tool events.

## Agents

`Agent.Service` is Location-scoped (`packages/core/src/agent.ts`). Default ID is `build`. `resolve(undefined)` picks configured default, then visible `build`, then first visible non-subagent. Hidden and subagent flags only affect that fallback. `select(id)` will run a named agent even if it is hidden or a subagent.

Built-ins from `packages/core/src/plugin/agent.ts` and `plugin/plan.ts`:

- `build`. Visible primary. Default.
- `plan`. Visible primary. Denies edits except under `~/.opencode/plan`.
- `general`. Subagent. No `question`, no nested `subagent`.
- `explore`. Subagent. Read/search only.
- `compaction`, `title`, `summary`. Hidden primaries with deny-all tools.

There is no v2 agent `tools` map. Tool access is permission rules. Last match wins. Config documents append rules, later documents win.

**Trap.** Session stores `agent` and `model` independently. `Session.switchAgent` only publishes `AgentSelected`. `SessionRunnerModel.resolve` reads `session.model`, not `agent.model`. The subagent tool does `agent.model ?? parent.model` itself. A UI that switches agent without switching model will keep the old model. Clients must coordinate both.

`agent.system` *replaces* the default system prompt. It does not append. Instructions (AGENTS.md, skills, MCP, entries) still layer on after it.

`Agent.Info.request` (headers/body/settings) is merged from config and then largely ignored by Core request preparation. V1 migration stuffed `temperature` / `top_p` into `agent.request.body`. Do not rely on per-agent sampling until that wiring is real.

## Providers and models

Catalog is Location-scoped (`packages/core/src/catalog.ts`). Baseline is models.dev (`https://models.opencode.ai`), bundled snapshot at boot, refresh every five minutes (`packages/core/src/models-dev.ts`).

Three separate questions:

1. Catalog. What exists, including config overlays.
2. Integration. What is connected (env keys, SQLite `credential`, OAuth). Saved credentials beat env.
3. Policy. What may be used. `specs/v2/provider-policy.md`. User-global policy outranks repository policy, on purpose, and runs last.

`ModelResolver` applies variant overlays, resolves credentials, merges settings/headers/body, expands `${ENV}` in endpoints, then loads a native `@opencode-ai/ai` provider or an AI SDK adapter.

`aisdk:` in the catalog does not guarantee the AI SDK path. `AISDKNative.map` rewrites known packages onto native providers. OpenAI-compatible with a string `baseURL` goes native. That is the Kilo path today, so `ctx.aisdk` hooks will not run for it.

Durable model identity is `{ providerID, id, variant? }`. The ID sent to the provider can differ (`modelID`). Persist `Resolved.ref`, not `LanguageModel.id`.

`Model.Info.capabilities.tools` is shown to UIs. Core does not currently strip tools when it is false.

## Tools, permissions, MCP

`Tool.Service` is Location-scoped. Later plugin transforms win on name collision. A tool is omitted from the model snapshot only when the last matching permission rule is deny on resource `"*"`. Resource-specific denies still advertise the tool. Execution does the real check.

Built-ins all set `codemode: false` (read, write, edit, patch, glob, grep, shell, webfetch, websearch, question, skill, subagent). Plugin and MCP tools default *into* Code Mode, a confined JS sandbox that can call catalogued tools. Nested calls still hit `tool.execute.before`, validation, and permissions.

Hook order: `tool.execute.before` → decode input → execute → encode output → `tool.execute.after` → image normalize → `ToolOutput` truncation (2,000 lines or 50 KiB, prefix kept, rest in `~/.local/share/opencode/tool-output/`, 7-day retention). Specs still describe head-plus-tail. The code keeps a prefix only. `execute.after` cannot see the truncated file.

MCP tools register under a sanitized server namespace after discovery. Permission action is the flattened server/tool name.

Permissions (`packages/core/src/permission.ts`):

- Last matching rule wins. No match means `ask`.
- Configured deny is checked first and cannot be overridden by saved "always" or by `permission.evaluate` hooks.
- `ask` publishes `permission.asked` and blocks on a Deferred. UI replies `once` / `always` / `reject`.
- `always` writes project-scoped rows in `permission`.
- `reject` without feedback raises `DeclinedError`, rejects every other pending permission in that session, and interrupts the step.
- `reject` with feedback raises `CorrectedError`. The model sees a tool failure and may continue.

V1 permission names in config are rewritten: `bash` → `shell`, `task` → `subagent`, `write`/`patch` → `edit`.

### Subagents

The `subagent` tool creates or resumes a child session through the same facade. Default nesting depth is 1. It rejects `mode === "primary"`. Child gets `parentID`, inherits Location, model is `agent.model ?? parent.model`.

Foreground: parent tool fiber waits on `session.resume(child.id)`. Background: returns immediately, Job marker, completion later admitted as synthetic input on the parent.

Parent and child have different session IDs, so they are concurrent coordinator executions. A fork is not a subagent. Forks copy messages, adopt the parent's newest instruction values, and remain top-level recoverable sessions with `parent_id` null.

## Plugins

Public API: `packages/plugin/src/effect` and `packages/plugin/src/promise`. No `src/v2` directory on this branch.

Activation order (`plugin/supervisor.ts`): internal pre → SDK host plugins → instance plugins → config package/file plugins → internal post (config overlays and terminal provider policy). Duplicate IDs fail. Load failure rolls back to the previous generation.

Useful hooks for a port:

| Hook | When |
| --- | --- |
| `catalog.transform` | Static provider/model overlays. This is what `KiloPlugin` uses. |
| `session.prompt` | Mutate prompt, metadata, delivery before admission |
| `session.context` | Mutate system, messages, advertised definitions, generation options. Can rename/remove definitions. Cannot invent executables. |
| `session.model.request` | Rewrite base URL and headers |
| `session.http.request` / `.response` | Replace the web Request/Response |
| `session.retry` | Alter retry decision/delay |
| `permission.evaluate` | After configured deny has already been applied |
| `tool.execute.before` / `.after` | Mutate or reject a local tool call |
| `aisdk.sdk` / `aisdk.language` | Only on the AI SDK fallback path |

If Kilo needs protocol changes, write a native provider under `packages/ai`. If it needs auth/account flows, use integration transforms. If it needs a product host, use `@opencode-ai/sdk` or replace `packages/cli`. Do not try to become OpenCode by stacking hooks.

## Storage

One SQLite database per application graph, shared by all projects.

Global paths (`packages/util/src/global.ts`): data `~/.local/share/opencode`, config `~/.config/opencode`, state `~/.local/state/opencode`, cache `~/.cache/opencode`.

Important tables, all in `packages/core/src/**/*.sql.ts`:

- `session_v2`. Identity, location, selected agent/model, usage, revert, `time_suspended` claim.
- `session_message`. Transcript projection, ordered by aggregate sequence.
- `session_inbox`. Pending work only.
- `instruction_blob`, `instruction_state`, `instruction_entry`. Content-addressed JSON and per-session epoch hashes.
- `event_sequence`. Always. Latest aggregate sequence.
- `event`. Payload history, only if `Bus.configured({ persist: true })`. Local CLI leaves this off. Workerd forces it on. Restart recovery does **not** need the payload table. It uses projections and the claim.
- `permission`, `credential`, `kv`, `project`, `workspace`, `worktree`.

This is not "rebuild the world from the log." Projectors run at commit. `Bus.project` registers callbacks. It does not replay old rows. Projections remain authoritative when payloads are dropped.

Attachments are inline base64 in inbox/message JSON (data/file URLs, 20 MiB cap). Snapshots are Git trees under `<data>/snapshot/<project-id>/`. Shell output lives under `<data>/shell/<project-id>/`. SDK embedded host defaults to `:memory:`. CLI always passes a disk filename.

Event replay ownership (`event_sequence.owner_id`) is a different mechanism from session execution claims. `Bus.claim` has no production caller right now.

## Config

Two systems.

**Core** (`opencode.json` / `opencode.jsonc`), Location-scoped, `packages/core/src/config.ts`. Lowest to highest:

1. Well-known remote sources
2. `.claude` then `.agents` directories
3. Global `~/.config/opencode/opencode.json(c)`
4. `OPENCODE_CONFIG`
5. Ancestor `opencode.json(c)`, farthest to nearest
6. Ancestor `.opencode/` directories and their configs, farthest to nearest
7. `OPENCODE_CONFIG_CONTENT`

Project `.opencode` config outranks every directly discovered `opencode.json`, including a closer one. `{env:VAR}` and `{file:path}` substitute before JSON parse. V1 and V2 syntax can coexist in one file. `ConfigNormalize` maps in memory and does not rewrite the file. Native v2 fields win over migrated v1 equivalents.

Different fields merge differently. Atomic values take the last document. Agents and providers overlay. Permission rules append. Provider *policy* reverses document order so the user-global file can ban a repo-enabled provider.

Parsed but currently inert: config `instructions` (local paths/globs/URLs do not reach the model) and `share`. Ambient instructions come from `AGENTS.md` discovery, not that field. `CLAUDE.md` fallback is not implemented.

**CLI** (`~/.config/opencode/cli.json`). TUI preferences. The daemon does not load it. V1 migrated `tui.json` and global `kv.json` into this file, never project-local TUI config.

V1 plugins listed in config are normalized and then not executed. New plugin API only.

## Instructions

Not the same thing as `agent.system`.

Request assembly (`SessionModelRequest.baseTranscript`): agent/default system prompt, then the instruction epoch baseline, then projected history lowered to AI messages.

`SessionContext.select` composes, in order: built-ins, code-mode instructions, ambient `AGENTS.md` discovery, selected-agent skill guidance, references, MCP guidance, API-managed entries. There is no instruction registry. The runner lists sources explicitly.

Values are canonical JSON, SHA-256 hashed, stored once in `instruction_blob`. Deltas publish `session.instructions.updated`. Initial observation has no chronological text. Later changes may freeze rendered text into a durable system message.

Unavailable source on first observation blocks the drain and leaves the prompt pending. After init, unavailable sources keep the last value.

Compaction moves the epoch (history for the runner starts at the latest completed compaction). Movement keeps instruction state so destination file changes show up chronologically. Committed revert clears it. Forks take the parent's newest values even if copied messages end earlier.

A second, easy-to-miss path: after a successful `read`, `SessionInstructions` searches upward for nested `AGENTS.md` and injects a durable synthetic message. That is transcript, not `instruction_state`. Compaction can make the same file eligible to inject again.

## How the pieces connect

```text
CLI / Desktop / TUI / App / ACP / SDK
        |              |           |
        | HTTP+SSE     | in-proc   | ndjson
        v              v           v
     Server  (Protocol HttpApi + EventFeed)
        |
        | process-global: Database, Bus, Session, SessionExecution,
        |                 LocationServiceMap, Workspace, Project
        |
        +-- Location A (dir=...)     +-- Location B
        |   Config, Catalog, Agent   |
        |   Tools, MCP, Permission   |
        |   Plugins, Filesystem      |
        |   SessionRunner --------+  |
        |                         |  |
        |                         v  v
        |                    packages/ai llm.stream
        |                         |
        |                    native protocol or AI SDK
        |
        +-- SQLite (everyone)
              session_v2, session_inbox, session_message,
              instruction_*, credential, permission, event_sequence
```

A Kilo product that wants to stay a separate host should take `@opencode-ai/sdk` and bring its own UI. A Kilo that wants OpenCode's daemon and UIs should keep Core/Server/Client and replace or wrap `packages/cli`. A Kilo that only needs a provider should use a catalog transform or a native `packages/ai` provider, not a new process.

## What changed vs v1 (visible in this tree)

The v2 branch does not contain the v1 runner. Comparison comes from retained schemas, `packages/core/src/v1`, `database/v1-migration.bun.ts`, `packages/cli/src/run/v1.ts`, and `origin/dev`'s leftover `packages/opencode`.

**Process.** V1 TUI started a Worker and spoke hand-written JSON RPC into `Server.Default().app.fetch`. Network mode was the special case. V2 makes HTTP/SSE the common path. Default is a shared authenticated daemon. Standalone is a child process, not a Worker. Effect CLI replaced yargs. The monolith `packages/opencode` was split into schema / protocol / core / server / client / cli / tui.

**Session loop.** V1 projected prompts into visible history immediately and drove an in-memory loop (`SessionPrompt.loop` is gone). V2 admits to `session_inbox`, delivers later, one `llm.stream` per physical attempt, reloads projections for continuation. V1 messages were User/Assistant plus mutable Parts and published coarse `session.updated` / `message.part.delta`. V2 publishes narrow facts and folds them into `session_message`.

**Storage.** V1 sessions were already SQLite (`session` / `message` / `part` with JSON columns), not JSON files. JSON files were credentials and UI prefs. Migration (`V1Migration.transformSession`, Bun only) rebuilds `session_v2` and `session_message`, advances `event_sequence`, records `kv["migration.v1-v2"]`, and leaves the old tables in place. `opencode-next.db` is a previous-v2 import source. Node and workerd use a no-op migration stub. `auth.json` imports into `credential`.

**Config.** V1 `tools` booleans and permission maps become ordered rules. `plugin` tuples become objects. `small_model` becomes the hidden `title` agent. `mode` folded into `agents`. `snapshot` → `snapshots`, `attachment` → `media`. System prompts did not vanish. They still lead the request. Ambient and dynamic context moved into the instruction algebra. Nested `AGENTS.md` via `read` is new and separate.

**HTTP.** Effect `HttpApi` replaced Hono in the OpenCode server. Hono still exists in `enterprise` and `function`. Generated clients replaced hand-written RPC. `packages/cli/src/run/v1.ts` is an output-format bridge over the v2 client, not a v1 engine.

**Plugins.** V1 plugins do not run. New Effect/Promise SDK, replayable state transforms, Location-scoped supervisor.

## Gotchas

- `--standalone` means private child, not embedded.
- Desktop "sidecar" is usually the same managed daemon the CLI discovers.
- TUI imports Core. Session traffic is still HTTP.
- Separate processes can open the same channel DB. WAL plus a 5s busy timeout make that survivable. It is not clearly a supported workflow.
- `session.active()` is this process's coordinator map. Absence means this process is idle, not that nothing is happening elsewhere.
- Direct Core `Session.interrupt` returns `false` for an unknown ID. HTTP interrupt fails through middleware with not-found.
- Instruction entry edits do not wake the runner. They apply at the next model boundary.
- Steer delivery is per-item transactional. Interrupt can leave the rest of a batch pending.
- Provider continuation checkpoints commit only after the response is fully consumed. That is why `SessionStep` reads the stream to the end.
- Native reasoning/provider metadata replay is model-sensitive. After a model switch, reasoning becomes ordinary text. Compaction drops native continuation state.
- Foreground subagents are nested executions, not a recursive call inside `drain`.
- Config `plugins` and CLI `plugins` are different classes of plugin.
- `oh-my-opencode` is an external ecosystem name, not code in this repo.

## Where to start reading

In this order:

1. `AGENTS.md` "V2 Session Core" and `specs/v2/session.md`. Contract. If they disagree with code, trust the code. Compaction estimation and tool-input deltas already disagree.
2. `packages/core/src/session/session.ts` then `inbox.ts` then `execution.ts` then `runner/llm.ts` then `runner/step.ts`.
3. `packages/server/src/routes.ts` and `handlers/session.ts`. HTTP boundary.
4. `packages/cli/src/services/server-connection.ts`. Process topology.
5. `packages/core/src/instance.ts` and `location-services.ts`. What is per Location.
6. `packages/core/src/tool.ts`, `permission.ts`, `plugin/host.ts`. The Kilo-shaped extension surface.
7. `packages/ai/src/llm.ts` and `route/client.ts` only after the runner makes sense.

`specs/v2/tools.md` and `specs/v2/event-stream-architecture.md` are useful once you have the loop in your head. Treat names in older specs as possibly stale. Current Core calls the event service `Bus`, not `EventV2`.

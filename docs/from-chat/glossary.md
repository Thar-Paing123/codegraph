# CodeGraph — Glossary

Terms used throughout the codebase and docs that aren't self-explanatory
on first read. Companion to [business-flow.md](business-flow.md) and
[architecture-overview.md](architecture-overview.md).

## Core data model

- **Node** — one symbol in the graph: a function, class, file, route, etc.
  Its kind is one of the fixed `NodeKind` strings in `src/types.ts`.
- **Edge** — a relationship between two nodes (`calls`, `imports`,
  `extends`, `references`, `instantiates`, …), one of the fixed `EdgeKind`
  strings.
- **`.codegraph/`** — the per-project directory `codegraph init` creates:
  holds `codegraph.db` (the SQLite graph) and config/trail state. Gitignored.
- **Extraction** — parsing source files with tree-sitter (or the native
  Rust kernel) into nodes + edges. Deterministic — derived from the AST,
  never LLM-summarized.
- **Resolution** — the pass after extraction that turns a bare reference
  ("this function call") into an edge to the actual definition node:
  import resolution, name matching, framework-specific patterns.
- **Synthesis / synthesizer** — a resolution-layer pass that adds edges
  tree-sitter *can't* see directly — callbacks, React re-render, cross-
  language native bridges. Always tagged `provenance: 'heuristic'`.
- **Provenance** — metadata on an edge recording how it was found:
  `'heuristic'` (synthesized/inferred) vs. a direct parse. Lets the agent
  (and a developer reading the graph) know how much to trust a given hop.
- **Route node** — a synthetic node representing a web-framework URL
  pattern (e.g. `GET /users/:id`), linked by a `references` edge to its
  handler. Produced by a framework resolver (Express, Django, Rails, …).
- **`navigates` edge** — links code that sends a user somewhere (a router
  push, a `<Link>`) to the screen/route it names. Framework-specific, on
  top of the core `EdgeKind` set.

## Querying & tools

- **`codegraph_explore`** — the single MCP tool exposed to agents by
  default. Takes a natural-language question or a bag of symbol/file
  names; returns verbatim source + call paths + blast radius in one call.
- **`codegraph_node`** — a secondary MCP tool (unlisted by default, opt-in
  via `CODEGRAPH_MCP_TOOLS`): one symbol's full source + caller/callee
  trail, or a file's line-numbered content. Returns *every* overload's
  body when a name is ambiguous.
- **Blast radius** — everything that depends on a symbol: its callers,
  transitively, up to a depth. What you'd need to check before changing it.
- **Flow / named-symbol flow** — the call path between two or more named
  symbols (e.g. "how does `mutateElement` reach `renderScene`"), including
  any dynamic-dispatch hops along the way. Computed by
  `src/graph/named-symbol-flow.ts`.
- **Explore budget** — the cap on how many `codegraph_explore` calls and
  how much output per call a session gets, scaled by indexed file count
  (`getExploreBudget` / `getExploreOutputBudget` in `src/mcp/tools.ts`).
  Bigger repos get more calls and more per-call output, never less.
- **Staleness banner** — a `⚠️` prefix on a tool response naming a file
  that's mid-resync (inside the watcher's debounce window) and telling the
  agent to `Read` it directly instead of trusting the graph for that file.
- **`projectPath`** — the parameter that lets an MCP tool call target a
  *different* indexed project than the server's own root — how a monorepo
  with multiple sub-project indexes, or a second repo, gets queried in one
  session.

## Infrastructure & ops

- **`node:sqlite`** — Node's built-in SQLite module (`DatabaseSync`,
  Node ≥22.5), the sole DB backend. No native build step, no WASM fallback.
- **WAL** — Write-Ahead Logging, SQLite's journal mode; `codegraph status`
  reports it to confirm the DB is in the expected mode.
- **FTS5** — SQLite's full-text search extension; powers `codegraph query`
  / symbol name search.
- **File watcher / auto-sync** — native OS file-change events (FSEvents on
  macOS, inotify on Linux, ReadDirectoryChangesW on Windows) debounced
  (~2s default) into an incremental re-index, so the graph never goes
  stale without the user running anything.
- **Daemon** — the background process (`codegraph daemon`/`daemons`) that
  can run the watcher/MCP server outside a single agent session; `codegraph
  daemon` lists running ones so you can stop one.
- **Lock file** — a guard against two indexing processes racing on the
  same `.codegraph/`; `codegraph unlock` clears a stale one.

## Installer & agents

- **Agent target** — one supported AI coding tool the installer can wire
  up: Claude Code, Cursor, Codex CLI, opencode, Hermes Agent, Gemini CLI,
  Antigravity IDE, Kiro, GitHub Copilot (VS Code / CLI / JetBrains). Each
  has its own file under `src/installer/targets/`.
- **`codegraph install`** — wires the CodeGraph MCP server into chosen
  agents' config files. Does **not** index any code.
- **`codegraph init`** — builds (or rebuilds) one project's `.codegraph/`
  index. Does **not** touch any agent configuration.
- **MCP (Model Context Protocol)** — the protocol AI agent harnesses use
  to call out to external tool servers like CodeGraph's; `codegraph serve
  --mcp` is the server process an agent's harness launches as a subprocess.
- **`SERVER_INSTRUCTIONS`** — the text sent in the MCP `initialize`
  response (`src/mcp/server-instructions.ts`); the single source of truth
  for how an agent should use CodeGraph's tools. Not duplicated anywhere
  else — the installer used to write a copy into each agent's `CLAUDE.md`/
  `AGENTS.md`, but stopped (issue #529) once this became the canonical copy.

## Product / business

- **CodeGraph platform** — the not-yet-shipped hosted product (per-PR
  "what to test / what could break" analysis). Currently just a waitlist
  landing page (`getcodegraph.com`); unconnected to the OSS CLI/MCP flow.
- **Telemetry** — anonymous, opt-out usage stats (no code, paths, or
  names) the CLI collects by default; `codegraph telemetry off` disables.

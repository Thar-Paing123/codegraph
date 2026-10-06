# CodeGraph — Business & Product Flow

What CodeGraph is, who uses it, and how a project moves from "just cloned
the repo" to "an AI agent is answering structural questions from the graph."
Written to orient a newcomer to the *product*, not the implementation —
see [architecture-overview.md](architecture-overview.md) for how the pieces
fit internally.

## 1. The problem CodeGraph sells a fix for

AI coding agents (Claude Code, Cursor, Codex CLI, opencode, …) answer
structural questions — "how does X reach Y", "what calls this", "what
breaks if I change this" — by **grep, glob, and Read**, one file at a time,
rebuilding the call graph by hand on every session. That's dozens of tool
calls and a large chunk of the context window spent on *discovery* before
the agent starts the actual work.

CodeGraph's pitch: pre-build that structure once (a knowledge graph of
every symbol, call edge, and dependency), and let the agent ask one precise
question and get back exactly the source + call paths + blast radius it
needs — "surgical context" instead of file-by-file search. The benchmark
numbers in [README.md](../../README.md#benchmark-results) are the proof
point: ~88% fewer tool calls, ~2x+ faster, ~0 file reads, measured across
seven real open-source repos.

## 2. What's actually shipped today

| Piece | What it is | Who touches it |
|---|---|---|
| **CLI** (`codegraph`) | Installer + indexer + MCP server, one self-contained binary (no Node required to install it) | Developer, once per machine / per project |
| **MCP server** (`codegraph serve --mcp`) | Exposes the graph to an AI agent over the Model Context Protocol | The agent's harness launches this automatically |
| **Library** (`@colbymchenry/codegraph` on npm) | The same engine, embeddable in another Node/Electron app | Developers building on top of CodeGraph |
| **Browser viewer** (`codegraph ui`, gated) | Visual graph explorer — not public yet, behind `CODEGRAPH_UI=1` | Early/internal use, dogfooding |
| **Hosted platform** (not shipped — waitlist only) | "For every PR, know what to test, what could break, which flows are affected" | Future product; today just a landing-page waitlist |

Everything in the left column is **100% local** — no data leaves the
machine, no API keys, SQLite only. The hosted platform is the
monetization path that doesn't exist yet; the CLI/MCP server is free,
open-source (MIT), and distributed via npm + direct binary download.

## 3. The end-to-end user flow

```
┌─────────────────────┐   ┌──────────────────────┐   ┌───────────────────────┐
│ 1. Install the CLI   │ → │ 2. Wire up agent(s)   │ → │ 3. Initialize project  │
│    install.sh/ps1    │   │    codegraph install  │   │    codegraph init      │
│    or npm i -g        │   │    (writes MCP config │   │    (builds .codegraph/│
│                       │   │     into each agent)  │   │     index, one-shot)   │
└─────────────────────┘   └──────────────────────┘   └───────────┬───────────┘
                                                                   │
                                   ┌───────────────────────────────┘
                                   ▼
                      ┌─────────────────────────────────┐
                      │ 4. Agent session (the loop that   │
                      │    repeats for the life of the     │
                      │    project):                       │
                      │                                     │
                      │   agent asks a structural question  │
                      │     → codegraph_explore (MCP tool)  │
                      │     → source + call paths +         │
                      │       blast radius, one call        │
                      │     → agent edits code              │
                      │     → file watcher auto-syncs        │
                      │       the index (<1s)                │
                      └─────────────────────────────────┘
                                   │
                      ┌────────────┴────────────┐
                      ▼                          ▼
          ┌───────────────────┐      ┌───────────────────────┐
          │ codegraph uninit   │      │ codegraph uninstall     │
          │ (drop one project's │      │ (remove from every      │
          │  index)             │      │  agent + the CLI itself)│
          └───────────────────┘      └───────────────────────┘
```

Two installer steps are easy to conflate — they're deliberately separate:

- **`codegraph install`** wires the MCP server into your agents. It does
  **not** index any code. Run it once per machine (or once globally); it
  covers every project you open afterward.
- **`codegraph init`** builds one project's actual graph into `.codegraph/`.
  Run it once per project (and never again by hand — see step 4's
  auto-sync loop).

## 4. What happens inside step 4, from the business side

This is the moment the product actually delivers value, and it is
deliberately a **single round-trip**:

1. The agent's harness (e.g. Claude Code) launches `codegraph serve --mcp`
   as a subprocess the first time an MCP tool is needed.
2. On `initialize`, the server hands the agent a short playbook
   (`src/mcp/server-instructions.ts`) that says: reach for
   `codegraph_explore` before grep/Read, trust its answer, don't
   re-verify with grep.
3. The agent calls `codegraph_explore` with either a natural-language
   question or a bag of symbol names. One call returns:
   - the relevant symbols' **verbatim, line-numbered source**, grouped by
     file (same shape `Read` would give — safe to `Edit` from directly);
   - the **call paths** between the named symbols, including
     dynamic-dispatch hops (callbacks, React re-render, interface→impl)
     that grep can't follow;
   - a **blast-radius** summary — what depends on this code, so the agent
     edits with the consequences already in view.
4. Meanwhile, a file watcher keeps the SQLite index in `.codegraph/`
   fresh — debounced sync (~1-2s after a save) so the next question the
   agent asks is answered from current code, with no manual
   `codegraph sync` step.

The commercial logic behind "why would someone adopt this": every repeated
grep/Read loop an agent *doesn't* run is tool calls, tokens, and wall-clock
time the user isn't paying for. CodeGraph's README documents this directly
with before/after benchmarks (median 44% cheaper, 62% fewer tokens, 53%
faster, across seven languages) — the product's growth loop is "try it on
your repo, see the benchmark hold up on your own session."

## 5. Growth & distribution loop (as far as the repo shows)

```
npm package + direct-download binary (free, MIT, local-first)
        │
        ▼
developer installs → wires agent → sees fewer tool calls / faster answers
        │
        ▼
anonymous, opt-out telemetry (no code/paths/names) informs maintainers
        │
        ▼
GitHub Releases + CHANGELOG + README benchmarks drive further adoption
        │
        ▼
waitlist signup → future hosted "CodeGraph platform"
   (per-PR risk/impact analysis — not built yet, business-model bet)
```

There is no paywall, license key, or account requirement anywhere in the
CLI/MCP path today — the entire flow in sections 3–4 is free and runs
without network access. The only business-facing artifact beyond the OSS
tool is the `getcodegraph.com` waitlist referenced in `README.md`, for a
not-yet-built hosted product.

## 6. Who the five "roles" in this flow actually are

| Role | What they do | Where in the flow |
|---|---|---|
| **End developer** | Installs, wires, indexes, then mostly forgets CodeGraph exists — it works through their agent | Steps 1–3 |
| **The AI agent** (Claude Code, Cursor, Codex, opencode, …) | Calls `codegraph_explore` instead of grep/Read; the actual "customer" of the MCP tool, even though a human installed it | Step 4 |
| **CodeGraph maintainers** | Ship the CLI/engine, watch telemetry + benchmark regressions, decide what synthesizer/language to add next | Ongoing, off to the side of this flow |
| **CI / scripts** | Use `codegraph affected` to find which tests a diff touches, non-interactively | Alternate entry point, bypasses the agent entirely |
| **Future hosted-platform customer** | Signs the waitlist today; would get PR-level risk/impact analysis later | Not yet connected to any flow above |

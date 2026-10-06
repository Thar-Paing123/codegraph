# CodeGraph — Architecture Overview

How a line of source code becomes an answer to `codegraph_explore`.
Companion to [business-flow.md](business-flow.md) (the product/user
journey) — this doc is the technical path underneath step 4 of that flow.
The canonical source for all of this is the repo-root
[AGENTS.md](../../AGENTS.md); this is a narrower, example-driven walk
through it.

## 1. The layered pipeline

```
 files on disk
     │
     ▼
┌─────────────────────────┐   tree-sitter / native Rust kernel
│ ExtractionOrchestrator    │   per-language extractors (src/extraction/)
│ (src/extraction/)          │   → nodes (functions, classes, …)
└───────────┬───────────────┘   → edges (calls, imports, extends, …)
            │
            ▼
┌─────────────────────────┐
│  SQLite DB                │   .codegraph/codegraph.db
│  (src/db/)                 │   nodes / edges / files tables + FTS5
└───────────┬───────────────┘
            │
            ▼
┌─────────────────────────┐
│ ReferenceResolver          │   import-resolver, name-matcher,
│ (src/resolution/)          │   framework patterns, callback/tier
│                             │   synthesizers (bridges dynamic dispatch)
└───────────┬───────────────┘
            │
            ▼
┌─────────────────────────┐
│ GraphTraverser /            │   BFS/DFS, impact radius, path finding,
│ GraphQueryManager            │   named-symbol flow, type hierarchy,
│ (src/graph/)                 │   dead-code detection
└───────────┬───────────────┘
            │
            ▼
┌─────────────────────────┐
│ ContextBuilder               │   formats results as markdown/JSON
│ (src/context/)                │   for the MCP tool response
└───────────┬───────────────┘
            │
            ▼
   codegraph_explore response
   (verbatim source + call paths + blast radius)
```

Everything downstream of extraction reads the SQLite DB — there is no
in-memory graph kept across calls; each query hits the DB fresh (reads are
sub-millisecond, per `server-instructions.ts`).

## 2. Public entry point

`src/index.ts` exports the `CodeGraph` class — the one surface library
consumers, the CLI, and the MCP server all drive:

```typescript
const cg = await CodeGraph.init('/path/to/project');  // first-time: create + index
// or: CodeGraph.open(...)                             // reopen an existing index

await cg.indexAll({ onProgress });   // full rebuild
cg.searchNodes('UserService');       // FTS5 search
cg.getCallers(nodeId);               // who calls this
cg.getCallees(nodeId);               // what this calls
cg.getImpactRadius(nodeId, depth);   // blast radius
await cg.buildContext('fix login bug', { maxNodes: 20 });
cg.watch();  / cg.unwatch();         // file-watcher auto-sync
cg.close();
```

The CLI (`src/bin/codegraph.ts`, commander-based) and the MCP server
(`src/mcp/`) are both thin front-ends over this same class — a CLI
`codegraph query` and the MCP tool's search path end up calling the same
`searchNodes`.

## 3. The data model: NodeKind / EdgeKind

Defined once in `src/types.ts`; every extractor and resolver must emit
exactly these strings — there's no per-language enum.

**NodeKind** — `file`, `module`, `class`, `struct`, `interface`, `trait`,
`protocol`, `function`, `method`, `property`, `field`, `variable`,
`constant`, `enum`, `enum_member`, `type_alias`, `namespace`, `parameter`,
`import`, `export`, `route`, `component`, `union`.

**EdgeKind** — `contains`, `calls`, `imports`, `exports`, `extends`,
`implements`, `references`, `type_of`, `returns`, `instantiates`,
`overrides`, `decorates`.

A `route` node + a `references` edge is how a web-framework URL ("GET
/users/:id") gets linked to the handler function that serves it — see
`src/resolution/frameworks/`. A `navigates` edge (an extension beyond the
core EdgeKind set, framework-specific) links the code that sends a user
somewhere to the screen/route it names, so "where does tapping this button
go" is one hop instead of a search.

## 4. Why "dynamic dispatch" needs its own layer

Static tree-sitter parsing only sees direct calls — `foo()` calling
`bar()`. It cannot see:

- a callback registered now and invoked later (`on('event', cb)`),
- React's `setState` → re-render → child JSX,
- an interface method resolved to one of several implementations,
- a native module call crossing a JS ↔ Swift/Kotlin bridge.

`src/resolution/callback-synthesizer.ts` and `src/resolution/frameworks/`
run a **whole-graph synthesis pass** after extraction to add these edges
back in, tagged `provenance: 'heuristic'` with `metadata.synthesizedBy` so
the caller always knows an edge was inferred rather than directly parsed.
This is the single biggest lever for the "zero file reads" benchmark
result in the README — a flow that stops at a dynamic-dispatch boundary
forces the agent to Read to bridge the gap by hand; closing the boundary
in the graph removes that Read entirely. The project's own house rule
(AGENTS.md) is explicit that a **half-bridged** flow is worse than an
unbridged one, because it still produces a Read, just one hop later.

## 5. Request lifecycle: one `codegraph_explore` call

```
Agent (Claude Code, Cursor, …)
   │  MCP tool call: codegraph_explore({ query: "mutateElement renderScene" })
   ▼
MCP server (src/mcp/, ToolHandler in tools.ts)
   │  1. resolve projectPath → find nearest .codegraph/
   │  2. parse query → natural language or a bag of symbol/file names
   │  3. GraphQueryManager: locate matching symbols (FTS5 + name-matcher)
   │  4. named-symbol-flow.ts: find the call path between named symbols,
   │     riding synthesized dynamic-dispatch edges, ≤1 unnamed bridge hop
   │  5. dynamic-boundary-report.ts: note where the graph run out of edges
   │  6. ContextBuilder: assemble verbatim source (re-read from disk,
   │     line-numbered) + relationship map + blast-radius summary
   │  7. explore budget (getExploreBudget / getExploreOutputBudget):
   │     cap call count and per-call size based on indexed file count
   ▼
Response: markdown/JSON with source blocks, calls/references/instantiates
lists, and a blast-radius section — the agent edits from this directly,
no further Read needed.
```

Per-file staleness is layered on top of every response: if the file
watcher's debounce window (default 2s) hasn't finished syncing a file the
response touches, a `⚠️` banner prepends the response naming that file and
telling the agent to `Read` it directly — every other file in the same
response stays trustworthy.

## 6. Module map (where to look for what)

| Module | Owns |
|---|---|
| `src/index.ts` | `CodeGraph` class — the one public surface |
| `src/db/` | SQLite connection, prepared statements, schema, `node:sqlite` adapter |
| `src/extraction/` | Tree-sitter wrappers, per-language extractors, native Rust kernel bridge |
| `src/resolution/` | Reference resolution, framework route detection, dynamic-dispatch synthesizers |
| `src/graph/` | Traversal, impact radius, named-symbol flow, type hierarchy, dead-code detection |
| `src/context/` | Formats query results as markdown/JSON |
| `src/search/` | FTS5 query parsing |
| `src/sync/` | File watcher (FSEvents/inotify/RDCW), debounce, git-hook helpers |
| `src/mcp/` | MCP server, tool definitions, the `server-instructions.ts` playbook |
| `src/installer/` | Per-agent MCP wiring (`codegraph install`/`uninstall`) |
| `src/bin/codegraph.ts` | CLI command definitions |
| `src/ui-server/` + `ui/` | Read-only browser viewer (gated, not yet released) |

## 7. The thing most worth internalizing

CodeGraph's only two levers for improving an agent's behavior are (a) the
MCP `initialize` instructions and tool descriptions, and (b) making the
**answer** more complete. Changing (a) to steer the agent toward a
*different* way of querying has been tried and measured not to work —
agents don't reliably change query style or tool choice from prose
instructions alone. What **does** work, repeatedly validated in this repo,
is (b): meeting the agent where it already calls `codegraph_explore` with
a precise bag of names, and making sure that one call's answer is
*sufficient* — full symbol bodies, every overload in one call, dynamic-
dispatch hops bridged end-to-end — so there is never a reason to fall back
to Read/Grep. See the "Retrieval performance" section of the root
`AGENTS.md` for the full reasoning and the benchmark discipline behind it.

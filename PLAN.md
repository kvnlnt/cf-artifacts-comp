# Lockstep: Build Plan

*Working name. Cloudflare "Build the next GitHub" competition entry.*
*Drafted Tuesday, October 6, 2026. Submission due Wednesday, October 14, 2026, 11:59 PM Pacific Daylight Time (PDT).*

---

## 1. Thesis

> **Conflicts are prevented at planning time, not resolved at merge time.**

Most entries will build smarter merging. Lockstep does the opposite: agents declare what they will touch before they start, locks are held at the level of individual functions, every commit carries machine-readable intent, and a three-tier funnel (rules, then a decision model, then a large language model) decides which overlaps actually matter and writes the integration code to fix them.

The concurrency model borrows directly from databases and systems programming: read/write locks, intention locks, leases with fencing tokens, and deadlock-free all-or-nothing acquisition.

---

## 2. Competition facts

| Item | Detail |
|---|---|
| Deadline | October 14, 2026, 11:59 PM PDT. **Target submitting by noon that day.** |
| Deliverables | 5–10 minute demo video, open-source repository, instructions to run it |
| Required stack | Cloudflare developer platform, **specifically Workers and Artifacts** |
| License | MIT, Apache 2.0, or BSD (2- or 3-clause), with a `LICENSE` file in the repository |
| Finalists | 3 teams announced October 16; 10-minute live demo at Cloudflare Connect, San Francisco, October 21. Must attend in person to win |
| Prize | $25,000 in Cloudflare credits plus the Connect VIP speakers dinner |
| Entries | One per entrant |
| IP | You keep all rights. Submissions are not confidential; Cloudflare may build similar things |
| Framing | "Hundreds of thousands of agents" working concurrently |

### Judging (each scored 1–5)

| Criterion | Weight | What answers it |
|---|---|---|
| Originality and quality of the prototype | **50%** | Plan-time conflict prevention; symbol-level locks; intent trailers; three-tier triage funnel; merge agent that writes glue code |
| Concurrency, coordination, context preservation, review, conflict handling | 25% | Locks (concurrency, coordination), trailers (context), Clef and merge agent (review, conflict handling) |
| Ease of use and user experience | 25% | Model Context Protocol (MCP) server so any coding agent can use it; live dashboard; one-command setup |

Ties are broken by the originality score, so the thesis and the demo narrative matter more than feature count.

---

## 3. Architecture

| Component | Cloudflare product | Role |
|---|---|---|
| Repository storage | **Artifacts** | `main` repository plus one fork per agent task. Writes over the Git protocol (isomorphic-git from Workers); reads via the Workers binding (`readFile`, `log`, `readCommit`) |
| API gateway | **Workers** | The single interface agents talk to: plan, acquire, commit, release, request merge |
| Lock managers | **Durable Objects** | One object per module (directory). Holds symbol-level read/write locks, wait queues, leases. Sharding by module is the scale story |
| Coordinator | Durable Objects | Global agent registry, task assignment, deadlock detection across modules, WebSocket fan-out to the dashboard |
| Merge pipeline | **Workflows** | Durable, retryable stages: extract facts, run rules, triage, merge agent, splice, validate, push |
| Triage | **Workers AI: Clef / Clef-flash** | Typed decisions with probabilities on each flagged overlap |
| Merge agent | Workers AI (large language model) | Writes glue code for overlaps Clef routes to it |
| Agent interface | Workers (Agents SDK, remote MCP server) | Exposes the API as MCP tools so Claude Code, Codex, or any agent can participate |
| Fact store | D1 | JSON Lines (JSONL) trailer facts indexed by symbol for fast joins; concern history |
| Dashboard | Workers static assets | Live view of agents, locks, concerns, merges |

### Agent lifecycle

```
1. plan       Agent receives a task, reads code, produces a footprint:
              { write: [symbols], read: [symbols], create: [new names] }
2. acquire    Coordinator sorts footprint globally, asks each module's
              Durable Object for locks. All granted, or none (agent queues
              and gets an estimate, or is told who it conflicts with).
3. work       Agent edits in its own Artifacts fork. Lease is renewed by
              heartbeat. Writes outside the footprint are rejected.
4. commit     Each commit carries Agent-Event trailers (JSON per line).
              Commit-message validator rejects malformed trailers.
5. release    Agent requests merge; write locks convert to "pending merge".
6. merge      Workflow runs the pipeline (below), splices blocks into main,
              releases locks, notifies agents whose read locks were affected.
```

### Merge pipeline

```
fork commits
   │
   ▼
extract trailers ──► JSONL facts (with branch + commit added)
   │
   ▼
rules engine (TypeScript joins) ──► structural overlaps only
   │      e.g. A assumes X, B changed X's contract
   ▼
Clef-flash (fast first pass, batches of up to 64 questions)
   │      confident "compatible" ──► auto-merge
   ▼
Clef (full model for close calls)
   │      auto_merge | glue | regenerate | escalate
   ▼
merge agent (glue / regenerate) ──► its own commit with trailers
   │
   ▼
splice blocks into latest main ──► type-check ──► push ──► release locks
```

---

## 4. Data model

### Symbol identity

`<file path>::<container>::<name>`, for example `src/orders/service.ts::OrderService::getOrders`.

Each function, method, class field, type, and top-level constant gets **two lockable parts**:

- `…::getOrders#sig`: the signature. Callers hold shared locks on signatures they depend on.
- `…::getOrders#body`: the body. Only the editor needs it.

Special cases:

- **Imports** are a set, merged automatically and sorted. Never locked.
- **Other top-level code** falls back to a `<file>::#header` lock.
- **New symbols** take a name reservation in their scope, not a lock.
- **Non-code files** use file-level locks (key-level for JSON and YAML is roadmap).

Parsing uses web-tree-sitter (tree-sitter compiled to WebAssembly) with the TypeScript grammar. **TypeScript only for the prototype.**

### Lock record (inside a module's Durable Object)

```ts
type LockMode = "S" | "X" | "IS" | "IX";        // shared, exclusive, intention-shared, intention-exclusive
interface Lock {
  symbol: string;          // "src/orders/service.ts::OrderService::getOrders#sig"
  mode: LockMode;
  agentId: string;
  taskId: string;
  fence: number;           // monotonically increasing fencing token
  leaseExpiresAt: number;  // epoch ms, renewed by heartbeat
}
```

Rules: S/S compatible; X conflicts with everything; IS/IX on a file or module let a coarser lock see activity inside without scanning symbols. Writes to Artifacts carry the fence; stale fences are rejected.

### Commit trailers

One trailer key, one JSON object per line, so the trailer block is JSON Lines:

```
Sort order list by priority

Orders now sort by priority, falling back to date for ties.

Agent-Event: {"kind":"task","agent":"agent-3","task":"TASK-142","goal":"Let warehouse staff see urgent orders first"}
Agent-Event: {"kind":"assumes","symbol":"src/orders/model.ts::Order.priority","claim":"non-null, defaults to 0"}
Agent-Event: {"kind":"contract","symbol":"src/orders/service.ts::OrderService::getOrders","change":"behavior","before":"sorted by date","after":"sorted by priority"}
Agent-Event: {"kind":"gap","symbol":"src/reports/monthly.ts","note":"still expects date order"}
Agent-Event: {"kind":"test","symbol":"src/orders/service.test.ts::sorts by priority"}
```

Event kinds: `task`, `assumes`, `contract`, `gap`, `test`, `lock` (written by the system, not the agent). Extraction adds `branch` and `commit`.

Query with plain Git:

```bash
git log main..agent-3 --format="%(trailers:key=Agent-Event,valueonly)"
```

### Rules (first set)

| Rule | Join | Severity |
|---|---|---|
| Broken assumption | A `assumes` symbol S; B `contract` on S | High, send to Clef |
| Double contract change | A and B both `contract` on S | Conflict, send to Clef |
| Unfilled gap | A `gap` on S; no other branch `contract`s or edits S | Glue needed |
| Filled gap | A `gap` on S; B edits S | Verify with Clef |
| Disjoint | No shared symbols | Auto-pass |

### Clef triage questions (per overlap)

- `breaks` (noul): Does the change invalidate the assumption?
- `severity` (score): None / Cosmetic / Incorrect behavior / Crash or data loss
- `action` (choice): `auto_merge` / `glue` / `regenerate` / `escalate`

Thresholds live in config and are shown on the dashboard. Tune them against a small hand-labeled set (target 20 cases).

---

## 5. Repository layout

```
cf-artifacts-competition/
├── LICENSE                     # MIT
├── README.md                   # Pitch, architecture diagram, run instructions
├── PLAN.md                     # This file
├── apps/
│   ├── api/                    # Worker: HTTP API, MCP server, Durable Objects, Workflows
│   └── dashboard/              # Live UI (static assets on Workers)
├── packages/
│   ├── symbols/                # tree-sitter parsing, symbol IDs, splice, import merge
│   ├── trailers/               # write, validate, extract Agent-Event JSONL
│   ├── rules/                  # fact joins producing concerns
│   └── triage/                 # Clef question builder + threshold routing
├── agents/                     # Agent harness + scripted scenario runner + load generator
└── demo/
    ├── sample-repo/            # Small TypeScript order-management app
    └── scenarios/              # Scripted tasks that produce the showcase conflicts
```

---

## 6. Schedule

| Day | Focus | Done when |
|---|---|---|
| **Tue Oct 6** (afternoon) | **Spikes and scaffold.** Artifacts: create repo, fork, push a commit from a Worker with isomorphic-git, read it back with the binding. Call Clef and Clef-flash. Load web-tree-sitter in a Worker. Scaffold monorepo, `LICENSE`, Wrangler config | All three spikes work end to end, or a fallback is chosen |
| **Wed Oct 7** | **Symbol layer.** Parse TypeScript into a symbol map with signature/body ranges. Splice algorithm: replace changed blocks into the latest file. Import-set merge | Unit tests: two agents edit different functions in one file, splice yields correct file with no text merge |
| **Thu Oct 8** | **Lock layer.** Module Durable Object with S/X/IS/IX locks, leases, fencing tokens, wait queue. Coordinator: global ordering, all-or-nothing acquisition, cycle detection. API endpoints | Tests: conflicting footprints queue correctly; expired lease plus stale fence is rejected; no deadlock under random footprints |
| **Fri Oct 9** | **Agents and trailers.** MCP server exposing `plan`, `acquire`, `read`, `write`, `commit`, `release`, `request_merge`. Trailer writer and validator. Agent harness driving a Workers AI model through the MCP tools | One real agent completes a task end to end: footprint, locks, fork, commits with valid trailers, merge request |
| **Sat Oct 10** | **Merge pipeline.** Workflow: extract, rules, Clef-flash, Clef, merge agent, splice, type-check, push, release, notify | The showcase conflict (Section 7) is caught by rules, diagnosed by Clef, and fixed by generated glue code |
| **Sun Oct 11** | **Dashboard.** Live WebSocket feed: agent list and status, lock heat map by file and symbol, concerns feed with Clef probabilities, merge timeline, trailer inspector per commit | Dashboard tells the whole story with no narration |
| **Mon Oct 12** | **Demo and scale.** Scripted scenario with 4–6 real agents. Load generator: 10,000+ simulated agents against the lock layer, measuring acquisition latency and throughput per Durable Object | Repeatable demo run under 3 minutes; scale numbers captured |
| **Tue Oct 13** | **Polish and record.** README with one-command setup, architecture diagram, deploy instructions. Record a draft video | A stranger could run it from the README |
| **Wed Oct 14** | **Buffer and submit.** Final video cut, final checks, submit by noon | Submitted |

---

## 7. Demo script (target 7 minutes)

1. **Problem (0:00–0:45).** Git was built for humans who take days per change. Agents take minutes and arrive by the thousand. Merge conflicts, and worse, changes that merge cleanly and still break each other.
2. **Thesis (0:45–1:15).** Prevent conflicts at plan time. Lock functions, not files. Record intent with every commit.
3. **Live run (1:15–4:00).** Five agents get tasks on the sample order app. Dashboard shows footprints, locks granted, one agent queued behind another with an estimate. Two agents edit different functions in the same file at the same time.
4. **The showcase conflict (4:00–5:30).** Agent A builds a monthly report assuming `getOrders()` sorts by date. Agent C changes it to sort by priority. Both pass their own tests; text merge would be clean. Rules flag the overlap from the trailers, Clef says "breaks: 0.94, action: glue," the merge agent adds an explicit date sort in the report with its own trailer explaining why. Show the trailer inspector.
5. **Scale (5:30–6:30).** Load test: lock managers shard per module, numbers on screen.
6. **Close (6:30–7:00).** Any agent can join through MCP. Everything runs on Cloudflare: Artifacts, Workers, Durable Objects, Workflows, Workers AI with Clef.

---

## 8. Scope cuts (mention as roadmap)

- Languages beyond TypeScript
- Regenerating a conflicting branch from its intent instead of merging
- Hot-spot analytics that flag contended functions as refactor candidates
- Key-level locks for JSON and YAML
- Visual regression checks using Clef image input with Browser Rendering screenshots
- Human review UI beyond the escalation queue

---

## 9. Risks and fallbacks

| Risk | Fallback |
|---|---|
| The Artifacts binding exposes no branch or merge methods, so writes must go through the Git protocol | isomorphic-git from a Worker (documented example). If it's slow, push from the agent harness with a scoped write token from `createToken("write")` |
| Artifacts is new; unknown limits | Check the limits page during the Tuesday spike; one fork per task, not per commit |
| web-tree-sitter too heavy for a Worker | Parse in the agent harness and send symbol ranges with the commit; validate server-side with the TypeScript compiler API in a Workflow step |
| Clef calibration doesn't match its claims | Hand-labeled set to tune thresholds; rules alone still produce a usable concern list |
| Merge agent writes bad glue code | Type-check gate before push; escalate on failure |
| Solo, 8 days | Strict daily "done when" gates. If a day slips, cut from the dashboard polish, never from the showcase conflict |

---

## 10. Open decisions

- **Team:** solo or bring a second person? Up to two people travel to Connect.
- **Name:** "Lockstep" is a placeholder.
- **Which models drive the agents:** Workers AI keeps it all-Cloudflare; MCP means Claude Code or Codex can also join, which is a strong ease-of-use point in the demo.
- **Merge agent model:** pick the strongest code model available on Workers AI during the Tuesday spike.

---

## References

- Competition: https://www.cloudflare.com/git-competition/
- Official rules: https://www.cloudflare.com/documents/build-next-gen-git-platform-competition-terms.pdf
- Artifacts: https://developers.cloudflare.com/artifacts/
- Artifacts Workers binding: https://developers.cloudflare.com/artifacts/api/workers-binding/
- Clef: https://developers.cloudflare.com/workers-ai/models/clef/

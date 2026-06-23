# Build Spec: Agentic E2E-Test Generation Harness (`bench-runner`)

> **How to use this file.** Open a Claude Code session in an empty directory and hand it this spec
> (`claude "$(cat benchmark-harness-spec.md)"`, or paste it as the first message). This is a *build*
> task: Claude Code is the **builder** of a reusable harness. Claude Code is **not** one of the agents
> being benchmarked — the candidates are TC, Cursor, Copilot, and Sonnet 4.6 (see §1).

---

## 1. Mission

Build a **reusable, repo-agnostic** harness that measures how well different agentic AI tools
**regenerate working E2E test specs** for a target repo whose tests have been removed. The harness must
generalize to new repos **by adding a manifest — never by editing engine code**.

This harness's job ends at producing a **converged, self-verified spec per candidate**, plus metadata.
It does **not** judge or compare quality — a separate agent, **tc-testbed**, consumes this harness's
output later and does the LLM-as-judge scoring. Do **not** build any judge here.

**Candidates (four):**
| Candidate | Type | Self-drives the run→fix loop? |
|---|---|---|
| **TC** ("Test Companion", VS Code ext, **no terminal**) | GUI tool, runs specs in-IDE | **Yes** — TC authors *and* executes specs internally |
| **Cursor** (default model) | CLI tool (`cursor-agent`) | Yes — its agent runs terminal commands |
| **Copilot** (default model) | CLI tool | Yes — its agent runs terminal commands |
| **Sonnet 4.6** (raw Anthropic API, user-provided key) | raw model | **No** — has no loop of its own; the **harness must implement** generate→run→fix→retry |

You are benchmarking each **tool as it ships** (you do not control Cursor's/Copilot's model). Sonnet 4.6
is the one candidate the harness drives itself, and must be reported as its own category — "Sonnet as
wrapped by the harness," not apples-to-apples with the self-driving tools.

## 2. The three phases (the manifest serves all three)

This is an **agent-drives-the-loop** design. The harness provides a *runnable* environment; the agent
does the generate→run→fix→re-run iteration. Execution here is the **agent's own feedback loop to produce
a working spec**, not a scoring gate.

1. **Setup.** Bring up the system under test (SUT), provision sealed per-candidate worktrees, cache deps,
   and take a clean DB snapshot. (Harness does this.)
2. **Generate + self-verify.** Each candidate writes the spec, runs it against the live SUT, fixes
   failures, and re-runs **until it passes** (or hits a bound). The three tools drive this themselves;
   the Sonnet-API adapter's loop is harness-implemented. **The SUT DB is restored to the clean snapshot
   before each candidate** so no one inherits a dirty state.
3. **Final output.** Write the converged spec + metadata in the handoff layout (§5) for tc-testbed.

No execution gate and no judge live in this harness — phase 2 execution is the *agent's* loop, and
correctness scoring belongs to tc-testbed.

## 3. v1 scope (this smoke test)

- Prove the **whole loop end-to-end on Mattermost only**, **already cloned locally** (ask me for the
  path — §10).
- **Recommended first candidate: the Sonnet 4.6 API adapter.** It's the most self-contained (uses the
  key I provide, needs no external CLI install/auth) and it exercises the genuinely new, riskiest pieces
  — the harness-driven generate→run→fix loop and the SUT snapshot/restore between runs. Once that loop is
  proven, add one CLI tool (launch+wait), then TC. (Flag if you'd sequence differently.)
- **Default executor = `local`** (this Mac, sequential). Containerization is designed-for but not
  required working in v1 — don't let Docker block the first run.
- **Defer, behind clean seams:** Cursor/Copilot adapters, TC driver extension, full containerization,
  BrowserStack execution for remote-driven frameworks.

Keep v1 lean: the full loop working for **one** candidate on Mattermost, on the right abstractions,
beats a half-built version of all four.

## 4. Non-negotiable architecture

Settled decisions — implement, do not redesign.

### 4.1 Containerize the toolchain, not the world (polyglot setup)
Setups vary **per language ecosystem, not per repo**. Target a small fixed set of base images, never one
per repo:
- `bench-node` (Node + pnpm) → Playwright, Cypress, Detox build, WebdriverIO, JS Selenium
- `bench-jvm` (JDK + Gradle/Maven) → Java Selenium, Espresso, JVM Appium
- `bench-python` (Python + uv) → pytest-Selenium, Python Appium

Push heavy/OS-specific parts out: **Playwright/Cypress** bundle browsers (in-process, hermetic);
**Selenium/Appium/Espresso** point at **BrowserStack** (no local grid/emulator/SDK image); **Detox** is
a host-only outlier (macOS sims) — deprioritize. Repo-specific deps/commands are **manifest data**, not
new images. **v1 may use the `local` executor and skip building images**, but write `local` and
`container` as interchangeable executor implementations.

### 4.2 One bare clone + N worktrees (no duplicate clones)
One **bare** clone as shared object store + one **git worktree per candidate** (`ground-truth`, `wt-tc`,
`wt-cursor`, `wt-copilot`, `wt-sonnet`), each on its own branch from the same base commit. Reset via
`git reset --hard <branch> && git clean -fdx`, never re-clone. Share deps across worktrees via a cache
keyed by **lockfile hash** (+ pnpm content-addressable store) so install runs **once per unique
lockfile**, not per candidate.

### 4.3 Sealed sandboxes (no contamination)
Each candidate's generation sandbox must contain **exactly** what a developer would have if the tests
were never written — nothing else. Close all three leaks **structurally**:
- **Sibling output:** each candidate only ever sees its own worktree. No shared output dir.
- **Ground truth:** golden specs are **never present in** a generation sandbox. They live only in
  `ground-truth`, consumed only by tc-testbed later — not by this harness at all during generation.
- **Recoverable history:** committing a deletion is not enough (`git show HEAD~1:<path>` recovers it).
  For v1, **exclude `.git` from the generation sandbox** so the agent sees a plain source tree with no
  repo to interrogate. (Alternative: orphan-branch with the specs-removed state as the first commit.)

Record the **repo-context dial**: `minimal` (vacuum) vs `developer` (neighbouring tests/docs visible).
Both are valid but *different* benchmarks. Non-negotiable part is only that **answer key + sibling output
are unreachable**.

### 4.4 SUT bring-up + reset (new — required because phase 2 executes)
Because candidates now *run* specs against a live server, state parity is mandatory or the comparison is
unfair. The harness:
- brings the SUT up from the manifest (`sut.type`, `sut.file`, `sut.ready`),
- migrates/seeds it to a known baseline, then takes a **DB snapshot**,
- **restores that snapshot before each candidate runs**, so no candidate inherits another's mutations.

For Mattermost (Postgres-backed): snapshot via `pg_dump`/volume snapshot after baseline; restore before
each candidate. **Note/caveat to encode:** DB-only restore does **not** roll back filesystem uploads —
fine for spec generation, flag if a task ever touches attachments. Confirm the actual compose file +
readiness endpoint by **inspecting the repo** (§10), don't assume.

### 4.5 Manifest = single source of truth (one prompt, fanned out)
One manifest per repo; each phase reads its slice. The **prompt is defined in one place and fanned out to
all candidates** — never pasted per-tool (that's the whole point of killing the manual loop). Target
schema:

```yaml
id: mattermost-accessibility
repo: https://github.com/mattermost/mattermost
clone_path: <LOCAL PATH — user-provided, §10>
base_commit: <sha>

# GENERATION
framework: playwright
prompt_template: prompts/playwright-enriched.md
specs_under_eval:                 # deleted for candidates; what they regenerate
  - <glob(s) — detect from repo, confirm, §10>
repo_context: minimal | developer # the §4.3 dial

# ENVIRONMENT (so the AGENT can run/fix; and the API adapter's loop can check pass/fail)
runtime: bench-node
executor: local | container       # v1 default: local
install: "<install cmd>"
test_cmd: "<runner cmd; emits JUnit so pass/fail is machine-readable>"
results: <path to JUnit xml>
sut:
  type: compose | none
  file: <path or null>
  ready: "<readiness check cmd>"
  reset: db-snapshot              # §4.4

# RETRY BOUND (Sonnet-API loop; see §4.7)
max_iterations: <int>
max_wallclock_ms: <int>

# HANDOFF
output_root: runs/               # §5
```

Adding a Java/Appium repo later = a new manifest with different strings; the engine never branches on
language.

### 4.6 Agent-adapter interface (two types, one contract)
Every candidate hides behind one contract the runner calls uniformly:

```
adapter.run({ prompt, worktreePath, env }) ->
  { wallClockMs, converged: bool, iterations?: int, filesWritten: string[], cost?: {tokens, usd} }
```

- **Self-driving tools (TC, Cursor, Copilot):** the harness gives them a runnable sandbox (live SUT +
  `test_cmd`) and a prompt instructing them to run/fix/iterate until passing; then **launch and wait**.
  *They* own the loop and decide when done. Wall-clock is measured prompt-submit → done and **includes
  their iteration time** (a tool that converges fast is genuinely better).
  - CLI (Cursor `cursor-agent`, Copilot CLI): spawn with `cwd = worktree`, **process exit = done**, parse
    JSON for cost/tokens.
  - TC (no terminal): driven by a small **driver extension** launched via `@vscode/test-electron` that
    invokes TC's registered command (in TC's `package.json`) and detects completion. Completion-detection
    is a **selectable strategy**: (1) await the command Promise [preferred], (2) explicit done-event,
    (3) file-stability timeout [fallback]. Leave command name + chosen strategy as **TODOs** (§10).
- **Raw model (Sonnet 4.6):** has no loop — the **harness implements** it (§4.7). Key via
  **`ANTHROPIC_API_KEY` env var, never in the manifest or any committed file.**

**Metrics policy:** **wall-clock** (prompt-submit → done) is captured **identically for all four** — make
it primary. Token/USD cost comes from CLI/API JSON; TC likely won't expose it → report `n/a`, don't
fabricate. Also record `converged` (did the candidate reach a passing spec).

### 4.7 The Sonnet-API agentic loop (the one genuinely new component)
For Sonnet 4.6 the adapter must itself perform: gather repo context (per the §4.3 dial) → call the API →
write the spec to the worktree → run `test_cmd` against the live SUT → parse JUnit for pass/fail →
on failure, feed failures back to the model → repeat. **Bounded** by `max_iterations`/`max_wallclock_ms`;
record `converged` + `iterations`. A lightweight JUnit pass/fail check is needed **only to drive this
loop's stopping condition** — it is not a scoring gate and is not used for the self-driving tools.

## 5. Handoff output layout (tc-testbed will be reshaped to consume this)
Write a predictable, self-describing structure per run:
```
runs/<repo-id>/<timestamp>/
  <candidate>/
    specs/            # converged spec files, in the repo's original spec paths
    meta.json         # wallClockMs, converged, iterations, cost|n/a, filesWritten, promptHash, baseCommit
  run-manifest.json   # repo, baseCommit, candidates, prompt, sandbox refs, sut reset mode
```

## 6. Suggested layout
```
bench-runner/
  bin/bench.mjs                 # bench run <manifest> [--candidates=...]
  src/
    manifest.mjs                # load + validate (§4.5)
    sandbox.mjs                 # bare clone + worktrees + sealed state (§4.2, §4.3)
    deps.mjs                    # lockfile-hash dep cache (§4.2)
    sut.mjs                     # bring-up + snapshot + restore-between-candidates (§4.4)
    executors/{local,container}.mjs
    adapters/{tc,cursor,copilot,sonnetApi}.mjs   # sonnetApi full in v1; rest stub
    adapters/interface.md
    junit.mjs                   # pass/fail parse — ONLY for the API loop's stop condition (§4.7)
    report.mjs                  # collect rows -> table/JSON/TSV
  manifests/mattermost.yaml
  prompts/playwright-enriched.md
  README.md                     # how to run + the §4.3 dial decision recorded
```
Conventions (match my stack): **Node ESM `.mjs`**, **pnpm**, Mac M2, terminal-first, lightweight deps,
**sequential** candidate runs.

## 7. Build order (verify each milestone)
1. **Manifest load + validate** against `manifests/mattermost.yaml`; print resolved config.
2. **Sandbox provisioner** → bare clone + `ground-truth` + one candidate worktree, specs removed +
   `.git` excluded per §4.3. **Verify:** golden specs absent from the sandbox, `.git` unreachable,
   `ground-truth` pristine. **← check in with me here before generating anything.**
3. **SUT bring-up + snapshot/restore** (§4.4) → server reaches `ready`; snapshot taken; restore works.
   **Verify:** restore returns DB to baseline.
4. **Sonnet-API adapter + local executor** (§4.6, §4.7) → context → API → write → run → fix → retry,
   bounded; restore SUT before the run. **Verify:** a converged (or bounded-out) spec + meta produced.
5. **Report + handoff** → write the §5 layout; emit table + JSON + TSV.
6. **Stubs** → Cursor/Copilot (interface-complete, TODO), TC (driver-ext with 3 completion strategies),
   container executor skeleton mirroring `local`.

## 8. Acceptance (v1)
- `bench run manifests/mattermost.yaml --candidates=sonnet` runs setup → generate+self-verify → handoff
  with **no manual file moving, no pasted prompts, no hand-passed paths**.
- Ground truth provably never present in the generation sandbox.
- SUT restored to baseline before the candidate runs.
- Output written in the §5 layout with wall-clock + cost + `converged` recorded.
- A second (unrun) manifest stub for a different framework exists, proving the engine doesn't hardcode
  Mattermost.

## 9. Guardrails
- **Never mutate `ground-truth` or the original clone's tracked tests.**
- Sequential runs; wall-clock from prompt-submit (not process-launch); **restore SUT before each
  candidate**.
- No global framework installs; rely on repo deps + bundled browsers.
- API key only via `ANTHROPIC_API_KEY` env; never written to disk.
- Don't fabricate cost for agents that don't report it.
- Bound the API loop; record `converged`.
- On genuine ambiguity, prefer the simplest option preserving §4 invariants and leave a marked TODO
  rather than guessing on §10.

## 10. Surface to me — do not guess
1. Mattermost **local clone path** + **base commit** to pin.
2. **Framework + spec globs** — confirm by **inspecting the repo on disk** (I believe Playwright
   accessibility specs under the e2e dir — verify, don't trust memory).
3. **`repo_context` dial** for v1: `minimal` vs `developer`.
4. **SUT bring-up for Mattermost** — compose file + readiness endpoint + how to snapshot/restore Postgres
   — confirm against the repo before wiring.
5. **TC's registered generate command** + which **completion-detection strategy** it supports — TODO
   until I provide it.

Start at §7 step 1 and **check in after the sandbox provisioner (step 2)** so we confirm the
contamination seal before any generation runs.

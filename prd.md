# Hendeka — PRD

**Version:** 0

Hendeka is an Open Source agent that gives a Repo Maintainer one reviewable task each weekday at 11:11, while taking care of all the other tasks that don't need a human in the loop. It runs a six-phase pipeline over a git repository, produces at most one artifact per weekday, and posts it to the configured forge. Runtime foundation is pi-mono (`@mariozechner/pi-ai`, `@mariozechner/pi-agent-core`, `@mariozechner/pi-coding-agent`, MIT). Default model configuration is Ollama (zero cost, fully offline); OpenRouter, direct provider APIs, and self-hosted endpoints are all first-class alternatives.

Onboarding: `hendeka add <url>`. No required paid services. Runs on Raspberry Pi, laptop, or $5 VPS.

---

## Naming

Hendeka (ἕνδεκα) is Greek for "eleven." In classical Athens, οἱ ἕνδεκα were the eleven magistrates responsible for daily custodial administration of the polis — a small body doing routine work that kept the system running. The name maps to the product: small body, routine work, keeps the repository functioning.

Hendeka is a sibling to **plato**, an Open Source agentic learning platform also built on pi-mono. The names are chosen together. Plato is the philosopher; his *Republic* presents the Divided Line and the cave as models for how understanding progresses from shadow to form. Hendeka are the magistrates in Plato's Athens — the practical workers who kept the city functioning while the philosophers reasoned about it. The two projects share infrastructure, conventions, and the broader philosophers.group / 11:11 Philosophers Group context, but they are distinct products with distinct scopes. Where the projects share architectural patterns (pi-mono integration, AGENTS.md discipline, agent-maintained codebases), the patterns are documented once and reused; where they diverge, each project owns its own decisions.

---

## 1. Hard constraints

These cannot be relaxed or worked around. Violating any of them is a failed build.

**1.1 The Sabbath is immutable.** No runs on Saturday or Sunday in the Maintainer's configured timezone. Enforced in four places:
1. Config parser rejects cron expressions matching day 0 (Sunday) or day 6 (Saturday).
2. Scheduler checks day-of-week against configured timezone before invoking the pipeline; exits silently if weekend.
3. CLI (`hendeka scope`, `hendeka config`) rejects any flag attempting to enable weekend runs.
4. Pipeline entry (`runPipeline()`) performs a final day-of-week assertion; throws on weekend.

Manual `hendeka run <repo>` on a weekend executes (the rule governs scheduled behavior only). Integration test: 14-day clock-advance simulation must fire zero weekend runs.

**1.2 Open Source end to end.** All dependencies MIT or Apache-2.0. Zero required paid services. Local models (Ollama, vLLM, MLX) are first-class v0 paths.

**1.3 Runs on commodity hardware.** Target 1 vCPU, 1 GB RAM, 25 GB disk. Must work on Raspberry Pi 4 with 4 GB+ RAM.

**1.4 One task per weekday. At most.** Zero-artifact days are valid with explanation. Hard enforcement at orchestrator, not soft preference.

**1.5 No custom `ModelProvider` or `BuildEngine` abstractions.** pi-ai is the model abstraction; pi-agent-core is the build runtime. Hendeka configures both. Re-wrapping them is an explicit anti-pattern.

**1.6 Scope gating at every write.** Every forge write and every Build-phase tool call passes through scope check. Scope enforcement in Build uses the `tool_call` extension returning `{ block: true, reason }`.

**1.7 Defense-in-depth sandboxing.** Every Build shell command on Linux runs through bubblewrap + Landlock + seccomp + egress proxy under the `hendeka-build` user. On macOS: sandbox-exec. `--unsafe` is the only bypass; logs WARN every run.

**1.8 Hendeka never writes AGENTS.md for target repos.** Phase 5 is prohibited from creating, modifying, or proposing AGENTS.md/CLAUDE.md in target repositories. Integration test asserts none created.

---

## 2. Success metrics (9)

Computed every run by Phase 0. Written to `metrics_rolling`.

1. Weekly active repos (≥1 run completed in last 7 days).
2. Artifact review time (median hours, post to Maintainer response).
3. Zero-artifact rate.
4. Artifact acceptance rate (merged / closed-accepted, rolling 30 days).
5. CI pass rate on initial push.
6. Post-merge revert rate (within 30 days). Lagging; first data ~day 30.
7. False-action rate (Hendeka actions reversed by Maintainer).
8. Cost per artifact (USD when paid provider; 0 when local).
9. Clean merge rate (% of merged PRs unmodified before merge).

---

## 3. System overview

Single Node.js process. No long-running daemon. Scheduler fires; Hendeka starts; runs six phases; exits. State in SQLite between runs.

```
         11:11 M–F (Maintainer timezone) — Sabbath enforced
Scheduler (systemd / launchd / node-cron fallback)
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│ Hendeka (single Node.js process)                         │
│                                                          │
│  0. Metric refresh   → reads forge state, updates metrics│
│  1. Cartography      → reads repo, forge, KPI endpoints  │
│  2. Issue triage     → classifies and acts on issues     │
│  3. PR triage        → classifies and acts on PRs        │
│  4. Prioritization   → picks the day's artifact type     │
│  5. Build            → runs pi-agent-core session        │
│  6. Notification     → local log + forge artifact        │
│                                                          │
│  Runtime: pi-ai + pi-agent-core + pi-coding-agent (MIT)  │
│  State:   SQLite (~/.hendeka/state.db)                   │
│  Traces:  JSONL + SQLite + optional OTel exporter        │
└──────────────────────────────────────────────────────────┘
        │                  │                   │
        ▼                  ▼                   ▼
   Model provider    Forge (GitHub v0)   KPI endpoints
   (any pi-ai        via @octokit/rest   via fetch+JSONPath
    provider;                            optional
    phases 0–5)
```

Phases run sequentially. Each writes outputs to SQLite before the next starts; crashes are resumable.

---

## 4. The six phases

"PR" below means the forge's ChangeRequest concept — PR on GitHub, MR on GitLab, PR on Gitea/Forgejo. The `Forge` interface handles the mapping.

### Phase 0: Metric Refresh
- **In**: forge state (last 30 days), prior runs from SQLite.
- **Do**: compute rolling metrics; determine `behavior_mode` (§10 circuit breaker).
- **Out**: `metrics_rolling`, `behavior_mode` for this run.

### Phase 1: Cartography
- **In**: repo URL, manifest (last known), KPI endpoint list.
- **Do**: shallow clone via `GitOps`; read target-repo `AGENTS.md`/`CLAUDE.md`/`.cursor/rules/*` into reference context; fetch forge state via `Forge`; fetch KPI values; build new manifest (§7); diff against prior.
- **Out**: updated manifest, drift flags.
- **Model**: small/cheap (triage tier).

### Phase 2: Issue Triage
- **In**: open issues with comments.
- **Do**: classify each (`spam`, `duplicate`, `needs_info`, `actionable`, `maintainer_handled`). Act per scope: spam → close with template; duplicate → link; needs_info → comment; actionable → label.
- **Out**: triage decisions, spam-candidate count.
- **Model**: triage tier.

### Phase 3: PR Triage
- **In**: open PRs with comments, review state, CI status.
- **Do**: classify (`ready_to_merge`, `needs_review`, `needs_changes`, `stale`, `abandoned`). Act per scope.
- **Out**: triage decisions, ready-to-merge count.
- **Model**: triage tier.

### Phase 4: Prioritization
- **In**: triage outputs, KPI signals, drift flags, behavior mode.
- **Do**: rank candidates; select artifact type by **deterministic rules** (not LLM):
  - Drift flag fired → **Manifest Drift Alert**
  - PR flagged ready-to-merge, no higher-value Build candidate → **Review Digest**
  - Untriaged issues >30 or spam ≥5 → **Triage Digest**
  - Strong KPI signal with no matching issue → **Priority Recommendation**
  - Scope-fit Build candidate exists → **PR**
  - Otherwise → zero artifact with explanation
- **Out**: ranked list, selected artifact, target item(s).
- **Model**: prioritization tier (ranking within type; type selection is rule-based).

### Phase 5: Build

If selected artifact is PR:

1. Create branch `hendeka/<issue-number>-<slug>` via `GitOps`.
2. Configure pi-coding-agent session (§8.1).
3. Run `session.prompt(taskPrompt)`.
4. Run `testCommand` if configured.
5. Self-review: separate `pi-ai.completeSimple()` call with a different model family, seeing diff + acceptance criteria only. Pass/fail gate.
6. On pass: push via `Forge.createVerifiedCommit` (App) or `GitOps.push` (PAT); open PR with §5 template; label `hendeka-generated`.
7. On fail: abandon; comment on source issue.

If not PR: generate content per §5 template; post to tracking issue or designated channel.

**Budget defaults (configurable)**: `maxTurns: 50`, `maxUSD: 2.00`, `maxWallClockSeconds: 1200`. Enforced by `scope-check` extension returning `{ block: true, reason: "budget exceeded" }`.

### Phase 6: Notification
- One-line summary to `~/.hendeka/notifications.log`.
- Comment on tracking issue if configured.
- Update manifest `stance` field from **explicit** Maintainer signals on previous artifact (rejection comments, style comments only — not silent commit amendments). Bounded ≤200 chars/field. Reversible via `hendeka config <repo> edit`.
- Record in SQLite.

---

## 5. Artifacts

Five types. Each includes mandatory "Why this one today" section citing metrics/manifest/KPI signals.

| Type | Trigger | Location |
|---|---|---|
| PR | Phase 4 selected Build candidate; Phase 5 completed + self-reviewed | New branch + PR on target repo |
| Triage Digest | Phase 2 flagged ≥30 untriaged or ≥5 spam | Comment on tracking issue |
| Review Digest | Phase 3 flagged ready-to-merge, no Build candidate won | Comment on tracking issue |
| Priority Recommendation | KPI signal without matching issue | New draft issue |
| Manifest Drift Alert | Cartography drift flag fired | Comment on tracking issue |

All artifacts include footer: run ID, behavior mode, next run time, `hendeka continue <run-id>` steering hint.

### PR template (mandatory structure)

```markdown
## Why this one today
[one paragraph citing specific metrics/manifest/issue signals]

## Changes
[bulleted summary of diff]

## Verification
- Tests: <command> → pass/fail
- Files changed: <count>
- Self-review: <one-line verdict>

## Self-review notes
[optional: caveats, flagged concerns, follow-ups]

## Steer this session
If this PR isn't quite right, resume the Build session:
`hendeka continue <run-id>`

---
🤖 hendeka[bot] · run <id> · mode: <mode> · next run: <iso8601>
```

---

## 6. CLI and configuration

### 6.1 Surface

```
hendeka                                   # state-aware: today if present, else next run time
hendeka add <url>                         # onboard; launches 3-step config wizard
hendeka config                            # instance config (tracing, custom providers)
hendeka config <repo>                     # interactive per-repo wizard
hendeka config <repo> set <key>=<value>   # non-interactive set
hendeka config <repo> show [--json]       # print effective config
hendeka config <repo> edit                # open manifest in $EDITOR; JSON Schema validated on save
hendeka today [<repo>]                    # display current artifact
hendeka status [<repo>]                   # last run, next run, behavior mode
hendeka metrics [<repo>]                  # current metrics, deltas
hendeka scope <repo> --set N              # shortcut for config <repo> set scope.level N
hendeka run <repo>                        # manual run (Sabbath rule: scheduled runs only)
hendeka trace <run-id>                    # view trace
hendeka open <run-id>                     # resume Build session in pi TUI (read-only)
hendeka continue <run-id>                 # resume with steering; human-driven continuation
hendeka eval --fixture <path>             # eval harness
hendeka docs <topic>                      # offline docs
```

### 6.2 `hendeka add` / config wizard

`hendeka add <url>` creates `~/.hendeka/repos/<slug>/`, invokes `hendeka config <slug>` wizard, installs scheduler, runs first Cartography.

**Three mandatory steps** (minimum viable setup):

1. **Model.** Provider menu (Ollama default-highlighted, then OpenRouter, Anthropic, OpenAI, Google, custom). For Ollama: autodetect via `ollama list`; prompt to pull missing models. For paid: prompt for env-var name holding API key. Validate via test `pi-ai.completeSimple()` call before accepting.
2. **Forge.** Detect GitHub App install; if absent, print install URL and wait. PAT fallback with explicit Verification-loss warning. Validate via `Forge.listOpenIssues()`.
3. **Timezone.** Detect from system; confirm. State Sabbath rule and 11:11 default; confirm. Install scheduler entry.

Then prompt: "Configure advanced options now, or accept defaults?" (default: accept). Advanced walks: scope level, budget, test command, acceptance criteria, observability exporter, KPI endpoints, stance. Each skippable. All editable later via `hendeka config <repo>`.

**Flag-driven** (automation): `hendeka add <url> --model ollama:qwen2.5-coder:32b --forge github-app --tz America/Chicago --accept-defaults` skips the wizard.

**Atomicity**: nothing written until final confirmation. Cancellation leaves filesystem and scheduler pre-wizard. Error mid-commit rolls back (removes repo dir + scheduler entry).

### 6.3 Manifest (`~/.hendeka/repos/<slug>/manifest.yaml`)

Written by wizard. Not committed to target repo. Maintainers can edit via `hendeka config <repo> edit` (JSON Schema validated on save).

```yaml
repo:
  url: https://github.com/owner/repo
  default_branch: main
  forge: github

model:
  provider: ollama                    # or: openrouter, anthropic, openai, google, vllm, ...
  build: qwen2.5-coder:32b            # Phase 5
  triage: qwen2.5-coder:7b            # Phases 1–3
  prioritization: qwen2.5-coder:32b   # Phase 4
  review: llama3.1:70b                # self-review; different family from build

schedule:
  time: "11:11"
  tz: America/Chicago
  # days not configurable; Sabbath is immutable

scope:
  level: 1                            # 0..3; 4 not in v0

test_command: "pnpm test"
acceptance_criteria:
  - "pnpm test passes"
  - "no files outside scope changed"

kpi_endpoints: []
observability:
  tracing_exporter: none              # or: langfuse, otlp

stance:
  style: "terse, no emoji in PR titles"
  avoid: ["deprecated/**"]

behavior_mode: normal                 # written by Phase 0
metrics_rolling: {}                   # written by Phase 0
```

### 6.4 Instance config (`~/.hendeka/config.yaml`)

```yaml
tracing:
  exporter: none                      # or: langfuse, otlp
  langfuse:
    host: https://cloud.langfuse.com
    public_key_env: LANGFUSE_PUBLIC_KEY
    secret_key_env: LANGFUSE_SECRET_KEY

providers:                            # custom pi-ai providers via registerProvider
  - name: my-vllm
    base_url: http://localhost:8000/v1
    api: openai-completions
    models:
      - id: Qwen2.5-Coder-32B
```

Secrets: always env-var name (`*_env` suffix). Never the value. All schemas exported as JSON Schema to `docs/schemas/`.

---

## 7. Interfaces (Hendeka-owned)

All types in §11. This section is the operational contract.

### 7.1 `GitOps`

`simple-git` + shell fallback. Plain git: clone, fetch, branch, commit, push, getHead. Works against any git remote.

### 7.2 `Forge`

Forge-neutral vocabulary: `ChangeRequest` = PR (GitHub) / MR (GitLab) / PR (Gitea/Forgejo). Methods: list and get issues/CRs; get CI status and auth identity; comment, close, review, open CR, add label; `createVerifiedCommit`.

**GitHub implementation** (v0):
- Both GitHub App (`createAppAuth`) and PAT auth through same interface.
- `createVerifiedCommit` routes through REST Git Data API (blobs → trees → commits → refs) when App auth active. `git commit` + forced committer is not Verified.
- DCO detection at install time; warning surfaced; artifacts labeled `dco-unsigned` when DCO-enforcing.
- Rate limit ceiling: 5,000 req/hr minimum on App installation tokens.

All writes pass through scope check and append to audit log.

**v0.x stubs** (GitLab, Forgejo): throw `{ kind: "not_implemented" }` at runtime; satisfy type; pass contract tests.

### 7.3 `KPIAdapter`

Plain HTTP with per-endpoint config. Secrets read from env-var names at runtime.

```yaml
kpi_endpoints:
  - name: error_rate
    url: https://...
    method: GET
    auth_header_env: MONITOR_TOKEN
    jsonpath: $.current
    unit: "%"
```

No MCP in v0.

---

## 8. Build runtime (pi-agent-core)

Phase 5 runs in-process. Full pi-mono API in §12.

### 8.1 Session configuration

```typescript
// src/phases/build/session.ts
import { createAgentSession, SessionManager } from "@mariozechner/pi-coding-agent";
import { getModel } from "@mariozechner/pi-ai";
import { buildSandboxedTools } from "../../runtime/sandbox-tools";
import { loadHendekaExtensions } from "../../runtime/extensions";
import { hendekaStreamFn } from "../../runtime/stream-fn";

async function runBuildPhase(input: BuildPhaseInput, runId: string): Promise<BuildResult> {
  const model = getModel(input.modelProvider, input.modelId);
  const sessionFile = `${hendekaHome()}/sessions/${runId}/build.jsonl`;
  const sessionManager = SessionManager.open(sessionFile);

  const { session } = await createAgentSession({
    model,
    thinkingLevel: input.thinkingLevel ?? "off",
    sessionManager,
    customTools: buildSandboxedTools(input.workdir),
    // Built-in pi tools disabled via setActiveTools([])
  });

  session.agent.streamFn = hendekaStreamFn;
  loadHendekaExtensions(session, input);

  try {
    await session.prompt(buildSystemPrompt(input));
  } finally {
    session.dispose();
  }

  return collectResult(sessionManager, input.workdir);
}
```

### 8.2 Hendeka extensions (required on every Build session)

Under `src/runtime/extensions/`. Registered via `loadHendekaExtensions(session, input)`.

| Extension | Event(s) | Responsibility |
|---|---|---|
| `scope-check` | `tool_call` | Scope ladder + budget enforcement (maxTurns, maxUSD, maxWallClock). Returns `{ block: true, reason }` on violation. |
| `context-prune` | `context` | Trim tool results >5KB older than 10 messages to one-line summary. Preserve most recent failure signal verbatim. Preserve all tool calls (trajectory). |
| `compaction-policy` | `session_before_compact` | Replace pi's default summarizer. Preserve file-ops history, tool-failure data, originating task. |
| `trace-bridge` | all events + `before_provider_request` | Write to `~/.hendeka/traces/<run-id>.jsonl`, SQLite `trace_events`, OTel spans. |

### 8.3 Sandboxed tools

`buildSandboxedTools(workspace)` returns:
- `createReadTool(workspace)`
- `createBashTool(workspace, { operations: { exec: sandboxedExec } })` — pi's default exec never invoked
- `createEditTool(workspace)`
- `createWriteTool(workspace)`

Built-in pi tools disabled. Hendeka-specific tools (`get_manifest`, `get_issue_thread`, `run_acceptance_test`) register via `customTools` if added.

### 8.4 Interactive handoff

Every Build session is a resumable JSONL file.

- **`hendeka open <run-id>`** — launches `pi --resume ~/.hendeka/sessions/<run-id>/build.jsonl`. Read-only exploration.
- **`hendeka continue <run-id>`** — resumes with steering enabled. Maintainer types guidance; agent continues.

**Extension set in interactive mode**:

| Extension | Scheduled | Interactive |
|---|---|---|
| `scope-check` | active | **active** (safety non-negotiable) |
| `budget-check` (part of scope-check) | active | **relaxed** (human is the budget) |
| `context-prune` | active | active |
| `compaction-policy` | active | active |
| `trace-bridge` | active | active |

**Scope escalation blocked in interactive mode.** Scope is repo-level, not session-level. To escalate: `hendeka config <repo> set scope.level N` first (with diff-preview confirmation), then `hendeka continue`. Scope-check reads live manifest value every tool call.

**Model choice**: defaults to original session's model; Maintainer can switch via pi's `/model` command. `hendekaStreamFn` still wraps all calls.

**Continuation commits** push through same `Forge.createVerifiedCommit` path; PR footer updates with `edited interactively · <timestamp>`. No new artifact created.

---

## 9. Safety

### 9.1 Scope ladder

| Level | Default? | Permits |
|---|---|---|
| 0 | no | read only; output to stdout/log |
| 1 | yes (after `hendeka add`) | comment on issues and CRs |
| 2 | no | close spam issues and abandoned CRs |
| 3 | no | push branches and open CRs |
| 4 | not in v0 | merge CRs |

Escalation: `hendeka scope <repo> --set N` prints permissions diff, requires explicit confirmation.

### 9.2 Sandboxing

**Linux stack** (all layered, all applied):
- bubblewrap (filesystem isolation)
- Landlock LSM (unprivileged FS access control; kernel 5.13+, ABI v6)
- seccomp-bpf (via bubblewrap; dangerous syscalls blocked)
- `PR_SET_NO_NEW_PRIVS`
- Scoped egress proxy (socat or similar): allowlist only `api.github.com`, configured model endpoint, configured KPI endpoints, declared package registries
- Separate `hendeka-build` unprivileged user (not orchestrator's user)
- Read-only repo bind mount; writable tmpfs for scratch

Preflight: `journalctl -kb -g landlock` must show kernel support; else bubblewrap-only with warning.

**macOS stack**: `sandbox-exec` with restrictive Seatbelt profile matching Linux allowlist.

**Integration**: `createBashTool(workspace, { operations: { exec: sandboxedExec } })`. This is the documented pi-mono operations-override pattern.

**Fallback**: `--unsafe` flag required if neither stack available. Logs WARN every run; surfaces in `hendeka today` output.

### 9.3 Target-repo AGENTS.md precedence

Target-repo `AGENTS.md`/`CLAUDE.md`/`.cursor/rules/*` read into Build session's reference context (informs repo conventions). Does NOT override Hendeka's safety extensions. Scope, sandbox, bot-to-bot rules, Sabbath come from Hendeka config — never from target repo. A target repo cannot instruct Hendeka to escalate scope, disable sandboxing, or run on weekends.

### 9.4 Bot-to-bot

Default ignore list: `dependabot[bot]`, `renovate[bot]`, `github-actions[bot]`, `codecov[bot]`. Hendeka does not comment on, close, or review their CRs unless Maintainer adds `hendeka/review-this` label.

---

## 10. Circuit breaker

`src/core/circuit-breaker.ts`. Sliding window N=10 runs. Thresholds are v0 starting calibration (named constants; retuned from telemetry in v0.x).

| State | Trigger | Mode | Effect |
|---|---|---|---|
| Closed | all thresholds met | `normal` | default |
| Half-open | acceptance <50% | `conservative` | tighter self-review; smaller Build scope; cheaper model via `agent.setModel()` |
| Half-open | false-action >5% | `triage_limited` | Level 2 closes suspended 7 days |
| Half-open | clean merge <40% | `scoped` | Build scope shrunk; tests required before push |
| Half-open | CI pass <70% | `pr_blocked` | no PR artifacts; prefer digests |
| Open | acceptance <40% for 3 weeks | `paused` | auto-downshift to Level 0; manual re-escalation |

**Half-open probe protocol**: one artifact in restricted mode; observe 24–72h; clean probe steps down one level; two consecutive clean probes return to `normal`.

Current mode surfaces in every artifact footer, `hendeka today`, `hendeka metrics`.

---

## 11. Model provider (via pi-ai)

No custom abstraction. pi-ai provides it.

### 11.1 Four provider paths

**1. Local (Ollama) — documented default**:
```yaml
model:
  provider: ollama
  build: qwen2.5-coder:32b
  triage: qwen2.5-coder:7b
```

**2. BYOK gateway (OpenRouter)**:
```yaml
model:
  provider: openrouter
  build: anthropic/claude-sonnet-4.7
  triage: google/gemini-2.5-flash
```

**3. Direct provider (Anthropic, OpenAI, Google, ...)**:
```yaml
model:
  provider: anthropic
  build: claude-opus-4.7
```

**4. Self-hosted (vLLM, LiteLLM, MLX, custom)**:
```yaml
model:
  provider: vllm
  base_url: http://localhost:8000/v1
  build: Qwen2.5-Coder-32B
```
Registered via `pi.registerProvider` in instance config.

### 11.2 `hendekaStreamFn` wrapper

Set on every pi-agent-core session: `session.agent.streamFn = hendekaStreamFn`.

- **OpenRouter**: inject `allow_fallbacks: false` + pinned `order`; set `X-Title: Hendeka` / `HTTP-Referer`; capture `provider_responses` → trace.
- **Anthropic** (direct or via OpenRouter): set `cacheRetention: "long"`.
- **All providers**: emit `model_call` trace event (model, tokens, cost, latency).
- **Ollama**: no-op wrapper.

### 11.3 Model selection policy

- Triage (phases 1–3): small/cheap.
- Prioritization (phase 4): mid-tier reasoning.
- Build (phase 5): Maintainer's choice.
- Self-review: mid-tier, **different model family from Build** (enforced by separate `pi-ai.completeSimple()` call).
- `conservative` mode: downgrade via `agent.setModel(cheaperModel)`.

---

## 12. Tech stack

| Component | v0 | Package | Abstracted by |
|---|---|---|---|
| Language | TypeScript | — | — |
| Runtime | Node.js ≥20 | — | — |
| Package manager | pnpm | — | — |
| Agent runtime | pi-mono (MIT, pinned) | `@mariozechner/pi-ai`, `@mariozechner/pi-agent-core`, `@mariozechner/pi-coding-agent` | none |
| Model provider | via pi-ai | pi-ai's built-in catalog | pi-ai (§11) |
| Forge | Octokit (GitHub) | `@octokit/rest` | `Forge` (§7.2) |
| Git | simple-git | `simple-git` | `GitOps` (§7.1) |
| KPI | fetch + jsonpath-plus | `jsonpath-plus` | `KPIAdapter` (§7.3) |
| State | SQLite | `better-sqlite3` | — |
| Tracing | OTel + Langfuse | `@opentelemetry/api`, `langfuse` | — |
| Config | YAML | `yaml` | — |
| Logging | Pino JSON | `pino` | — |
| CLI | Commander | `commander` | — |
| Scheduler | systemd / launchd / node-cron fallback | `node-cron` | — |
| Sandbox | bubblewrap + Landlock (Linux), sandbox-exec (macOS) | shell via `operations.exec` | — |

All deps MIT or Apache-2.0. No required paid services. `@mariozechner/pi-*` packages pinned to exact versions.

**pi-mono upgrade cadence** (CI-enforced):
- Patch (`0.x.Y`): auto-merged on green three-provider eval.
- Minor (`0.X.y`): Maintainer-reviewed after eval green.
- Major (`X.y.z`): planning issue first, no auto-merge.

`scripts/vendor-pi-mono.sh` provides defensive vendoring path.

---

## 13. Codebase conventions

Strict TypeScript: `strict`, `noImplicitAny`, `noUncheckedIndexedAccess`. No `any` outside generated code. `Result<T, E>` discriminated unions for errors. Explicit parameter passing. No DI frameworks. No globals. No module-level mutable state.

### 13.1 File layout (enforced)

```
/src
  /core              # shared types, Result<T,E>, circuit breaker
  /phases
    /metric-refresh
      index.ts       # phase entry
      types.ts
      prompts.ts
      README.md
    /cartography
    /issue-triage
    /pr-triage
    /prioritization
    /build
    /notification
  /runtime           # pi-mono integration
    stream-fn.ts
    sandbox-tools.ts
    extensions/
      scope-check.ts
      context-prune.ts
      compaction-policy.ts
      trace-bridge.ts
      index.ts       # loadHendekaExtensions
      README.md
  /adapters
    /forge
      index.ts
      github.ts
      gitlab.ts      # v0.x stub
      forgejo.ts     # v0.x stub
      README.md
    /kpi
      index.ts
      http-jsonpath.ts
      README.md
    /git-ops
      index.ts
      simple-git.ts
      README.md
  /cli               # one file per command
  /state             # SQLite schemas and typed access
  /tracing           # OTel + Langfuse exporter
/tests               # mirrors /src 1:1
/fixtures            # eval harness inputs
/docs
  prd.md             # this document
  schemas/           # exported JSON Schemas
/scripts
  vendor-pi-mono.sh
AGENTS.md            # human-curated, ≤200 lines
CLAUDE.md            # symlink → AGENTS.md
CONTRIBUTING.md
```

Every phase dir: exactly the four files. Every adapter: `index.ts` + ≥1 impl + `README.md`. v0.x stubs throw `{ kind: "not_implemented" }` at runtime; satisfy type; pass contract tests.

### 13.2 AGENTS.md discipline

- Human-curated, ≤200 lines. Agents read it; they don't edit it.
- Never auto-generated.
- Hendeka is prohibited from creating/proposing AGENTS.md for target repos.
- Contents: pinned build/test/lint commands, PR conventions, file-layout enforcement, "what agents cannot infer" section, type-ownership rules, pi-mono API summary (from §12).

### 13.3 Never let the same agent write and judge

Self-review uses a different model family from Build. Enforced by separate `pi-ai.completeSimple()` call with distinct provider config.

### 13.4 Contract-passes-under-N

Two forcing functions:
- **Domain layer**: `Forge` contract test suite passes under GitHub live + GitLab stub + Forgejo stub. Same for `KPIAdapter`, `GitOps`.
- **Runtime layer**: eval harness passes under Ollama + OpenRouter + Anthropic direct.

Provider-specific logic lives in `hendekaStreamFn` or extensions. Never in phase code. If a provider requires special-casing in phases, the interface leaked.

---

## 14. Observability

Three layers, always on:

1. **JSONL**: `~/.hendeka/traces/<run-id>.jsonl`. Plain text, greppable.
2. **SQLite**: `trace_events` table. Indexed for `hendeka trace` and metric computation.
3. **OTel spans**: every phase, every pi-ai call, every tool call, every forge request.

Build phase trace source: `session.subscribe()` native event stream. The `trace-bridge` extension converts every pi-agent-core event to a Hendeka trace event.

**Exporters** (in `~/.hendeka/config.yaml`):
- `none` — JSONL + SQLite only (default)
- `langfuse` — Langfuse SDK (OSS, self-hostable)
- `otlp` — OpenTelemetry Protocol to any compatible backend

---

## 15. Deployment

### 15.1 Lifecycle

No long-running daemon. Install → onboard → scheduled invocation → exit. State in SQLite between runs. Resumable from any crash.

At 11:11 M–F: scheduler invokes `hendeka run <slug> --scheduled`. Hendeka starts, runs six phases, writes artifact + trace, exits. Between runs: nothing running.

Forge identity (`hendeka[bot]`) separate from process lifecycle. Authenticate, act, exit. Nothing stays authenticated.

### 15.2 Scheduler installers

- **Linux (systemd)**: user timer `~/.config/systemd/user/hendeka@<slug>.timer`, service `hendeka@<slug>.service`. Enabled on install. Runs as Maintainer's user; Build phase drops to `hendeka-build` via bubblewrap.
- **macOS (launchd)**: `~/Library/LaunchAgents/dev.hendeka.<slug>.plist`.
- **Fallback (cron)**: `hendeka install cron` generates crontab entry.
- **Last-resort (node-cron)**: `hendeka daemon start` runs lightweight userspace scheduler.

All installers honor Sabbath (§1.1): cron expression excludes weekends AND pipeline entry asserts. Both must agree.

### 15.3 Platform support

Requires: Node.js ≥20, git. Optional: bubblewrap (Linux), ollama (if using local models).

Supports: Linux (Debian 12+, Ubuntu 22.04+, Fedora 40+, Arch), macOS 14+, Raspberry Pi 4/5 with 4 GB+ RAM.

Does not support: Windows native (WSL works, not tested).

---

## 16. Build order

Sequential phases; steps within parallelizable.

### Phase A — skeleton

1. Repo setup per §13 (TypeScript strict, pnpm, Commander, directory layout).
2. Human-curated `AGENTS.md` (≤200 lines). `CLAUDE.md` symlink.
3. Install and pin pi-mono packages. `scripts/vendor-pi-mono.sh`.
4. Contract test framework for `Forge`, `KPIAdapter`, `GitOps`.
5. Domain interface files only (no implementations).
6. v0.x `Forge` stubs (GitLab, Forgejo) passing contract tests.
7. SQLite schema: `repos`, `runs`, `traces`, `trace_events`, `metrics_runs`, `metrics_rolling`, `artifacts`, `scope_changes`.
8. OTel + Langfuse wiring (`src/tracing/`).
9. **Runtime integration layer** (`src/runtime/`): `stream-fn.ts`, `sandbox-tools.ts`, four extensions, `loadHendekaExtensions`.
10. `GitOps` via simple-git.
11. Register Hendeka GitHub App; store ID + private key in env.
12. `Forge` GitHub impl with App + PAT. `createVerifiedCommit` via Git Data API.
13. `hendeka config` wizard (three-step core + optional advanced). Atomic write. Model validation via test pi-ai call. Forge validation via `listOpenIssues`. Installs scheduler with Sabbath enforcement in cron AND pipeline entry.
14. `hendeka add <url>` wraps: create manifest dir, invoke `hendeka config`, run first Cartography.
15. Bare `hendeka` state-aware routing.
16. `hendeka config` subcommands: `set`, `show`, `edit`. Instance-level + per-repo. JSON Schema validation on save.
17. Config loading resolves env-var references for secrets; never logs values.

### Phase B — pipeline

18. Phase 1 Cartography: shallow clone, target-repo AGENTS.md/CLAUDE.md/.cursor/rules ingestion, forge fetch, KPI fetch, manifest build.
19. Manifest diff + drift flags.
20. Phases 2 + 3 triage with scope gating.
21. Phase 4 prioritization (rule-based type selection).
22. Phase 5 Build: session config per §8.1; push via `Forge.createVerifiedCommit` or `GitOps.push`; open CR.
23. Self-review: separate `pi-ai.completeSimple()` with different model family.
24. Scope ladder enforcement at every forge write and every Build tool call.
25. **Pi prompt templates**: at minimum PR description, self-review rubric, Phase 5 system-prompt header shipped under `~/.pi/agent/prompts/hendeka/`. Phase code references by name, not inline.

### Phase C — metrics, breaker, non-PR artifacts, interactive

26. Phase 0 metric refresh.
27. Circuit breaker (`src/core/circuit-breaker.ts`) with unit tests for every mode transition + half-open probe.
28. `behavior_mode` surfacing in manifest + footer + `hendeka today` + `hendeka metrics`.
29. Non-PR artifact types.
30. Trace writer end-to-end (JSONL + SQLite + OTel + Langfuse).
31. CLI: `hendeka today`, `hendeka status`, `hendeka metrics`, `hendeka trace`.
32. Interactive handoff: `hendeka open <run-id>`, `hendeka continue <run-id>`. Extension set per §8.4. Scope escalation blocked in interactive mode. Continuation commits push through same verified-commit path.
33. Phase 6 stance update (conservative: explicit signals only).

### Phase D — ship

34. Eval harness: `hendeka eval --fixture <path>`. Runs under three providers; all above threshold.
35. Sandbox hardening: bubblewrap + Landlock + seccomp + egress proxy (Linux); sandbox-exec (macOS); `hendeka-build` user; denied-connection integration test.
36. Scheduler installers (systemd, launchd, cron, node-cron fallback). **Sabbath 14-day clock-advance test**.
37. Docs pages per §17. Quickstart leads with Ollama. Maintainers complete setup without hand-editing YAML.
38. Bot-to-bot rules; `hendeka[bot]` App publication to GitHub Marketplace.
39. End-to-end: Hendeka on its own repo produces real PR passing contract tests + CI. First merged PR is the v0 shipping event.

---

## 17. Documentation (required v0 pages)

Under `docs/`:

- `quickstart.md` — **Ollama path first** (<10 min, zero paid services); then OpenRouter BYOK path (<10 min).
- `install.md` — package install + scheduler + GitHub App.
- `config.md` — three-step wizard, advanced options, flag-driven install, `hendeka config` subcommands, manifest schema for all four provider paths.
- `interactive.md` — `hendeka open` / `hendeka continue`; what changes in interactive mode; steering examples.
- `lifecycle.md` — no-daemon lifecycle; what runs when; where state lives; debugging failed runs.
- `scope.md` — scope ladder + escalation.
- `metrics.md` — each metric; reading the footer; behavior modes.
- `artifacts.md` — all five types with examples.
- `troubleshooting.md` — linked from every CLI error.
- `architecture.md` — phases + runtime + extensions; pi-mono relationship.
- `contributing.md` — pointer to `AGENTS.md`.

Static site at `hendeka.dev`; bundled with package for offline `hendeka docs <topic>`.

---

## 18. Definition of Done (v0)

All must hold. One issue per item. One DoD item per Claude Code task.

1. `pnpm install -g hendeka && hendeka add <url>` works on fresh Debian 12, macOS 14, Raspberry Pi OS.
2. Quickstart <10 min median for 3 unfamiliar Maintainers on Ollama path, no YAML hand-edits. Separate gate for OpenRouter path.
3. `hendeka config <repo>` wizard: three-step core (model, forge, timezone) with validated inputs (pi-ai call; `listOpenIssues`). Advanced skippable. Atomic write on final confirm.
4. `hendeka config <repo>` {`set`, `show`, `edit`} functional. Edit validates on save via JSON Schema; rejects invalid.
5. `hendeka config` (instance) writes tracing exporter + custom pi-ai providers.
6. First scheduled run fires at 11:11 M–F; produces artifact or zero-artifact explanation.
7. **Sabbath enforced in all four places (§1.1).** 14-day clock-advance integration test: zero weekend runs.
8. `hendeka` with no args routes correctly based on state.
9. `hendeka today` output includes `Session: hendeka open <run-id>` discoverability line.
10. Artifact footer: run ID, mode, next run time, `hendeka continue <run-id>` hint.
11. PR template includes "Why this one today" and "Steer this session" blocks.
12. Bot commits GitHub-Verified via Git Data API (App auth); trailers applied (PAT).
13. Scope level 1 default after `hendeka add`; escalation requires explicit command + diff preview.
14. DCO warning on install for DCO-enforcing repos.
15. Metrics 1–5, 7–9 compute and surface. Metric 6 tracked from first merged PR.
16. Sandboxed Build does not network outside allowlist (integration test asserts denied connection).
17. No secrets in config files (grep check + code review). Env-var name references only.
18. Linux: bubblewrap + Landlock (when supported) + seccomp + egress proxy default.
19. macOS: sandbox-exec default.
20. Build phase runs as `hendeka-build` user.
21. `--unsafe` is the only disable path; logs WARN every run; surfaces in `hendeka today`.
22. Domain contract-passes-under-N: `Forge` suite passes GitHub live + GitLab stub + Forgejo stub.
23. Runtime three-provider eval: harness passes under Ollama, OpenRouter, Anthropic direct. Ollama is documented default.
24. `GitOps` and `Forge` separate; no phase code imports Octokit or forge-specific types.
25. No custom `ModelProvider` or `BuildEngine` interfaces. LLM calls through `pi-ai.streamSimple`/`completeSimple` via `hendekaStreamFn`. Build uses `createAgentSession()`.
26. `hendekaStreamFn` on OpenRouter: `allow_fallbacks: false`, pinned `order`, attribution headers, `provider_responses` to trace. On Anthropic: `cacheRetention: "long"`. On Ollama: no-op.
27. Strict TypeScript in CI; no `any` outside generated code.
28. `hendeka eval --fixture <path>` runs shipped fixture; produces score.
29. OTel spans for every phase, pi-ai call, tool call, forge request. JSONL + SQLite always on. Langfuse and OTLP exporters tested. Default `none`.
30. Circuit breaker unit-tested for every mode transition including half-open probe.
31. Current `behavior_mode` visible in footer, `hendeka today`, `hendeka metrics`.
32. All CLI errors include docs links. All §17 doc pages exist, including `interactive.md`, `config.md`, `lifecycle.md`.
33. Pi prompt templates: ≥3 phase prompts under `~/.pi/agent/prompts/hendeka/` (PR description, self-review rubric, Phase 5 system-prompt header). Phase code references by name, not inline.
34. Interactive handoff:
    - `hendeka open <run-id>` launches pi with session resumed, full history.
    - `hendeka continue <run-id>` resumes with steering; scope-check active; budget relaxed; other extensions active.
    - Scope escalation blocked (scope-check reads live manifest; interactive cannot raise).
    - Continuation commits push through same verified-commit path; PR footer updates.
35. Reflexive stance update (conservative): Phase 6 updates manifest `stance` from explicit Maintainer signals only (rejection comments, style comments). Silent commit amendments do NOT trigger. Bounded ≤200 chars/field. Reversible.
36. `AGENTS.md` ≤200 lines, human-curated. CI rejects PRs that bulk-regenerate it.
37. Hendeka does NOT write `AGENTS.md` for any target repo (integration test asserts none created).
38. Claude Code (or equivalent) adds new `Forge` implementation following only `AGENTS.md` + contract tests. Verified by implementing GitLab against v0 stub.

---

## 19. Out of scope for v0

- Custom `ModelProvider` / `BuildEngine` abstractions.
- GitLab / Forgejo implementations (stubs ship; impls v0.x).
- Hendeka owning local-model deployment (`pi-pods` handles this).
- Multi-repo cross reasoning.
- Web UI.
- Push notifications to external channels (email, Slack, Discord).
- Windows native.
- IDE integration.
- Level 4 scope (merging own PRs).
- Writing `AGENTS.md` for target repos.
- Phone-home telemetry.
- MCP support.
- GitHub Actions runner.
- Weekend operation.
- pi skills (deferred to v0.x; prompt templates ship in v0).

---

## 20. Pi capabilities leveraged

**v0 required**: `createAgentSession`; `SessionManager` with JSONL persistence; extension system (`tool_call` blocking, `context` rewriting, `session_before_compact`); `streamFn` replacement; sandboxable tool factories (`createBashTool` with `operations` override); native event stream via `session.subscribe()`; auto-compaction hook; runtime model switching (`agent.setModel()`).

**v0 recommended**: prompt templates under `~/.pi/agent/prompts/hendeka/`.

**v0.x roadmap**: skills (`~/.pi/agent/skills/`); session forking for reflexive retry; inter-extension event bus (`pi.events`); custom slash commands (`pi.registerCommand`); tree navigation (`ctx.navigateTree`) for replay debugging.

**Out of scope**: MCP (pi-mono does not support).

---

## 21. Pi-mono gaps (document to prevent waiting)

- No MCP support. Do not add MCP servers.
- No built-in cross-run memory store. Manifest `stance` field (§6.3) is the memory.
- No native circuit breaker. Hendeka ships in `src/core/circuit-breaker.ts` (§10).
- No long-running daemon. Sessions started by CLI, torn down on `session.dispose()`.
- No built-in cost enforcement. `scope-check` extension counts usage events, blocks on budget.

---

## 22. Type appendix

Copy into `src/core/types.ts`.

```typescript
// ---------- Primitives ----------

export type Sha = string & { readonly __brand: "Sha" };
export type RunId = string & { readonly __brand: "RunId" };
export type IsoTimestamp = string & { readonly __brand: "IsoTimestamp" };

export type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

// ---------- Scope and behavior modes ----------

export type ScopeLevel = 0 | 1 | 2 | 3;   // 4 not in v0

export type BehaviorMode =
  | "normal"
  | "conservative"
  | "triage_limited"
  | "scoped"
  | "pr_blocked"
  | "paused";

// ---------- Forge layer ----------

export type Forge = {
  readonly id: string;
  readonly capabilities: ForgeCapabilities;

  listOpenIssues(): Promise<Issue[]>;
  listOpenChangeRequests(): Promise<ChangeRequest[]>;
  getIssue(n: number): Promise<IssueDetail>;
  getChangeRequest(n: number): Promise<ChangeRequestDetail>;
  getCIStatus(crNumber: number): Promise<CIStatus>;
  getAuthIdentity(): Promise<AuthIdentity>;

  commentOnIssue(n: number, body: string): Promise<void>;
  commentOnChangeRequest(n: number, body: string): Promise<void>;
  closeIssue(n: number, comment?: string): Promise<void>;
  closeChangeRequest(n: number, comment?: string): Promise<void>;
  createReview(crNumber: number, review: Review): Promise<void>;
  openChangeRequest(params: OpenChangeRequestParams): Promise<ChangeRequest>;
  addLabel(issueOrCR: number, label: string): Promise<void>;

  createVerifiedCommit(params: VerifiedCommitParams): Promise<Sha>;
};

export type ForgeCapabilities = {
  verifiedBotCommits: boolean;
  commitSigningAPI: boolean;
  draftChangeRequests: boolean;
  requiredReviewers: boolean;
  ciStatusQuery: boolean;
  labels: boolean;
};

export type AuthIdentity = {
  kind: "app" | "pat" | "oauth";
  login: string;
};

export type Issue = {
  number: number;
  title: string;
  author: string;        // `[bot]` suffix marks bots per forge convention
  createdAt: IsoTimestamp;
  updatedAt: IsoTimestamp;
  labels: string[];
  state: "open" | "closed";
};

export type IssueDetail = Issue & {
  body: string;
  comments: Comment[];
};

export type Comment = {
  author: string;
  body: string;
  createdAt: IsoTimestamp;
};

export type ChangeRequest = Issue & {
  sourceBranch: string;
  targetBranch: string;
  draft: boolean;
};

export type ChangeRequestDetail = ChangeRequest & {
  body: string;
  comments: Comment[];
  reviews: ReviewState[];
  filesChanged: number;
  additions: number;
  deletions: number;
};

export type ReviewState = {
  reviewer: string;
  state: "approved" | "changes_requested" | "commented";
  submittedAt: IsoTimestamp;
};

export type CIStatus = {
  state: "pending" | "success" | "failure" | "error" | "unknown";
  checks: Array<{ name: string; state: CIStatus["state"]; url?: string }>;
};

export type Review = {
  event: "COMMENT" | "APPROVE" | "REQUEST_CHANGES";
  body: string;
  comments?: Array<{ path: string; line: number; body: string }>;
};

export type OpenChangeRequestParams = {
  title: string;
  body: string;
  sourceBranch: string;
  targetBranch: string;
  draft?: boolean;
  labels?: string[];
};

export type VerifiedCommitParams = {
  branch: string;
  baseSha: Sha;
  message: string;
  files: Array<{ path: string; content: string; mode?: "100644" | "100755" | "120000" }>;
};

// ---------- GitOps ----------

export type GitOps = {
  clone(url: string, dir: string, opts: { depth?: number; auth?: GitAuth }): Promise<void>;
  fetch(dir: string): Promise<void>;
  createBranch(dir: string, name: string, fromSha: Sha): Promise<void>;
  commit(dir: string, message: string, trailers: Record<string, string>): Promise<Sha>;
  push(dir: string, branch: string, auth: GitAuth): Promise<void>;
  getHead(dir: string): Promise<Sha>;
};

export type GitAuth =
  | { kind: "token"; token: string }
  | { kind: "ssh"; keyPath: string }
  | { kind: "app-installation"; installationToken: string };

// ---------- KPI ----------

export type KPIAdapter = {
  fetchAll(endpoints: KPIEndpoint[]): Promise<KPISignal[]>;
};

export type KPIEndpoint = {
  name: string;
  url: string;
  method: "GET" | "POST";
  auth_header_env: string;   // env var name, never the secret
  jsonpath: string;
  unit?: string;
  body?: string;
};

export type KPISignal = {
  name: string;
  value: number | string;
  unit?: string;
  fetchedAt: IsoTimestamp;
  error?: string;
};

// ---------- Build phase ----------

export type BuildPhaseInput = {
  runId: RunId;
  workdir: string;
  goal: string;
  relevantFiles: string[];
  testCommand?: string;
  acceptanceCriteria: string[];
  referenceContext: string;   // manifest excerpts + issue thread + target-repo AGENTS.md
  modelProvider: string;      // pi-ai provider id
  modelId: string;
  thinkingLevel?: "off" | "minimal" | "low" | "medium" | "high";
  budget: BuildBudget;
  scope: ScopeLevel;
};

export type BuildBudget = {
  maxTurns: number;
  maxUSD: number;
  maxWallClockSeconds: number;
};

export type BuildResult = {
  status: "completed" | "failed" | "abandoned";
  commits: Sha[];
  testsPassed: boolean | "not_run";
  cost: { tokensIn: number; tokensOut: number; usd: number };
  selfReviewNotes?: string;
  abandonReason?: string;
};

// ---------- Trace events ----------

export type TraceEvent =
  | { type: "phase_start"; runId: RunId; phase: PhaseName; ts: IsoTimestamp }
  | { type: "phase_end"; runId: RunId; phase: PhaseName; status: "ok" | "error"; ts: IsoTimestamp }
  | { type: "forge_request"; runId: RunId; method: string; path: string; ts: IsoTimestamp }
  | { type: "forge_response"; runId: RunId; status: number; latencyMs: number; ts: IsoTimestamp }
  | { type: "model_call"; runId: RunId; model: string; tokensIn: number; tokensOut: number; usd?: number; latencyMs: number; providerMetadata?: unknown; ts: IsoTimestamp }
  | { type: "tool_call"; runId: RunId; name: string; argsSummary: string; ts: IsoTimestamp }
  | { type: "tool_result"; runId: RunId; name: string; status: "ok" | "error"; summary: string; ts: IsoTimestamp }
  | { type: "decision"; runId: RunId; phase: PhaseName; decision: string; rationale: string; ts: IsoTimestamp }
  | { type: "artifact_draft"; runId: RunId; artifactType: ArtifactType; ts: IsoTimestamp }
  | { type: "artifact_posted"; runId: RunId; artifactType: ArtifactType; url?: string; ts: IsoTimestamp }
  | { type: "abandon"; runId: RunId; phase: PhaseName; reason: string; ts: IsoTimestamp }
  | { type: "metric_computed"; runId: RunId; metric: string; value: number; ts: IsoTimestamp }
  | { type: "circuit_breaker_state"; runId: RunId; mode: BehaviorMode; ts: IsoTimestamp };

export type PhaseName =
  | "metric_refresh"
  | "cartography"
  | "issue_triage"
  | "pr_triage"
  | "prioritization"
  | "build"
  | "notification";

export type ArtifactType =
  | "pr"
  | "triage_digest"
  | "review_digest"
  | "priority_recommendation"
  | "manifest_drift_alert";

// ---------- Errors ----------

export type HendekaError =
  | { kind: "scope_violation"; attempted: string; currentLevel: ScopeLevel; requiredLevel: ScopeLevel }
  | { kind: "budget_exceeded"; dimension: "turns" | "usd" | "wallclock"; limit: number; actual: number }
  | { kind: "forge_auth_failed"; detail: string }
  | { kind: "model_unreachable"; provider: string; detail: string }
  | { kind: "sandbox_unavailable"; platform: string; detail: string }
  | { kind: "sabbath_violation"; attemptedAt: IsoTimestamp }
  | { kind: "config_invalid"; path: string; detail: string }
  | { kind: "not_implemented"; what: string };
```

v0.x adapter stubs satisfy the interface by implementing all methods to throw `{ kind: "not_implemented" }`.

---

## 23. pi-mono API cheat sheet

### 23.1 Imports

```typescript
import { getModel, streamSimple, completeSimple, type Model } from "@mariozechner/pi-ai";
import { Agent, type AgentEvent } from "@mariozechner/pi-agent-core";
import {
  createAgentSession,
  SessionManager,
  createReadTool,
  createBashTool,
  createWriteTool,
  createEditTool,
  type ExtensionAPI,
  type ExtensionContext,
  isToolCallEventType,
} from "@mariozechner/pi-coding-agent";
import { Type } from "@sinclair/typebox";
```

### 23.2 Core surface

| Function | Shape | Used for |
|---|---|---|
| `getModel(provider, id)` | `(string, string) => Model` | Resolve any pi-ai model |
| `streamSimple(model, ctx, opts?)` | `(model, msgs[], { tools?, signal?, onEvent? }) => AsyncIterable<StreamEvent>` | Direct streaming; phases 0–4 |
| `completeSimple(model, ctx)` | `(model, msgs[]) => Promise<{ content, usage }>` | One-shot; self-review |
| `createAgentSession(opts)` | see §8.1 | Start Build session in-process |
| `SessionManager.open(path)` | `(string) => SessionManager` | Open/create JSONL session file |
| `session.prompt(text)` | `(string) => Promise<void>` | Send message; run loop to done |
| `session.agent.streamFn = fn` | assignment | Replace default stream fn |
| `session.subscribe(handler)` | `(fn) => { dispose: () => void }` | Trace bridge source |
| `agent.setModel(model)` | `(Model) => Promise<void>` | Runtime switch; conservative mode |
| `session.dispose()` | `() => void` | Always call in `finally` |

### 23.3 Extension patterns

```typescript
// src/runtime/extensions/<n>.ts
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

export default function(pi: ExtensionAPI) {
  // scope-check
  pi.on("tool_call", async (event, ctx) => {
    if (!scopeAllows(event.toolName, currentScopeLevel())) {
      return { block: true, reason: `Scope ${currentScopeLevel()} forbids ${event.toolName}` };
    }
  });

  // context-prune
  pi.on("context", async (event, ctx) => {
    return { messages: pruneOversizedToolResults(event.messages) };
  });

  // compaction-policy
  pi.on("session_before_compact", async (event, ctx) => {
    return {
      compaction: {
        summary: customSummary(event.preparation),
        firstKeptEntryId: event.preparation.firstKeptEntryId,
        tokensBefore: event.preparation.tokensBefore,
      },
    };
  });

  // trace-bridge
  pi.on("before_provider_request", (event, ctx) => {
    writeTrace({ type: "model_call", payload: event.payload, ts: now() });
  });
}
```

Events Hendeka uses:

| Event | When | Extension |
|---|---|---|
| `tool_call` | before tool executes; can block with `{ block: true, reason }` | scope-check |
| `context` | before each LLM call; can rewrite messages | context-prune |
| `session_before_compact` | before auto-compaction; can replace summary | compaction-policy |
| `before_provider_request` | after payload assembled, before HTTP | trace-bridge |
| `agent_start` / `agent_end` | once per `session.prompt()` | trace-bridge |
| `turn_start` / `turn_end` | each LLM response + tool cycle | trace-bridge |
| `tool_execution_start` / `tool_execution_end` | tool lifecycle | trace-bridge |
| `tool_result` | after tool; can modify result | (reserved) |

Return `{ block: true, reason }` from `tool_call` to cancel. Return `{ messages }` from `context` to replace. Return `undefined` to pass through.

### 23.4 Sandboxed tools

```typescript
import { createBashTool, createReadTool, createWriteTool, createEditTool } from "@mariozechner/pi-coding-agent";

const sandboxedBash = createBashTool(workspace, {
  operations: {
    exec: async (command, cwd, options) => {
      return runUnderBubblewrap(command, cwd, options);
    },
  },
});

const tools = [
  createReadTool(workspace),
  sandboxedBash,
  createEditTool(workspace),
  createWriteTool(workspace),
];

await createAgentSession({ model, sessionManager, customTools: tools });
```

### 23.5 Provider registration

```typescript
pi.registerProvider("my-vllm", {
  baseUrl: "http://localhost:8000/v1",
  api: "openai-completions",
  apiKey: "VLLM_API_KEY",  // env var name
  models: [{
    id: "Qwen2.5-Coder-32B",
    name: "Qwen2.5 Coder 32B (self-hosted)",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 131072,
    maxTokens: 8192,
  }],
});
```

### 23.6 Custom tool registration

```typescript
import { Type } from "@sinclair/typebox";

pi.registerTool({
  name: "get_manifest",
  label: "Get Manifest Section",
  description: "Read a slice of the Hendeka manifest for this repo",
  parameters: Type.Object({
    section: Type.String({ description: "Top-level manifest section name" }),
  }),
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    const value = readManifestSection(params.section);
    return {
      content: [{ type: "text", text: JSON.stringify(value, null, 2) }],
      details: { section: params.section },
    };
  },
});
```

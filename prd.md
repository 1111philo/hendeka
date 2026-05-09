# Hendeka — v0 PRD

**AI agents, harnesses, and orchestrators fall out of style too quickly to depend on them for truly autonomous code management.** Whatever you adopt today gets replaced; whatever you built around it gets thrown out with it.

**Hendeka lets you change the stack underneath without rebuilding what you've configured above.** Model, harness, forge, sandbox, and tracing sit behind adapters. Configuration, catalog, traces, and discipline sit above. Swap a layer below; the layer above keeps working.

Open Source because repo maintenance is critical infrastructure and should not be vendor-locked.

Hendeka aims to become a substrate that autonomously builds and maintains anything. **The OS for autonomous work.** v0 ships the autonomous Maintainer because it's the most demanding application the substrate could host today: it exercises every layer (model, harness, forge, sandbox, tracing) under real conditions. A substrate that supports the Maintainer can support whatever comes next.

The v0 Maintainer briefs a harness on the day's task, reviews the output, and acts on the forge when the work meets the bar.

### Swappable layers

| Layer | Hendeka's adapter |
|---|---|
| Model | configured per harness; `provider:` block |
| Harness | `Harness` Protocol |
| Forge | `Forge` Protocol |
| Sandbox | platform-detected at runtime |
| Tracing | exporter config in manifest |

### Persistent layer

The manifest (stance, rubric weights, schedule, behavior thresholds, KPI endpoints). The catalog (every harness invocation, what it produced, what came of it). The trace history. The discipline (the v0 application's six-phase routine, circuit breaker, companion-work rules, cascade fall-through).

### The v0 application: autonomous Maintainer

**Principle: Hendeka pushes the harness to write better code, then acts on what comes out.** Forge writes belong to Hendeka.

v0 starts with three harnesses — Claude Code, hermes-agent, pi-mono — chosen to span the capability space (opaque-CLI, full-lifecycle-hooks, mid-capability) so the Protocol is forced to handle real variation. Two forges: GitHub and GitLab. One repository per install. Sections §4 onward spec the autonomous Maintainer in detail.

Name: Greek (ἕνδεκα, "eleven") — the eleven magistrates of classical Athens. 11 is important because Hendeka is seeded in [1111](https://philosophers.group/), a philosophy reading group turned philanthropic non-profit.

---

## 0. Open decisions (TODO)

**0.1 Language.** TBD. Hendeka is a substrate; the language must serve substrate-grade requirements, not application-grade preferences. Constraints: small static binaries (no runtime install on the Maintainer's machine); predictable subprocess driving with strict timeout and signal-based cancellation (the substrate orchestrates many heterogeneous subprocesses); direct access to sandbox primitives (bubblewrap, Landlock, seccomp); strict static typing with no public-API escape hatches; no GC pauses in timing-critical paths.

The harnesses themselves can be Python and TypeScript and whatever else — they're applications. The substrate doesn't have that luxury. Candidates: Rust (primary fit on every constraint), Go (secondary; weaker sandbox primitive control), C (purist; most contributor-hostile). Decide before Phase A.1.

Resolving §0.1 unblocks more than Phase A.1. **§13 (tech stack), §14 (codebase conventions and file layout), and §16's translation mapping are intentionally draft** until the language is chosen. The architectural sections (§1–§12, §15) and the operational sections (§17–§24) are language-agnostic.

**0.2 Default harness for the wizard.** Defer to ship-prep.

**0.3 Default model provider per harness.** Defer to ship-prep.

---

## 1. Constraints

### 1.1 Hard

- **1.1.1** Open Source end-to-end. Probably AGPLv3. Zero required paid services. Local-model paths first-class.
- **1.1.2** Commodity hardware. 1 vCPU, 2 GB RAM, 25 GB disk for Hendeka itself.
- **1.1.3 Hendeka owns forge writes.** Every issue close, comment, label, push, PR open is Hendeka calling `Forge`. Harness reasons; Hendeka acts.
- **1.1.4 Forge permissions cached, not gated.** Hendeka caches the App's granted permissions; refreshes each run. Phase 4 reads cache to filter candidates. Cache is **not** a preflight gate — forge enforces; Hendeka catches 403s and re-raises as `ForgePermissionDenied` with install URL.
- **1.1.5 Behavior mode is the gate.** Every `Forge` write checks `behavior_mode` first; raises `BehaviorRestricted` when disallowed. Modes set by §9, not user-configurable.
- **1.1.6 Sandbox for Build.** Linux: bubblewrap + Landlock + seccomp + scoped egress proxy under `hendeka-build` user. macOS: sandbox-exec. Wraps the harness's bash invocation. `--unsafe` is the only bypass; logs WARN.
- **1.1.7 Stable Protocols.** No phase code imports harness/forge-specific types. All harness via `Harness`. All forge via `Forge`.
- **1.1.8 Exactly one artifact per scheduled run.** Phase 4 cascade always produces one. Build abandonment falls through. Daily Note is the bottom; never fails.
- **1.1.9 One repository per install.** Multi-repo out of scope.

### 1.2 Strong defaults

- Schedule: weekdays at 11:11 in Maintainer's timezone. Weekend-enable triggers confirmation; logged in `scope_changes`.
- Webhook replies do not honor schedule.
- Forge permissions configured in GitHub App install UI, not Hendeka.

---

## 2. Success metrics

User: the Maintainer of one repository who delegates maintenance to an autonomous Maintainer-role colleague.

Computed by Phase 0. Tagged with `harness` and `forge`.

**Scheduled** (per run): (1) run completion rate, (2) artifact acceptance rate (rolling 30d), (3) CI pass rate on initial push, (4) clean merge rate, (5) zero-artifact rate (Daily Notes count; not failure), (6) cost per artifact.

**Reply** (per webhook): (7) latency, (8) acceptance, (9) cost.

**Cross-stream**: (10) false-action rate, (11) post-merge revert within 30d.

`hendeka metrics --by-harness` slices by harness.

---

## 3. System overview

```
       Scheduled trigger (11:11 weekdays)
       systemd timer / launchd / cron
                  │
                  ▼
         hendeka run --scheduled
                  │
┌─────────────────┴─────────────────────────────┐
│  Webhook trigger (forge POST)                 │
│                  │                            │
│                  ▼                            │
│         hendeka-webhook receiver              │
│         (always-on; ~20 MB)                   │
│                  │                            │
│                  ▼                            │
│         hendeka respond <event-id>            │
│                                               │
│  Both spawn:                                  │
│  ┌─────────────────────────────────────┐      │
│  │ Configured Harness with:            │      │
│  │   - Hendeka system prompt           │      │
│  │   - Hendeka custom tools            │      │
│  │   - Sandbox wrapper around bash     │      │
│  │   - Trace bridge → SQLite + JSONL   │      │
│  └─────────────────────────────────────┘      │
│                                               │
│  State:   ~/.hendeka/                         │
│           ├── manifest.yaml                   │
│           ├── state.db                        │
│           ├── traces/<run-id>.jsonl           │
│           └── workdir/                        │
└───────────────────────────────────────────────┘
```

Agent invocations spawn → work → exit. Webhook receiver is the only always-on process.

---

## 4. The six phases

"PR" means the forge's change-request concept. `Forge` adapter handles the mapping.

### Phase 0 — Metric Refresh
Compute rolling metrics. Set `behavior_mode` per §9.

### Phase 1 — Cartography
Shallow clone. Read target-repo `AGENTS.md`/`CLAUDE.md`/`.cursor/rules/*` into reference context. Fetch forge state, KPI values. Build manifest delta. Flag drift.

### Phase 2 — Issue Triage
Classify: `spam` / `duplicate` / `needs_info` / `actionable` / `maintainer_handled`. Act per behavior mode.

### Phase 3 — PR Triage
Classify: `ready_to_merge` / `needs_review` / `needs_changes` / `stale` / `abandoned`. Act per behavior mode.

### Phase 4 — Prioritization

Deterministic cascade. First viable type selected. If Phase 5 abandons, re-enter with Build excluded.

1. **PR** — Build candidate AND `Contents: Write` granted AND mode allows.
2. **Manifest Drift Alert** — drift flag fired.
3. **Review Digest** — PR ready-to-merge.
4. **Triage Digest** — untriaged >30 or spam ≥5.
5. **Priority Recommendation** — KPI signal, no matching issue.
6. **Daily Note** — always available.

Exactly one artifact per run.

### Phase 5 — Build (PR path)

1. Branch `hendeka/<issue-number>-<slug>`.
2. Spawn `Harness` with Build prompt, workspace, budget, custom tools. Prompt: not done when surface issue is fixed — done when tests, docs, dead-code are also resolved in this PR.
3. Agent maintains `.hendeka/plan.md` with sections: tried / failed / left / what else this touches.
4. Companion-work checks before completing:
   - **Tests** — new paths get new tests; changed behavior gets updated tests; removed behavior gets removed tests (deleted, not skipped).
   - **Docs** — README, docs, docstrings, type hints, config docs, CLI help. New public surface gets new docs.
   - **Dead code** — removed in same PR.
5. Run `test_command` if configured.
6. **Adversarial self-review**: separate `Harness` session, different model where supported, prompt "find reasons this should not merge." 7-dimension rubric (1–5): solves_issue, respects_avoid_list, matches_stance, tests_cover_change, docs_current, no_dead_code, diff_focused. Weighted sum > threshold (default 25/35). Fail = abandon.
7. Pass: push via `Forge.create_verified_commit` (App) or `GitOps.push` (PAT). Open PR. Label `hendeka-generated`.
8. Fail: comment on source issue with rubric findings. Phase 4 cascade re-enters with Build excluded.

Budget defaults: `max_turns: 50`, `max_usd: 2.00`, `max_wall_clock_seconds: 1200`. Wall-clock kill when harness lacks lifecycle hooks.

### Phase 6 — Notification
Summary to `~/.hendeka/notifications.log`. Comment on tracking issue if configured. Update `stance` from explicit signals only — rejection comments, style comments. Never silent commit amendments. ≤200 chars/field. Reversible. Record run metadata.

---

## 5. Artifacts and replies

### 5.1 Six artifact types

Each includes "Why this one today."

| Type | Trigger | Location |
|---|---|---|
| PR | Phase 5 self-review passed | Branch + PR |
| Manifest Drift Alert | Drift flag | Comment on tracking issue |
| Review Digest | PRs ready-to-merge | Comment on tracking issue |
| Triage Digest | Untriaged ≥30 or spam ≥5 | Comment on tracking issue |
| Priority Recommendation | KPI signal, no matching issue | New draft issue |
| Daily Note | Bottom of cascade | Comment on tracking issue |

Footer: run ID, harness, behavior mode, next run.

PR template and Daily Note template: **[`docs/templates.md`](./templates.md)**.

### 5.2 Replies (webhook)

**Triggers**:
- `issue_comment` on Hendeka PR (author ≠ Hendeka)
- `issue_comment` mentioning `@hendeka` on any issue
- `pull_request_review_comment` / `pull_request_review` on Hendeka PR
- `pull_request closed` on unmerged Hendeka PR (silent stance signal)
- `push` on Hendeka branch by non-Hendeka (silent stance signal)

**Flow**: receiver verifies HMAC → queues → `hendeka respond <event-id>` spawns → behavior-mode gate → resume harness session if supported, else fresh session with replay → agent replies → code changes pushed if requested → trace + cost.

**Rate limits** (orchestration, before harness call): 3/hour per PR, 30/day per repo, 10 mentions/day per user. Comment-storm cooldown: 30 min if >5 comments on one PR within 10 min. Daily USD ceiling: $5 default.

### 5.3 Identity

GitHub App `hendeka[bot]`, per-repo install. Verified commits via Git Data API. PAT fallback uses `Co-authored-by` trailer. DCO-enforcing repos: warning at install; artifacts labeled `dco-unsigned`.

---

## 6. CLI

```
hendeka                       # state-aware status; no triggering
hendeka config                # first run: wizard. subsequent: edit
hendeka config set <k>=<v>
hendeka config show           # --json
hendeka config edit           # $EDITOR; JSON Schema validated
hendeka permissions           # cache + behavior mode + install URL
hendeka permissions --refresh # re-query forge
hendeka today                 # alias for bare hendeka
hendeka status                # last run, next run, mode, harness, forge
hendeka metrics               # --by-harness, --since, --json
hendeka trace <run-id>
hendeka continue <run-id>     # --read-only
hendeka doctor                # health check
hendeka eval --fixture <path> # contributor-facing
hendeka docs <topic>          # offline docs
```

15 commands. No `add`. No manual-run. No `<repo>` arguments.

### 6.1 Wizard

Four mandatory steps:

1. **Harness** — hermes / pi / Claude Code. Default highlighted. Probe install; fail with instructions if missing.
2. **Forge** — GitHub (default; detect App install) or GitLab. Print install URL if missing. Validate via `Forge.list_open_issues()`.
3. **Provider** — menu from `HarnessCapabilities.supported_providers`. Skipped for Claude Code.
4. **Schedule** — timezone detected. Weekday-only default. 11:11 default.

Then advanced (skippable): stance, budget, test_command, acceptance_criteria, kpi_endpoints, observability, per-phase model overrides.

Atomic: nothing written until final confirmation.

Flag-driven: `hendeka config --repo <url> --harness hermes --forge github-app --provider ollama --tz America/Chicago --accept-defaults`.

---

## 7. Manifest

`~/.hendeka/manifest.yaml`. Wizard writes; `config edit` modifies. JSON Schema validated.

Full example: **[`docs/manifest-example.md`](./manifest-example.md)**. Schema: `docs/schemas/manifest.schema.json`.

Top-level keys: `repo`, `harness`, `forge`, `provider`, `models`, `schedule`, `forge_permissions`, `behavior_mode`, `test_command`, `acceptance_criteria`, `rubric_threshold`, `rubric_weights`, `budget`, `stance`, `rate_limits`, `webhook`.

Secrets: env-var names only (`*_env` suffix). Never literal values. CI grep + manifest validator.

---

## 8. Permissions

Forge is the source of truth for what Hendeka *can* do. Behavior mode (§9) is what Hendeka *will* do within that ceiling.

### 8.1 Forge permissions

GitHub App scopes Hendeka uses:

| Scope | For |
|---|---|
| Metadata: Read | always |
| Issues: Read / Write | triage; spam close, comments, labels |
| Pull Requests: Read / Write | triage; abandoned PR close, opening PRs, reviews |
| Contents: Read / Write | clone + reads; branches + commits |
| Webhooks | event delivery |

Read-only install → read-only Hendeka. Cache used for **planning** (Phase 4 filtering) and **reporting** (`hendeka permissions`, wizard gaps). Not a preflight gate.

### 8.2 Behavior modes

| Mode | Effect |
|---|---|
| `normal` | use granted forge permissions |
| `conservative` | rubric threshold 28/35; smaller Build scope; cheaper model |
| `triage_limited` | skip issue-closing for 7 days |
| `scoped` | Build scope shrunk; tests required |
| `pr_blocked` | no PR artifacts |
| `paused` | no writes; read + trace only |
| `replies_quiet` | webhook replies only on `@hendeka` mention |
| `replies_suspended` | webhook replies dropped |

### 8.3 Single gate, in code

```python
async def close_issue(self, n: int, comment: str | None = None) -> None:
    if self._behavior_mode in ("triage_limited", "paused"):
        raise BehaviorRestricted(attempted="close_issue", mode=self._behavior_mode)
    try:
        await self._github_client.close_issue(n, comment)
        self._audit_log.append(("close_issue", n, "success"))
    except GithubException as e:
        if e.status == 403:
            raise ForgePermissionDenied(
                attempted="close_issue",
                forge_message=str(e),
                install_url=self._forge_permissions["install_url"],
            ) from e
        raise
```

### 8.4 Bot-to-bot

Default ignore: `dependabot[bot]`, `renovate[bot]`, `github-actions[bot]`, `codecov[bot]`. Override with `hendeka/review-this` label.

### 8.5 Target-repo AGENTS.md

Read into Build session reference context. Does **not** override sandbox, behavior mode, rate limits, schedule.

---

## 9. Circuit breaker

Two state machines. `src/hendeka/core/circuit_breaker`.

**Scheduled** (window N=10):

| Trigger | Mode |
|---|---|
| Thresholds met | `normal` |
| Acceptance <50% | `conservative` |
| False-action >5% | `triage_limited` |
| Clean-merge <40% | `scoped` |
| CI pass <70% | `pr_blocked` |
| Acceptance <40% × 3 weeks | `paused` |

**Reply** (window N=20):

| Trigger | Effect |
|---|---|
| Reply-acceptance ≥60% (30d) | `normal` |
| Reply-acceptance <40% | `replies_quiet` |
| Daily cost exceeded × 3 | `replies_suspended` |
| Maintainer pause | `replies_suspended` |

Half-open probe: one artifact in restricted mode; observe 24–72h; clean probe steps down; two consecutive return to `normal`.

---

## 10. Harness layer

**Goal: enable any harness.** v0 starts with three, chosen to span the capability space so the Protocol is forced to handle real variation:

| Harness | Lifecycle hooks | Session resume | Cross-session memory | Why it's in v0 |
|---|---|---|---|---|
| Claude Code | No (opaque) | Within session | None | Tests the no-hooks degradation path; widely adopted |
| hermes-agent | Yes | Yes (FTS5) | Built-in + pluggable | Tests the full-capability path; Open Source institutional backing |
| pi-mono | Extension events | Session forking | None | Tests the mid-capability path; TypeScript-native; smallest scope |

If the Protocol works cleanly for all three, it will work for what comes next.

**All harnesses are subprocess invocations.** No harness gets in-process integration, regardless of Hendeka's language. The Protocol abstracts how each harness is started, prompted, and observed. Capability differences are declared in `HarnessCapabilities`; Hendeka's behavior degrades cleanly when capabilities are absent.

Per-harness install + capability detail: `docs/harnesses/<n>.md`.

`Harness` Protocol + `HarnessCapabilities`: §16. Methods: `probe`, `list_supported_providers`, `start_session`, `resume_session`, `prompt`, `terminate`.

**Capability degradation**:
- No lifecycle hooks → wall-clock budget, post-hoc trace parsing, sandbox wraps subprocess.
- No session resume → fresh session with replay.
- No cross-session memory → manifest + SQLite carries state.
- No per-phase model swap → same model across phases.

**Adapters**: `src/hendeka/adapters/harness/{hermes,pi,claude_code}` (one module per harness in the chosen language). Honest capability declaration (false > aspirational). Pass the harness contract suite.

**Provider integration is the harness's job.** Hendeka does not implement providers. Adding a provider = upstream contribution to a harness.

**Adding a harness**: implement the `Harness` Protocol → declare `HarnessCapabilities` honestly → pass the contract test suite → add wizard entry → write `docs/harnesses/<n>.md`. The contract suite is the load-bearing artifact: passing it qualifies a harness for inclusion. PR to Hendeka. v0.x: entry-point discovery so this stops requiring a Hendeka PR.

---

## 11. Forge layer

| Forge | Auth | Verified commits |
|---|---|---|
| GitHub | App, PAT, OAuth | Yes (Git Data API) |
| GitLab | App, PAT | No |

Forgejo: stub. Raises `NotImplementedError`. v0.x or community.

`Forge` Protocol: §16. Every write performs §8.3 single-gate check. Adding a forge: same pattern as harness.

---

## 12. Safety

**Sandbox** (Build): wraps the harness's bash invocation.
- Linux: bubblewrap + Landlock + seccomp + `PR_SET_NO_NEW_PRIVS` + scoped egress proxy + `hendeka-build` user + RO repo bind + writable tmpfs.
- macOS: sandbox-exec.
- Egress allowlist: forge, harness's model endpoint, KPI endpoints, declared package registries.
- `--unsafe` only bypass; logs WARN; surfaces in `today`.

**Cost / abuse**: rate limits + daily ceiling (§5.2) enforced in orchestration before harness call.

**Secrets**: env-var names only. CI grep + manifest validator.

---

## 13. Tech stack

Functional surface: GitHub (App auth + Git Data API for Verified commits) and GitLab forge clients, git operations with auth, HTTP + JSONPath for KPI, embedded SQLite, HMAC-validated webhook receiver, Smee.io relay, OpenTelemetry tracing with optional Langfuse, YAML + JSON Schema, structured logging, CLI, subprocess driver with strict timeout and signal-based cancellation, strict static typing in CI. OS-level: systemd / launchd / cron, bubblewrap (Linux), sandbox-exec (macOS). Harness is installed by the Maintainer separately. All deps under AGPLv3-compatible licenses.

---

## 14. Codebase conventions

Load-bearing across any language: strict typing with no public-API escape hatches; phase code never imports harness/forge-specific types (CI-lint); adapter contract suites (`Forge` against GitHub + GitLab + Forgejo stub, `Harness` against the three v0 harnesses); self-review uses a fresh harness session with an adversarial prompt; Hendeka's own `AGENTS.md` is human-curated and ≤200 lines.

**Architectural split**: source separates the *substrate* from the *application*. Substrate code (foundation layer — harness/forge/git/kpi adapters, sandbox primitives, trace bridge, manifest schema, custom tool registration) does not depend on autonomous-Maintainer code. Application code (the v0 autonomous Maintainer — the six phases, the cascade, the circuit breaker, the rubric) depends on the substrate but the substrate does not depend back. CI-lint enforces the directional dependency. This split makes future applications (review-only, security-audit, dependency-bump) implementable without touching the substrate.

Source organization, file conventions, and lint/typing toolchain follow the chosen language's idioms. Detail will live in `AGENTS.md` once written.

---

## 15. Custom tools registered with the harness

- `get_manifest(section)` — manifest slice
- `get_issue_thread(number)` — forge thread (read)
- `get_kpi(name)` — KPI signal
- `run_acceptance_test(criterion)` — wrapper around `test_command`, structured pass/fail
- `record_plan_section(section, content)` — append to `.hendeka/plan.md`

Registration mechanism varies per harness. Phase code calls uniform `tools.register()`.

---

## 16. Type appendix

`Harness`, `HarnessCapabilities`, `Forge`, `ForgePermissions`, `BehaviorMode`, `RubricScore`, `BuildPhaseInput`, `BuildResult`, `TraceEvent`, exceptions (`ForgePermissionDenied`, `BehaviorRestricted`, `BudgetExceeded`, `RateLimited`, `SandboxUnavailable`, `HarnessUnavailable`, `HarnessCapabilityMissing`, `ConfigInvalid`).

Source: **[`docs/types-appendix.md`](./types-appendix.md)** — Python syntax for illustration; semantically language-agnostic. Translates to the chosen language (TODO §0.1) using that language's idiomatic forms for sum types, structs, traits/interfaces, and branded primitives.

This file is the contract. Every Hendeka task references it.

---

## 17. Webhook receiver

Always-on. ~20 MB. Single-process. Validates HMAC, queues to SQLite, spawns `hendeka respond <event-id>`.

Modes: `public` (VPS + TLS), `smee` (NAT; uses Smee.io). Same interface.

Restartable; queued events resume (subprocess invocation idempotent).

Implementation: `docs/webhooks.md`.

---

## 18. Observability

Three layers, always on:
- JSONL: `~/.hendeka/traces/<run-id>.jsonl`
- SQLite: `trace_events` indexed by run_id, harness_id, phase
- OTel spans: every phase, harness invocation, tool call, forge request

Exporters: `none` (default), `langfuse`, `otlp`. Trace tagged with `harness_id` + `forge_id`.

---

## 19. Deployment

No long-running agent. Webhook receiver is the only always-on process. Scheduled and reply spawn → work → exit.

Schedulers: systemd user timer + service (Linux), launchd plists (macOS), cron fallback. Webhook: systemd service, launchd plist, or `hendeka webhook start`.

Platforms: Linux (Debian 12+, Ubuntu 22.04+, Fedora 40+, Arch), macOS 14+, Raspberry Pi 4/5 with 4 GB+. Not Windows native.

Implementation: `docs/deployment.md`.

---

## 20. Documentation pages

`docs/`: `quickstart.md`, `install.md`, `config.md`, `permissions.md`, `webhooks.md`, `replies.md`, `interactive.md`, `lifecycle.md`, `metrics.md`, `artifacts.md`, `troubleshooting.md`, `architecture.md`, `harnesses/<n>.md`, `forges/<n>.md`, `contributing.md`. Plus `types-appendix.md`, `templates.md`, `manifest-example.md`.

---

## 21. Build order

### Phase A — skeleton

1. Repo setup per §14 (chosen language toolchain).
2. `AGENTS.md` ≤200 lines. `CLAUDE.md` → `AGENTS.md`.
3. `src/hendeka/core/types` (extension per chosen language) translated from `docs/types-appendix.md`.
4. Contract test framework: `Harness`, `Forge`, `KPIAdapter`, `GitOps`.
5. Domain Protocol files only.
6. Forgejo stub passing contracts.
7. SQLite schema: `runs`, `traces`, `trace_events`, `metrics_runs`, `metrics_rolling`, `artifacts`, `scope_changes`, `webhook_events`, `rate_limit_events`, `audit_log`. All include `harness_id` + `forge_id`.
8. OTel + Langfuse wiring.
9. `GitOps` impl.
10. Register Hendeka GitHub App; ID + private key in env.
11. `Forge` GitHub (App + PAT). `create_verified_commit` via Git Data API. Behavior-mode check; 403 → `ForgePermissionDenied`.
12. `Forge` GitLab.
13. `hendeka config` wizard (4-step + advanced). Atomic write. Validates harness reachability and forge auth.
14. Bare `hendeka` state-aware routing (no triggering).
15. `config` subcommands: `set`, `show`, `edit`. JSON Schema validated.
16. Config loading resolves env-vars; never logs values.
17. `hendeka permissions` CLI: cache + behavior mode + install URL. `--refresh` re-queries.
18. `ForgePermissions` cache populated at wizard; refreshed when >24h stale.
19. `hendeka doctor`.

### Phase B — harness adapters

20. `Harness` Protocol + `HarnessCapabilities` + contract suite.
21. hermes-agent adapter.
22. pi-mono adapter.
23. Claude Code adapter.
24. Custom tool registration mechanics per harness.

### Phase C — scheduled pipeline

25. Phase 1 Cartography.
26. Manifest diff + drift flags.
27. Phases 2 + 3 with behavior-mode enforcement.
28. Phase 4 cascade (rule-based; reads `forge_permissions`).
29. Phase 5 Build via `Harness`.
30. Adversarial self-review (separate session, different model).
31. Plan file `.hendeka/plan.md`.
32. Companion-work integration.

### Phase D — webhook

33. Webhook receiver implementation (HTTP framework per §0.1). HMAC validation. Queued events.
34. Smee.io client.
35. `hendeka respond <event-id>`. Session resume per harness capability.
36. Rate limits in orchestration.

### Phase E — metrics, breaker, observability

37. Phase 0 metric refresh.
38. Scheduled circuit breaker (every transition tested).
39. Reply circuit breaker.
40. Mode surfacing in footer / `today` / `metrics`.
41. Non-PR artifacts (including Daily Note).
42. Trace writer end-to-end.
43. CLI: `today`, `status`, `metrics --by-harness`, `trace`.
44. `continue <run-id>` (degrades when harness lacks resume).
45. Phase 6 stance update.

### Phase F — ship

46. Eval harness `hendeka eval --fixture`.
47. Sandbox hardening; `hendeka-build` user; denied-connection test.
48. Scheduler installers. Webhook installer. 14-day clock-advance test.
49. Docs per §20.
50. `hendeka[bot]` App publication.
51. End-to-end: Hendeka opens a real PR on its own repo passing contracts + CI.

---

## 22. Definition of Done

One issue per item. One DoD per Claude Code task.

1. `hendeka config` wizard works on Debian 12, macOS 14, Raspberry Pi OS.
2. Quickstart <10 min median (3 unfamiliar Maintainers, hermes+Ollama).
3. Wizard 4-step + advanced. Atomic. Validation before accept.
4. `config` {`set`, `show`, `edit`} functional; edit validates on save.
5. First scheduled run fires at 11:11 weekdays; produces an artifact.
6. Default schedule weekdays-only; weekend-enable confirmation; logged.
7. 14-day clock-advance: zero weekend runs default; weekends fire when configured.
8. Bare `hendeka` and `today` produce status without triggering.
9. Artifact footer: run ID, harness, mode, next run.
10. PR template includes "Why this one today," rubric summary, plan reference, "What else this touched," "Working with me."
11. PR signed "Hendeka, autonomous Maintainer"; `hendeka[bot]` identity; Verified via Git Data API.
12. DCO warning at install for DCO-enforcing repos.
13. Forge perms cached at wizard; `permissions` shows cache + URL; `--refresh` re-queries; no grant/revoke.
14. Single-gate enforcement: behavior-mode check in every `Forge` write; 403 caught and re-raised. No preflight.
15. Metrics 1–10 compute and surface. Metric 11 starts after first merge. All tagged with `harness_id` + `forge_id`.
16. `metrics --by-harness` produces per-harness slice.
17. Sandboxed Build does not network outside allowlist (denied-connection test).
18. No secrets in config (CI grep + validator).
19. Linux sandbox: bubblewrap + Landlock + seccomp + egress proxy.
20. macOS sandbox: sandbox-exec.
21. Build runs as `hendeka-build` user.
22. `--unsafe` is the only bypass; WARN; surfaces in `today`.
23. Contract suites: `Forge` passes GitHub live + GitLab live + Forgejo stub. `Harness` passes hermes + pi + Claude Code.
24. Phase code never imports harness/forge-specific types (CI lint).
25. Strict typing in CI; lint passes.
26. `eval --fixture` runs across harness/forge combinations.
27. OTel spans for every phase, harness invocation, tool call, forge request. Default `none`. Langfuse + OTLP tested.
28. Scheduled breaker: every transition + half-open probe unit-tested. Reply breaker: `replies_quiet` + `replies_suspended` unit-tested.
29. Behavior mode visible in footer, `today`, `status`, `metrics`.
30. CLI errors include docs links. All §20 docs exist.
31. Adversarial self-review: separate session, 7-dimension rubric, weighted threshold (default 25/35). Test: 24.9 fails, 25.0 passes.
32. Companion-work fixture: stale docstring + unused import + skip-if-over-removed-feature. Build cleans all three. `docs_current`, `no_dead_code`, `tests_cover_change` ≥4.
33. Plan file `.hendeka/plan.md` with "what else this touches"; committed; in PR description.
34. Webhook receiver: starts via systemd/launchd; HMAC validated; queued; subprocess non-blocking; Smee mode works.
35. Reply flow: events → `respond`; harness session resumed where supported; reply in same thread; behavior mode + rate limits enforced.
36. Rate limits: 3/hr per PR, 30/day per repo, 10/day per user; storm cooldown; daily $5.
37. Reply gating: `replies_quiet` (mention-only), `replies_suspended` (none).
38. Phase 6 stance from explicit signals only; ≤200 chars/field; reversible.
39. Hendeka's AGENTS.md ≤200 lines, human-curated; CI rejects bulk-regen.
40. Capability degradation: wall-clock budget when no lifecycle hooks; fresh-session-with-replay when no resume; manifest-state when no cross-session memory.
41. `doctor` reports pass/fail across all subsystems.
42. Adding a `Harness` or `Forge` requires only passing the contract test suite — no Hendeka core changes. Verified by implementing GitLab fully against contracts starting from the stub, and by adding a fourth harness as a community-style PR exercise that touches only the new adapter file + a wizard entry + a docs page.
43. Phase 4 cascade always produces an artifact. Test: a run with every higher-priority type rejected produces a Daily Note. Build abandonment falls through. No zero-artifact runs.
44. Daily Note contains "today I considered" cascade trace, repo state, behavior mode, daily cost usage, stance update.
45. Substrate/application split: substrate code (adapters, sandbox, trace bridge, manifest schema, tool registration) has zero imports of autonomous-Maintainer code. CI-lint enforces. Verified by introducing a stub second application (e.g., a no-op "review-only" mode) that depends on the substrate without modifying any substrate code.

---

## 23. Out of scope (v0)

Multi-repo. Plugin discovery (entry points). Provider abstraction in Hendeka. Forgejo full impl. Custom `ModelProvider` / `BuildEngine`. Local-model deployment. Multi-repo cross-reasoning. Web UI. External notification channels. Windows native. IDE integration. Self-merge. Phone-home telemetry. MCP inside Hendeka. Voice. Mobile. Paid gateway services as default. Dataset-level self-evolution.

---

## 24. Handoff to Claude Code

The PRD is the contract. Don't paste it whole into one task.

1. **Seed**: commit `docs/prd.md`, `docs/types-appendix.md`, `docs/templates.md`, `docs/manifest-example.md`. Hand-write `AGENTS.md` ≤200 lines using §16 names + harness/forge Protocol shapes.
2. **Resolve TODO §0.1** before Phase A.1.
3. **Split along §21 phases**. One session per phase.
4. **Use §22 DoD as task surface**. One issue per item; hand one at a time.
5. **Every task references §16, the relevant phase section, AGENTS.md.** Not the full PRD.

`AGENTS.md` is the permanent context. Keep it ≤200 lines.

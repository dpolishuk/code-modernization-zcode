# Installing and working with Code Modernization for ZCode

A complete guide: install the plugin, run your first modernization end to end, and know
what to expect at every step — including where **you** decide and where scripts decide.

## 1. Requirements

- **ZCode Desktop** with dynamic workflows (the `CreateWorkflow` tool and the
  `dynamic-workflows` skill). Everything else degrades gracefully without it (see §8).
- **python3** (3.8+) — runs the map/extract-rules/verify/report scripts. Without it the
  heavy steps still work but the HTML report and the proof pack cannot run.
- Optional but recommended: **`scc`** (`brew install scc`) or `cloc` for lines/complexity
  metrics, **`git`**, and **`lizard`** only if you plan to use `--portfolio` mode.
- A build toolchain for the *target* stack of whatever you modernize (e.g. a JDK 17 if
  you're moving to Java 17). The `preflight` command tells you exactly what's missing.

## 2. Install

1. Clone this repo anywhere you like:

   ```bash
   git clone https://github.com/dpolishuk/code-modernization-zcode.git
   ```

2. In ZCode open **Plugin Marketplace → Add → Add Plugin Marketplace**, paste the
   clone's `plugins/` directory, and add it. The market shows as **dev-code-modernization**.

3. Go to **Personal → dev-code-modernization → Code Modernization → Install**.

4. Managed afterward under **Settings → Plugins** (enable/disable, remove).

## 3. Verify the installation

In a new ZCode chat, type `/code-modernization` — autocomplete should offer the 13
commands (`modernize`, `modernize-preflight`, `modernize-assess`, `modernize-map`,
`modernize-extract-rules`, `modernize-review`, `modernize-brief`, `modernize-uplift`,
`modernize-transform`, `modernize-reimagine`, `modernize-verify`, `modernize-harden`,
`modernize-status`).

If a command's shell ever prints a literal `${CLAUDE_PLUGIN_ROOT}` path (the token
arrived unsubstituted), the install root is
`~/.zcode/cli/plugins/cache/dev-code-modernization/code-modernization/<version>/` —
the commands' preambles tell the agent to fall back to it automatically, but you can
paste it yourself if a step stalls.

## 4. Update / uninstall

- **Update**: `git pull` in the clone, then refresh the market in the Plugin
  Marketplace UI and update the plugin from **Personal → dev-code-modernization**.
- **Uninstall**: **Settings → Plugins → Code Modernization → Remove**. Your work
  artifacts (`analysis/`, `modernized/`) are plain files in the workspace and stay.

## 5. The working model

The plugin works inside one workspace (any folder you open in ZCode) per set of systems:

```
your-workspace/
  legacy/<name>        the code under modernization — a symlink to where it really
                       lives (or a copy). NEVER edited by the plugin.
  analysis/<name>/     everything the plugin learns and decides:
                       INTENT.md, PREFLIGHT.md, ASSESSMENT.md, topology.json,
                       TOPOLOGY.html, BUSINESS_RULES.md, MODERNIZATION_BRIEF.md,
                       VERIFICATION.md, SECURITY_FINDINGS.md, REPORT.html …
  modernized/          everything it builds (rewrites, uplifted copies, reimagines)
```

Two principles hold throughout:

- **Scripts decide, agents propose.** Verdicts (PROVEN / NOT PROVEN, same/different
  outputs) are computed by `scripts/proof_pack.py` and friends from files — never by
  the model's opinion.
- **Legacy code is untrusted data.** Every agent prompt fences mined text and forbids
  acting on instruction-shaped content in source; credentials are always masked to a
  2–4 character preview.

## 6. Your first modernization (the happy path)

Work through these in order; every command ends by naming the exact next one.

### Step 0 — Say what you want

```
/code-modernization:modernize billing --source /work/systems/billing
```

Two short pop-ups at most (what do you want: a version move / a rewrite / a rebuild /
understand it first — and what must stay true). It symlinks the code at `legacy/billing`,
writes `analysis/billing/INTENT.md` verbatim from your answers, and prints your personal
road map. Nothing has been analyzed yet.

### Step 1 — Preflight (5 minutes, one time)

```
/code-modernization:modernize-preflight billing "Java 17"
```

Asks you five questions only a person who knows the system can answer (scope, local
buildability, bespoke build machinery, prior attempts, off-limits areas) — "don't know"
is a valid answer — while it checks tooling, proves the build actually compiles one file
and one small project, verifies the tree is complete, finds the **scope boundary**
(is `billing` really the whole system or a slice of a monorepo?), and checks the source
is protected from edits (it suggests `chmod -R a-w legacy/billing` if nothing enforces
that). Output: `PREFLIGHT.md` with a Ready / Ready-with-gaps / Not-ready verdict per
command. **Fix anything red before proceeding.**

### Step 2 — Assess

```
/code-modernization:modernize-assess billing
```

Three specialists in parallel (structure, technical debt, security). You get
`ASSESSMENT.md` (inventory, complexity, debt top-10, CWE-tagged findings, doc gaps, and
a recommended pattern: Rehost / Replatform / Refactor / Rearchitect / Rebuild / Replace),
`ARCHITECTURE.mmd`, and the first `REPORT.html`. Managing many systems? Use
`/code-modernization:modernize-assess --portfolio /work/systems` for a ranking heat-map
of every subdirectory.

### Step 3 — Map

```
/code-modernization:modernize-map billing
```

Produces `topology.json` and the interactive `TOPOLOGY.html`: a zoomable map of domains,
modules sized by LOC, call/data edges, entry points, dead-end candidates, and 2–4
end-to-end **persona flows** (a claimant files a weekly claim — which nodes make that
happen). Open it, search a module, click it, play a flow. This map drives sharding in
the next step.

### Step 4 — Extract the business rules (the big one)

```
/code-modernization:modernize-extract-rules billing
```

A sharded, multi-agent workflow (see §8): one extractor per module, every rule's
`file:line` citation re-checked by a referee agent, duplicates consolidated, every P0
rule double-judged by a compliance and a fidelity judge. What lands is
`BUSINESS_RULES.md` (Given/When/Then rule cards, priority, confidence, suspected
defects, SME questions) and `DATA_OBJECTS.md`. Rules the referees refuted are counted,
not hidden — that number is the quality the verification bought.

### Step 5 — Review the flagged rules (you decide)

```
/code-modernization:modernize-review billing
```

Only the rules needing judgement (P0 with a suspected defect, an SME question, or below
High confidence) come up as pop-ups: **Right / Wrong / Not sure**, plus a free-text note.
Your verdicts go to `RULE_REVIEWS.json` and steer everything downstream — a rule marked
*wrong* is never used as the oracle for the rewrite.

### Step 6 — The brief (you approve before any build)

```
/code-modernization:modernize-brief billing "Java 17"
```

Synthesizes everything into `MODERNIZATION_BRIEF.md`: target architecture, 3–6 phases
(each with the exact build command, modules, scale, risks, checkable entry/exit
criteria), business walkthroughs for non-technical approvers, the **Behavior Contract**
(the P0 rules that must be proven equivalent), validation strategy, and open questions.
**Nothing is built until you approve** — and you steer by *editing this file*: an edited
criterion is binding, a chat message is not.

### Step 7 — Build (one phase at a time)

| Track | Command | What happens |
|---|---|---|
| Same stack, newer version (.NET Fx→8, Java 8→17) | `/code-modernization:modernize-uplift billing "Java 8" "Java 17"` | Delta catalog → dual-run harness → baseline → **pilot ONE unit → you approve the playbook** → batched fan-out with a circuit breaker |
| Rewrite in a different stack, piece by piece | `/code-modernization:modernize-transform billing <module> "<target>"` | Characterization tests FIRST → idiomatic rewrite → canary + equivalence diff → architecture critique |
| Greenfield rebuild from extracted intent | `/code-modernization:modernize-reimagine billing "<vision>"` | Spec mining → checkpoint 1 → architecture + critic → checkpoint 2 → parallel service scaffolding |

Each track has its own plan gate: read the plan it presents and approve before code.

### Step 8 — Verify (independent proof)

```
/code-modernization:modernize-verify billing          # all built modules
/code-modernization:modernize-verify billing pricing  # just one
```

Runs in a fresh session if possible. It re-runs every suite from clean, keeps JUnit
XML / raw logs, breaks a copy of the code on purpose (**canary** — the tests must go
red or they pin nothing), invents 10+ adversarial inputs nobody used in development,
compares outputs byte-for-byte, then `proof_pack.py` computes one verdict per module:
**PROVEN / PARTLY PROVEN / NOT PROVEN**, with reasons. A person signs the sign-off block
in `VERIFICATION.md`; that signature is approval, PROVEN is only evidence.

### Step 9 — Harden

```
/code-modernization:modernize-harden billing
```

Five class-scoped finders, every finding adversarially refuted, Critical/High
double-judged. You get `SECURITY_FINDINGS.md` plus a reviewable patch
(`security_remediation.patch`, with credential hunks split into a gitignored
`.local.patch`). You review and apply it yourself — the plugin never edits `legacy/`.

### Anytime — Where am I?

```
/code-modernization:modernize-status billing
```

One screen: what exists, what is stale relative to its inputs, and the single most
useful next command.

## 7. Choosing a track (when you're not sure)

Run `preflight` + `assess` and read the **Recommended Modernization Pattern** — the
assessment routes you. Quick heuristics:

- Code is good, only the runtime is old → **uplift**. (If the delta catalog later shows
  most of the tree must change, the workflow says so and you switch to transform.)
- Stack is dying or the team is moving languages → **transform** (strangler fig, the
  old system keeps running).
- Architecture itself is the problem → **reimagine**, using the mined rules as the spec.
- Just want understanding / documentation / risk numbers → stop after `brief`; the
  assessment, map and rules are deliverables in their own right.

## 8. The workflow steps: what to expect from CreateWorkflow

The heavy fan-outs (extract-rules, harden-scan, uplift deltas/migrate, portfolio,
reimagine scaffolding) run as **dynamic workflows**:

- The command loads the `dynamic-workflows` skill and calls `CreateWorkflow` with the
  plugin's script under `workflows/`. You'll see a **confirmation dialog** with the
  script's phases before it starts — that's your chance to check the plan.
- Calibration for `extract-rules`: ~10–15 agents per shard, ~8 agents finishing per
  minute; 4 shards (1,600 lines) ≈ 58 agents / 10 minutes; a 44-program estate ≈
  466–647 agents / 50–80 minutes. **With more than 12 shards it asks before launching.**
- If a run is interrupted, it is **resumable** — completed agents replay from the
  journal (`ResumeWorkflowRun`); never re-run from scratch.
- Coverage is honest: dead finders, unverified rules and skipped shards come back as
  named gaps and re-passable lists (`rerunModules`, `deadFinders`, `unverified`), not
  as silence.
- **No dynamic workflows available?** Every command has a fallback path using plain
  subagents (the plugin's own agents: legacy-analyst, business-rules-extractor,
  security-auditor, test-engineer, version-delta-analyst, uplift-migrator, scaffolder,
  architecture-critic). It's less parallel, equally honest.

## 9. Where you decide (the six gates)

| # | Gate | What you do |
|---|---|---|
| 1 | Preflight answers | Answer 5 questions (or "don't know") |
| 2 | Rule review | Confirm/correct flagged business rules |
| 3 | Brief approval | Approve the plan; edit the file to steer it |
| 4 | Plan gates in build steps | Approve each track's plan before code |
| 5 | Proof acceptance & sign-off | Accept or reject each found difference; sign `VERIFICATION.md` |
| 6 | Security patch | Review and apply `security_remediation.patch` yourself |

## 10. Day-2 operations

- **Stale artifacts**: `status` flags anything older than what it derives from (e.g. a
  brief older than new rules) — rerun the generating command.
- **Several systems** in one workspace: every command takes the system name; artifacts
  live per-system under `analysis/<name>/`.
- **Secrets**: findings with credentials go to gitignored `SECRETS.local.md` /
  `*.local.patch` only — check `analysis/.gitignore` exists before committing anything.
- **Long runs**: prefer fresh sessions for `verify` (independence), keep the run IDs of
  workflows for resume.

## 11. Troubleshooting

| Symptom | Fix |
|---|---|
| Command not in autocomplete | Plugin disabled or market stale — refresh in Plugin Marketplace, check **Settings → Plugins** |
| `python3: command not found` in report refresh | Install Python 3.8+; that step is a convenience and degrades to a one-line notice |
| Literal `${CLAUDE_PLUGIN_ROOT}` in output | Use `~/.zcode/cli/plugins/cache/dev-code-modernization/code-modernization/<version>/` as the plugin root (commands do this automatically when they can) |
| `CreateWorkflow` not offered | Dynamic workflows unavailable in your build — commands fall back to plain subagents automatically |
| Workflow stopped mid-run | Resume it (`ResumeWorkflowRun`); if the *script* was wrong, amend it (`AmendWorkflow`) — finished agents are cached |
| Tests can't run locally (no legacy runtime) | Expected for mainframe-style stacks: equivalence falls back to recorded traces; the verdict then cannot be PROVEN and says so |
| Build fails in uplift fan-out | The circuit breaker stopped it on purpose — fold `playbookGaps` into `PLAYBOOK.md`, re-verify one failed unit, re-invoke with the failed/blocked/remaining lists |

## 12. Safety and privacy

- The plugin never writes under `legacy/`; it verifies this from file times.
- Source is untrusted: prompts fence mined content, and planted instructions are
  *reported* (`injectionFlags`), never followed.
- Credentials are masked everywhere shareable; raw values only ever appear in gitignored
  `.local` files, and only with `--show-secrets`.
- No telemetry of any kind ships in this port (upstream's usage-count hooks were removed).

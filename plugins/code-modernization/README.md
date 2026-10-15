# Code Modernization (ZCode port)

Guided modernization for any legacy codebase: start with `/code-modernization:modernize`, get an
assessment, an interactive map, the business rules mined from the code, and a plan you approve —
then an upgrade, a rewrite or a rebuild, with independent proof that the new code behaves like the
old. Includes security hardening and a one-page report.

This is a port of Anthropic's
[`code-modernization`](https://github.com/anthropics/claude-plugins-official/tree/main/plugins/code-modernization)
Claude Code plugin (Apache 2.0, see `LICENSE`) to the ZCode plugin format. Upstream is v1.0.0;
this port starts at 0.1.0.

## The pipeline

Each step is a standalone command; run them in order. Output lands in `analysis/<system>/` and
`modernized/`, with a refreshed `analysis/<system>/REPORT.html`.

| # | Command | What you get |
|---|---|---|
| 0 | `/code-modernization:modernize` | Front door: records what you want in `INTENT.md` |
| 1 | `/code-modernization:modernize-preflight` | Environment/scope checks, build smoke test |
| 2 | `/code-modernization:modernize-assess` | Inventory, complexity, debt, security (`--portfolio` ranks many systems) |
| 3 | `/code-modernization:modernize-map` | Dependencies, data flow, entry points (interactive `TOPOLOGY.html`) |
| 4 | `/code-modernization:modernize-extract-rules` | Business rules as Given/When/Then cards with file:line citations, cross-checked |
| 4b | `/code-modernization:modernize-review` | You confirm or correct the flagged rules |
| 5 | `/code-modernization:modernize-brief` | The phased plan — approval required before any build |
| 6 | `/code-modernization:modernize-uplift` / `-transform` / `-reimagine` | The build (same-stack upgrade / cross-stack rewrite / greenfield rebuild) |
| 7 | `/code-modernization:modernize-verify` | Independent proof: one verdict per module (PROVEN / PARTLY PROVEN / NOT PROVEN) |
| 8 | `/code-modernization:modernize-harden` | Security scan of the legacy system with a reviewed patch |
| — | `/code-modernization:modernize-status` | Where you are, what is stale, the exact next command |

A person decides six things: the five preflight answers, the rule review, brief approval, accepting
each difference the proof finds, signing the proof, and applying the security patch.

## How proof works

Verdicts come from `scripts/proof_pack.py`, not model opinion: clean test re-runs from result files,
byte-level old-vs-new output comparison (with masked variable fields), adversarial fresh inputs, a
canary (a deliberate break must turn tests red), confirmation that each P0 rule has a test that
actually ran, and source-untouched checks via file times. Legacy code is treated as untrusted data
throughout and is never edited.

## What differs from the Claude Code original

- **No telemetry.** Upstream's usage-count hooks are removed entirely (they pipe through Claude
  Code's own telemetry). This plugin ships no hooks.
- **No live progress pane / review deck / sign-off dialog.** Those are Claude Code function-hook UI
  modules with no ZCode equivalent. The artifacts they surfaced are all still produced as files:
  `REPORT.html`, `TOPOLOGY.html`, `BUSINESS_RULES.md` (rule review happens through the `review`
  command's pop-ups), and `VERIFICATION.md`'s sign-off block.
- **No `userConfig`.** ZCode manifests do not carry plugin settings. The defaults apply (`legacy/`
  as the read-only source tree, track auto-detection).
- **Workflows** (`workflows/*.ts`) are ported to ZCode dynamic workflows. The commands load the
  `dynamic-workflows` skill and call `CreateWorkflow` with the script path and a `pluginRoot` arg;
  each workflow subagent reads its specialist definition (e.g. `agents/legacy-analyst.md`) from the
  plugin, with a fallback persona if that is unavailable. Stopped runs resume via `ResumeWorkflowRun`.
- **Preflight Check 7** (is the source protected from edits?) anchors on the filesystem guarantee
  (`chmod -R a-w` or a read-only mount) plus any agent-permission rules the host happens to declare,
  instead of Claude Code's `permissions.deny` settings.
- **Commands** normalize on `$ARGUMENTS` parsing (ZCode does not substitute positional argument
  declarations), and `${CLAUDE_PLUGIN_ROOT}` resolution has a stated fallback to the plugin cache
  directory.

## Install (local dev marketplace)

1. Clone this repository.
2. In ZCode: **Plugin Marketplace → Add → Add Plugin Marketplace**, paste
   the clone's `plugins/` directory (the one holding `marketplace.json`).
3. **Personal → dev-code-modernization → Code Modernization → Install**.
4. Try it on something small:
   `/code-modernization:modernize mysystem --source /path/to/legacy/code`

A complete walkthrough (every command, every artifact, every human gate, troubleshooting)
is in the repository's [docs/usage.md](../../docs/usage.md).

## Development notes

- `scripts/` is upstream Python, verbatim except the removed `telemetry.py`; its pytest suite lives
  in `scripts/tests/` (needs `pytest`, not bundled).
- `workflows/*.ts` typecheck against the dynamic-workflows facade under `--strict`; each declares
  its arguments in a `/* zcode-workflow */` block and narrows them at runtime as well.

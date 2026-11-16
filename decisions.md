# ironspec — decisions and research log

Research date: 2026-11-16. Every fact below was fetched from the cited source on that date.
Anything not confirmed from a primary source is in [Unverified](#unverified--check-at-the-named-milestone)
and is not relied on by `SPEC.md`.

## Q&A log (question tool)

| # | Question | Recommended option | Answer |
|---|---|---|---|
| Q1.1 | CLI framework, given Typer now needs compiled pydantic-core | Cyclopts | Cyclopts |
| Q1.2 | How to implement DRY / diff-coverage / secret gates without Node, Go or compiled deps | Built-in + adapters | Built-in + adapters |
| Q1.3 | Product / CLI name | Keep `<FW>` for now | **ironspec** |
| Q1.4 | Delivery of SPEC and milestones given the single-branch session | SPEC on this branch + PR | SPEC on this branch + PR |
| Q2.1 | CLI command and state directory name | `ironspec` + `.ironspec/` | `ironspec` + `.ironspec/` |
| Q2.2 | Scope of the v1 benchmark (AC-12) | Harness + scripted runner | Harness + scripted runner |

Open questions: none.

## ADRs

### ADR-001 — Python floor is 3.11 (build prompt said ≥ 3.10)
- 3.10 reached end of life on 2026-10-01. Oldest supported is 3.11; newest stable is 3.15.
- Decision: `requires-python = ">=3.11"`, test matrix 3.11–3.15.
- Source: https://devguide.python.org/versions/

### ADR-002 — Cyclopts, not Typer (changes a locked decision; user confirmed in Q1.1)
- Typer 0.29.0 made pydantic a required dependency; pydantic pins pydantic-core, which ships
  only platform wheels. That violates "no compiled dependencies". `typer-slim` is discontinued.
- Cyclopts 5.5.1 needs Python 3.11+ and its whole tree has `py3-none-any` wheels
  (attrs, docstring-parser, rich-rst, rich, pygments, markdown-it-py, mdurl).
- Sources: https://typer.tiangolo.com/release-notes/ ,
  https://pypi.org/pypi/typer/json , https://pypi.org/pypi/cyclopts/json ,
  https://cyclopts.readthedocs.io/en/latest/vs_typer/README.html

### ADR-003 — Runtime dependency set is pure-Python only
- Rejected as runtime deps (compiled somewhere in the tree): PyYAML (platform wheels only),
  jsonschema (requires rpds-py), diff-cover (jinja2 → MarkupSafe), pydantic (pydantic-core),
  detect-secrets (requires PyYAML).
- Chosen: `cyclopts`, `ruamel.yaml` (pure wheel; `ruamel.yaml.clib` not required),
  `fastjsonschema` (no deps), `python-ulid` (pure). Standard library for HTTP (`urllib.request`),
  git (`subprocess`), hashing, TOML reading.
- A release CI job fails if any resolved runtime wheel is not `py3-none-any`.
- Source: PyPI JSON API, `https://pypi.org/pypi/<name>/json` for each package named above.

### ADR-004 — Gates ship built-in implementations, with optional adapters (user confirmed in Q1.2)
- jscpd 5.4.3 needs Node 18+. diff-cover has a compiled transitive dep. gitleaks is not on PyPI.
  No pure-Python multi-language clone detector was found.
- Decision: built-in standard-library implementations are the default for the DRY, diff-coverage
  and secret gates. Adapters for jscpd, diff-cover and gitleaks are opt-in per repo.
- diff-cover reads Cobertura, Clover, JaCoCo and LCov; the built-in gate reads Cobertura XML and LCov.
- Sources: https://github.com/kucherenko/jscpd , https://github.com/Bachmann1234/diff_cover ,
  https://github.com/Yelp/detect-secrets

### ADR-005 — Positioning against AI-DLC is corrected
- The build prompt says AI-DLC's weakness is prompt-enforced rules. That is no longer true.
  AI-DLC 2.11.4 is a native `aidlc` binary with a "deterministic engine", harness hooks,
  phase verification gates and tool-owned verification receipts. It has 5 phases and 33 stages,
  ends each stage with Approve / Request Changes, and documents no headless mode
  (`AIDLC_SKIP_HUMAN_PRESENCE_GUARD=1` lowers its human-presence guard). License MIT-0.
- Decision: "Prompts guide. Code enforces." stays as ironspec's principle, but it is not claimed
  as a differentiator against AI-DLC. The differentiators stated in `SPEC.md` are: no binary,
  one human approval, TDD / AC-trace / diff-coverage gates, and a conflict-free git-native event log.
- Adopted ideas: gate results are computed by the tool, never supplied by the agent; a
  human-presence guard on approval commands with an explicit, recorded override.
- Sources: https://github.com/awslabs/aidlc-workflows ,
  https://github.com/awslabs/aidlc-workflows/releases ,
  https://awslabs.github.io/aidlc-workflows/guide/04-phases-and-stages/

### ADR-006 — superpowers: what to take
- v7.0.2, MIT. Skills are `SKILL.md` with `name` and a "Use when…" `description`. Enforcement is
  prompts plus one SessionStart hook (matcher `startup|clear|compact`) that injects a bootstrap.
  Its TDD rule is prompt-only.
- Decision: re-inject the ironspec bootstrap on startup, clear and compact; write skill
  descriptions as "Use when…"; back the TDD rule with gates.
- Sources: https://github.com/obra/superpowers , https://github.com/obra/superpowers/releases ,
  https://raw.githubusercontent.com/obra/superpowers/main/hooks/hooks.json

### ADR-007 — Beads is cited as inspiration only
- Beads is Go, MIT, now at `gastownhall/beads`, and Dolt is its only storage backend.
  `issues.jsonl` is an export, "not the source of truth or a backup". v0.57.0 removed the JSONL
  storage layer and the git merge driver.
- Decision: docs must not describe Beads as JSONL-in-git. Ideas reused natively: hash-based IDs,
  a ready queue, atomic claims.
- Sources: https://beads.gascity.com/architecture ,
  https://newreleases.io/project/github/steveyegge/beads/release/v0.57.0 ,
  https://github.com/gastownhall/beads/releases

### ADR-008 — Instruction files
- Claude Code reads `AGENTS.md` natively, but only when no `CLAUDE.md` exists; `@AGENTS.md` in
  `CLAUDE.md` is supported and never causes a double read.
- Copilot CLI and cloud agent read repo-wide instructions, `AGENTS.md` and `CLAUDE.md`.
  VS Code reads `.github/copilot-instructions.md` or `AGENTS.md`.
- AGENTS.md is free-form Markdown; the nearest file to the edited path wins.
- Decision: as in the build prompt — `AGENTS.md` canonical, `CLAUDE.md` = `@AGENTS.md` plus
  Claude-only notes, `.github/copilot-instructions.md` a short pointer.
- Sources: https://code.claude.com/docs/en/memory ,
  https://docs.github.com/en/copilot/reference/custom-instructions-support ,
  https://code.visualstudio.com/docs/agents/reference/custom-instructions-reference , https://agents.md/

### ADR-009 — Hook formats and support tiers
- Claude Code (`.claude/settings.json`): deny a tool call with exit 2, or exit 0 and
  `{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny","permissionDecisionReason":"…"}}`.
  Block Stop with `{"decision":"block","reason":"…"}`; input carries `stop_hook_active`;
  8 consecutive continuations maximum.
- Copilot CLI and cloud agent (`.github/hooks/*.json`, `{"version":1,"hooks":{…}}`, fields `bash`,
  `powershell`, `timeoutSec`): deny with stdout
  `{"permissionDecision":"deny","permissionDecisionReason":"…"}` or exit 2. `agentStop` can block.
  **Hook timeouts fail open.** The cloud agent honours only `bash`.
- VS Code agent hooks are in preview; VS Code reads `.github/hooks/*.json` and `.claude/settings.json`.
- Decision: three tiers — *enforced* (Claude Code, Copilot CLI, Copilot cloud agent),
  *preview* (VS Code agent mode), *CI-only* (everything else). Because hook timeouts can fail
  open, git hooks and CI are the guarantee in every tier.
- Sources: https://code.claude.com/docs/en/hooks ,
  https://docs.github.com/en/copilot/reference/hooks-configuration ,
  https://code.visualstudio.com/docs/copilot/customization/hooks

### ADR-010 — One skill copy, in `.claude/skills/`
- Copilot reads `.github/skills`, `.claude/skills` and `.agents/skills`. Claude Code reads
  `.claude/skills` and does not mention `.agents/skills`.
- Decision: install each skill once under `.claude/skills/<name>/`. No `.github/skills` copy.
- Sources: https://code.claude.com/docs/en/skills ,
  https://docs.github.com/en/copilot/concepts/agents/about-agent-skills

### ADR-011 — Question tools per harness
- Claude Code `AskUserQuestion`: 1–4 questions per call, 2–4 options each; not available in subagents.
- VS Code `#vscode/askQuestions`: experimental in 1.109, moved into core in 1.110.
- Copilot CLI: `ask_user` tool. Cloud agent: none documented → numbered-list fallback.
- Sources: https://code.claude.com/docs/en/agent-sdk/user-input ,
  https://code.visualstudio.com/updates/v1_109 , https://code.visualstudio.com/updates/v1_110 ,
  https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference

### ADR-012 — Permission profiles: what can be shipped as a file
- Claude Code: `permissions.allow|ask|deny` and `sandbox.*` in `.claude/settings.json`; deny is
  evaluated first. Shipped as generated config.
- Copilot CLI: flags only (`--allow-tool`, `--deny-tool`, `--allow-url`, `--deny-url`); deny wins.
  No documented repo-level permissions file → ironspec documents the flag set.
- Copilot cloud agent: firewall allowlist is configured in the repository settings UI only →
  documented checklist; this is a guideline, not an enforced rule.
- Sources: https://code.claude.com/docs/en/permissions , https://code.claude.com/docs/en/sandboxing ,
  https://docs.github.com/en/copilot/how-tos/copilot-cli/allowing-tools ,
  https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/customize-the-agent-firewall

### ADR-013 — GitHub mirror API
- Sub-issues have a REST API; limits are 100 sub-issues per parent and 8 levels.
  Projects v2 has a REST API. API version 2026-03-10 has breaking changes for some project item endpoints.
- Limits: `GITHUB_TOKEN` 1,000 requests/hour per repo; about 80 content-creating requests/minute;
  on a limit, honour `retry-after`, else wait at least 60 s and back off exponentially.
- Decision: tasks mirror to sub-issues when a feature has ≤ 100 tasks, otherwise to a checklist.
- Sources: https://docs.github.com/en/rest/issues/sub-issues ,
  https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/adding-sub-issues ,
  https://docs.github.com/en/rest/projects/projects ,
  https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api

### ADR-014 — `merge=union` needs an order-independent fold
- Git documents that union merge "tends to leave the added lines in the resulting file in random
  order". Behaviour on duplicate lines is not documented.
- Decision: state is rebuilt by de-duplicating events on `eid` and sorting on `(ts, eid)`; file
  order is never trusted. `prev` must reference an existing event, so the log is validated as a
  graph, not as a file-ordered chain.
- Sources: https://git-scm.com/docs/gitattributes , https://git-scm.com/docs/githooks

### ADR-015 — Release and supply chain
- PyPI Trusted Publishing from GitHub Actions (`id-token: write`,
  `pypa/gh-action-pypi-publish@release/v1`); PEP 740 attestations are on by default with it.
  SBOM with `cyclonedx-py` (package `cyclonedx-bom`). Build backend `hatchling` (pure wheel).
- Sources: https://docs.pypi.org/trusted-publishers/using-a-publisher/ ,
  https://github.com/pypa/gh-action-pypi-publish , https://docs.pypi.org/attestations/

### ADR-016 — `STATUS.md` is per feature; the root dashboard is derived (deviates from build prompt §4)
- A single committed root `STATUS.md` rewritten on every task transition would conflict between
  feature branches, which is the exact problem §6 of the build prompt sets out to remove.
- Decision: the committed file is `.ironspec/features/<id>-<slug>/STATUS.md`. A root `STATUS.md`
  covering all features is generated on demand and gitignored.
- Source: design reasoning from ADR-014; no external source.

### ADR-017 — Name
- `ironspec` was free on PyPI and npm on 2026-11-16. A GitHub account named `ironspec` exists, so
  the project stays under its current owner. Re-check all three before the first publish.
- Sources: https://pypi.org/pypi/ironspec/json , https://registry.npmjs.org/ironspec , https://github.com/ironspec

## Unverified — check at the named milestone

| Item | Milestone |
|---|---|
| Exact Claude Code tool names to match in the PreToolUse matcher for file edits | M3 |
| Format for blocking `agentStop` inside VS Code (preview) | M3 |
| Question/option limits of `#vscode/askQuestions`; its identifier under the Copilot harness | M3 |
| Whether Copilot hook input arrives on stdin (the docs page does not say so explicitly) | M3 |
| Exact Claude permission patterns for force-push and merge denial | M3 |
| Per-skill licenses in `anthropics/skills` (README says many are Apache-2.0; only `docx` was checked) | M5 |
| GraphQL sub-issue fields; `gh project` subcommands | M6 |
| `uv.lock` behaviour for hash-pinned dependencies | M8 |
| PyPI "too similar name" rule | M8 |
| How union merge treats duplicate lines and a missing trailing newline (covered by tests, not by docs) | M1 |

## Open design points (settled inside milestone plans; no spec change expected)

- How `sync.*` events written by a CI sync job get committed back to the feature branch (M6).
- Docs-site generator for M8 (a development-only dependency, so the pure-wheel rule does not apply).

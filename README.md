# Hermes Agent contribution workspace

This is [dodo-reach/hermes-agent](https://github.com/dodo-reach/hermes-agent),
a public working fork of
[NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent).
It exists to prepare focused contributions to upstream; it is not a separate
Hermes distribution or a runtime installation.

Official product documentation remains at
[hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs/).

## Repository setup

- Local checkout on the primary development machine: `~/hermes-agent`
- `origin`: `dodo-reach/hermes-agent` — push feature branches here
- `upstream`: `NousResearch/hermes-agent` — authoritative source and PR target
- Local `main` tracks `upstream/main` but pushes explicitly to `origin`
- Git identity uses a GitHub noreply address

This fork's `main` intentionally has this fork-only README. **Never base an
upstream PR on `main`**, or the README replacement will appear in the PR.
Always create contribution branches directly from `upstream/main`.

## Where to look

- `run_agent.py` — core conversation loop
- `model_tools.py`, `toolsets.py`, `tools/` — tool discovery and execution
- `hermes_cli/`, `cli.py` — CLI, configuration, setup, and subcommands
- `gateway/` — messaging runtime and platform adapters
- `plugins/` — provider and capability plugins
- `skills/`, `optional-skills/` — bundled and opt-in skills
- `ui-tui/`, `tui_gateway/` — terminal UI and its Python backend
- `apps/desktop/` — Electron desktop client; read its nested `AGENTS.md`
- `tests/`, `scripts/run_tests.sh` — test suite and required Python runner

This map is only an entry point. The filesystem, scoped `AGENTS.md` files, and
current upstream code are authoritative.

## Start every contribution this way

```bash
cd ~/hermes-agent
git status --short --branch
git fetch --prune upstream
git switch -c feat/short-description upstream/main
```

Before changing code:

1. Read `AGENTS.md` completely. It contains the authoritative architecture,
   contribution, security, dependency, and testing rules.
2. Check for a more specific `AGENTS.md` in the area being edited, especially
   under `apps/desktop/`.
3. Reproduce the issue on current `upstream/main` and confirm the intended
   design before implementing a fix.
4. Inspect existing helpers and adjacent tests; prefer the smallest complete
   change over new infrastructure.

## Validate before publishing

Run the narrowest relevant checks while developing, then the required suite
for the affected area. Python tests must use the repository wrapper:

```bash
scripts/run_tests.sh path/to/relevant/test_file.py
git diff --check
git diff --stat upstream/main...HEAD
git diff upstream/main...HEAD
```

Before every public push, verify all of the following:

- The diff contains only work required for the PR.
- No `.env`, credentials, tokens, private keys, auth files, sessions, profiles,
  databases, logs, generated assets, or personal filesystem paths are present.
- Tests exercise behavior, not source text or changing catalog snapshots.
- Commit authorship uses the configured GitHub noreply address.
- `git status --short` is clean.

Then publish only the feature branch:

```bash
git push -u origin feat/short-description
gh pr create --repo NousResearch/hermes-agent
```

Do not force-push `main`. If an existing PR branch must be rewritten, use
`--force-with-lease`, never `--force`.

To refresh the fork-only landing branch without disturbing PR bases:

```bash
git switch main
git fetch upstream
git merge --no-edit upstream/main
git push origin main
```

If upstream also changed `README.md`, resolve that file in favor of this
workspace brief. Feature branches still start from `upstream/main`.

## Retained work

Known contribution history preserved during the 2026-07-16 consolidation:

- `feat/unbroker-eu-dpa-clean` —
  [PR #59806](https://github.com/NousResearch/hermes-agent/pull/59806) work,
  already public
- `origin/codex/agent-control-orchestration` —
  [PR #43030](https://github.com/NousResearch/hermes-agent/pull/43030) public branch
- `preserve/pr-43030-local-20260716` — newer local WIP; not public
- `preserve/runtime-hyperframes-path-fix-20260716` — local WIP; not public
- `archive/pr-*` — local references to closed historical PRs
- `backup/pr-59806-pre-fix-20260716` — local pre-review safety point

Branches under `preserve/`, `archive/`, and `backup/` are reference material,
not PR-ready branches. Do not push or merge them wholesale. Review their diffs,
rebase or cherry-pick only the intended work onto a fresh branch from
`upstream/main`, run the relevant tests, and perform the privacy check above.

## Core upstream constraints

- Preserve prompt-cache stability and strict message-role alternation.
- Prefer existing code, then a CLI command plus skill, service-gated tool,
  plugin, or MCP server before adding a core model tool.
- Put behavioral settings in `config.yaml`; `.env` is for secrets only.
- Use `get_hermes_home()` and `display_hermes_home()` for profile-aware paths.
- Do not add speculative hooks, source-reading tests, or change-detector tests.
- Preserve contributor authorship when salvaging external work.

When this summary and `AGENTS.md` disagree, follow `AGENTS.md` and current
upstream code.

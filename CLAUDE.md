# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`hyperresearch` is a Python package that turns Claude Code itself into a deep-research
agent. It ships two things that must be kept in sync:

1. **A vault CLI** (`hyperresearch` / `hpr`) — markdown+SQLite knowledge base, web/scholarly
   fetching, linting, run manifests.
2. **A prompt harness** — 19 skill markdown files (`src/hyperresearch/skills/`) and 16 subagent
   prompts (string constants in `src/hyperresearch/core/hooks.py`) that `hpr install` renders and
   writes into a target project's `.claude/skills/` and `.claude/agents/`.

Editing a prompt is a code change here: the prompts are Jinja templates, they are golden-tested,
and they reference CLI commands that must exist.

## Commands

```bash
pip install -e ".[dev]"          # dev setup (CI installs exactly this + build)

pytest tests/ -v                 # full suite (~1-2 min)
pytest tests/test_core/test_prompt_golden.py -v          # one file
pytest tests/test_core/test_profiles.py -k field_is_consumed   # one test
ruff check src/ tests/           # lint gate — CI runs this before pytest
mypy src/hyperresearch/          # strict mode; configured but NOT run in CI
python -m build                  # CI also asserts the wheel builds
```

CI matrix: Ubuntu on 3.11/3.12/3.13 plus one `windows-latest` 3.11 job (the maintainer develops on
Windows; socket/path/signal differences have shipped real bugs — see #104). Branch protection pins
the check names, so don't rename the matrix jobs.

Exercising the CLI against a scratch vault:

```bash
cd /tmp/scratch && hyperresearch init . && hyperresearch install .
hyperresearch status -j
```

## Architecture

### Layers

- `core/` — vault (`vault.py`), file→SQLite sync (`sync.py`), schema + migrations (`db.py`,
  `migrations.py`), frontmatter/note IO, fetching (`fetcher.py`, `oa.py`), run manifests
  (`runs.py`), profiles/levers/render (the prompt-templating layer), and `hooks.py` (all subagent
  prompt constants + every installer).
- `cli/` — one module per command group, wired into the typer app in `cli/__init__.py`.
- `web/` — pluggable fetch/search providers behind `web/base.py::get_provider` (`builtin`,
  `crawl4ai`, `exa`, `tavily`, `parallel`, `serply`). Optional ones are `ImportError`-guarded and
  map to a `[project.optional-dependencies]` extra.
- `scholar/` — eight literature/specialist providers behind one registry, merged and deduplicated
  by DOI then normalized title+year.
- `mcp/`, `serve/`, `export/`, `graph/`, `indexgen/`, `search/`, `models/`.

### Markdown is truth, SQLite is a cache

Notes are markdown + YAML frontmatter under `research/notes/`. The SQLite index under
`.hyperresearch/` is fully derived — deleting it and running `hyperresearch sync` must reconstruct
it. Any new note field therefore lives in frontmatter first and is *mirrored* into SQLite by
`sync.py`. Bump `SCHEMA_VERSION` in `core/db.py` and add a matching idempotent migration in
`core/migrations.py`; `tier`, `type`, `status` and `content_type` are CHECK-constrained
vocabularies, so widening one means rebuilding the table in a migration.

### Prompt rendering (the non-obvious part)

Skill files and agent prompt bodies are Jinja templates with **custom delimiters** — `<< var >>`,
`<% block %>`, `<# comment #>` — because the prompts legitimately contain `{{ }}` and `{ }` for
spawn placeholders and JSON examples (`core/render.py`). Rendering is `StrictUndefined`: a typo
fails the install instead of shipping a prompt with a hole.

The render context is every profile in `core/profiles.py` by name, plus `p` for the active one.
Profiles are the *scale gears* — source targets, fan-out, word targets, per-agent model map — and
users override them via `[profile.<name>]` in `.hyperresearch/config.toml`. Rules that follow:

- **Numbers live in profiles, never hardcoded in prompt prose.** `test_profiles.py` asserts every
  `Profile` field is read somewhere outside `profiles.py`; a field you add and don't template is a
  test failure.
- **Posture lives in levers, numbers never do** (`core/levers.py`). Levers render role-scoped shim
  files pasted verbatim into subagent spawns. Shim text must not restate a budget. The cite-checker
  and ship gate deliberately get no shim.
- **Golden tests** (`tests/fixtures/golden_prompts/`) pin the `full`-profile render byte-for-byte.
  Any deliberate prompt or profile change requires regenerating the affected golden in the same
  commit and saying why — that's what makes prompt drift reviewable.

### Runs

Each run owns `research/runs/<vault_tag>/` with a `run.json` manifest (`core/runs.py`): step
state, spend, chapter plan, escalation queue. `run resume` derives the next skill slug from
`hooks.STEP_SKILL_BY_ID`, which is generated from `_HYPERRESEARCH_STEP_SKILLS` — so a renamed step
skill can never be advertised as a slug that doesn't exist. Run tags are validated by
`vault.validate_run_tag` (path-traversal seam); pass a tag through it rather than joining it onto
`runs/` yourself.

### Security invariants worth not breaking

- **Fetched bodies are untrusted data.** `core/untrusted.py` wraps web-sourced bodies in
  `<untrusted-source>` fences on every path that serves a body (`note show` in all forms, `search`
  with bodies). Wrapping happens *after* truncation so the closing fence can't be severed. Notes
  written by our own subagents are not wrapped.
- **Outbound fetches go through `web/safe_http.py`** (SSRF gate, size caps) — including URLs that
  arrive inside third-party API JSON, e.g. open-access locations.
- API keys are scoped per host (`S2_API_KEY` only to semanticscholar.org, etc.); the shared fetch
  helper must never leak a header to another provider.

## Conventions

- **CLI output is dual-mode.** Data commands take `-j/--json` and return an `Envelope`
  (`models/output.py`) via `cli/_output.py::output`. Agents consume the JSON; humans get rich.
  Errors carry a stable `error_code`.
- **Imports inside command bodies.** CLI modules import `hyperresearch.core.*` lazily inside the
  function to keep `hpr --help` fast and to keep optional deps optional. Follow the local style.
- **Ruff** with `E,F,I,N,W,UP,B,SIM,RUF`, line length 100 (E501 ignored). Files with deliberate
  en dashes have per-file `RUF001/RUF002` ignores in `pyproject.toml` — add one rather than
  replacing an en dash that a golden asserts on.
- **CHANGELOG.md** is Keep-a-Changelog style; add entries under `[Unreleased]` only, written as
  prose explaining the wrong behavior and the fix, crediting the issue number and reporter.
- **Version** is set in `pyproject.toml` and `hyperresearch.__version__`; a packaging test asserts
  they match, and that `[dev]` covers every optional provider the suite imports.
- PRs stay narrowly scoped: no drive-by reformatting, no unrelated output-string or fixture edits
  (see `.github/pull_request_template.md`).

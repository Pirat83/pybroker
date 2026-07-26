# Dependency Migration Triage: ruff `>=0.3.4` → `>=0.16.0`

**Source PR:** [Pirat83/pybroker#17](https://github.com/Pirat83/pybroker/pull/17) — "Update ruff requirement from >=0.3.4 to >=0.16.0" (dependabot, base branch `dev`)
**Mode:** Dry-run / calibration. No `gh pr create` or `gh pr comment` was executed against any real repo. This report is what would be produced.

---

## Phase 0 — Identify the bump

- **Package:** `ruff` (dev/lint tooling only — not imported by `src/pybroker`)
- **Declared in:** `requirements.txt:20` → `ruff>=0.3.4` (unpinned upper bound; Dependabot is just bumping the floor)
- **Change:** `>=0.3.4` → `>=0.16.0`. Diff is a single line in `requirements.txt`.
- **Consumers:** `setup.cfg` tox envs — `[testenv:format]` runs `ruff format {posargs:--diff src tests}` (`setup.cfg:73`), `[testenv:lint]` runs `ruff check {posargs:src tests}` (`setup.cfg:80`). Both are wired into CI as separate jobs (`.github/workflows/main.yml` `format`/`lint` jobs, and identically in `.github/workflows/schedule.yml`), each a required check before `publish`.
- **Config:** `[tool.ruff]` / `[tool.ruff.lint]` / `[tool.ruff.format]` in `pyproject.toml:5-88`.

**Right-sizing:** This is a dev-tool-only bump (not shipped to package users), but the version jump is large — `0.3.4` (Mar 2024) to `0.16.0` (2026-07-23) spans roughly two years and dozens of minor releases. Per the skill's own guidance, a linter is exactly the kind of dependency where "still imports" says nothing about whether its *definition of correct* changed (see mypy case study). That argues for real scrutiny of Phase 3 despite this being "just a linter." The mitigating factor, confirmed below, is that this repo pins its lint rule *selection* explicitly rather than relying on ruff's defaults — which turns out to matter a lot.

## Phase 1 — Get the real changelog, not the PyPI blurb

Read ruff's own [v0.16.0 blog post](https://astral.sh/blog/ruff-v0.16.0) (the authoritative migration guide ruff itself points to), not just Dependabot's truncated changelog excerpt. Headline breaking changes in 0.16.0:

1. **Default rule set expansion: 59 → 413 rules enabled by default** (new flake8-bugbear and pyupgrade rules turned on by default). This is the single largest behavior change in the release and the one most likely to newly-fail an unpinned `ruff check`.
2. **Markdown formatting**: `ruff format` now formats Python code blocks inside Markdown/Quarto files by default, with new `<!-- fmt: off -->` suppression.
3. **New `ruff: ignore` / `ruff: file-ignore` suppression comments**, alongside existing `noqa` and `ruff: disable`/`enable`.
4. **JSON output**: `filename`/`location`/`end_location`/fix-edit location fields may now be `null` instead of defaulting to `""`/row 1, col 1.
5. Formatter now shows fixes in `check`/`format --check` output; `format --check` gained the `github`/`gitlab` output formats.

Critically, the blog post itself anticipates exactly this repo's situation and gives the escape hatch verbatim:

> "If you want to revert to the old default rule set, you can select the old rules with: `[lint] select = ["E4", "E7", "E9", "F"]`"

That is **byte-for-byte the `select` line already present at `pyproject.toml:53`.** This repo isn't relying on ruff's defaults at all — it explicitly selects only pycodestyle errors (E4/E7/E9) and pyflakes (F), which was true before 0.16.0 and remains the *documented* way to opt out of the new 413-rule default set. Finding #1, the scariest-looking line in the whole release, does not apply here by construction of the existing config.

I did not exhaustively transcribe every intermediate 0.4.x–0.15.x changelog entry (impractical for a ~2-year, dozens-of-release span) — instead I verified empirically (Phase 3) that the actual rule set selected by this config and the actual formatter output are unchanged end-to-end across the full range, which is strictly stronger evidence for *this repo* than a changelog read alone.

## Phase 2 — Map to actual usage

- `ruff` has **zero usage sites in `src/` or `tests/`** as an imported/called API — it's invoked only as an external CLI via tox (`setup.cfg:73,80`) and CI (`.github/workflows/main.yml:20-21,36-37`; `.github/workflows/schedule.yml:21-22,37-38`).
- Config surface in `pyproject.toml`:
  - `pyproject.toml:36-38` — `line-length = 79`, `indent-width = 4`
  - `pyproject.toml:41` — `target-version = "py312"`
  - `pyproject.toml:43-47` — `[tool.ruff.lint.per-file-ignores]` (E203/E402 on `src/pybroker/*.py`, F401/E402 on `__init__.py`, E203/E402/F403/F405 on test files)
  - `pyproject.toml:53-54` — `select = ["E4", "E7", "E9", "F"]`, `ignore = ["E402"]`
  - `pyproject.toml:64-88` — `[tool.ruff.format]` (double quotes, space indent, auto line-ending, docstring formatting disabled)
- `noqa` usage in the codebase: exactly 5 occurrences, all `# noqa: F401` on `from .fixtures import *` lines (`tests/test_log.py:10`, `tests/test_indicator.py:14`, `tests/test_model.py:12`, `tests/test_strategy.py:14`, `tests/test_data.py:15`). No `ruff: ignore` / `ruff: disable` comments anywhere (grep returned nothing) — the new suppression syntax (item 3 above) is additive, not a migration requirement.
- No `.md` files exist under `src/` or `tests/` — the new default Markdown-formatting behavior (item 2 above) is scoped out simply because CI only ever runs `ruff format` against `src tests` (`setup.cfg:73`), never against `README.md` or `docs/`.
- No code anywhere in `src/`, `tests/`, or CI config parses ruff's JSON output (grepped for `--output-format`/`--format` in workflows and `setup.cfg` — no hits), so item 4 (nullable JSON fields) is not applicable.
- No test in `tests/` invokes `ruff` programmatically (grepped `tests/` for `ruff` — no hits); the only place ruff's behavior is "tested" is the CI `format`/`lint` jobs themselves.

## Phase 3 — Migration plan

| Usage site | Classification | Reasoning |
|---|---|---|
| `select = ["E4","E7","E9","F"]` (pyproject.toml:53) vs. new 59→413 default rule expansion | **Safe as-is** | Explicit `select` fully replaces ruff's default rule set in every version tested; the new defaults never apply here. Confirmed empirically (below) that the *set of rule codes* matching this select expression is identical between 0.3.4 and 0.16.0 (no E4/E7/E9/F codes were added, renamed, or removed in that prefix across the range). |
| `ruff format` invocation on `src tests` (setup.cfg:73) vs. new Markdown-formatting-by-default | **Safe as-is** | No `.md` files live under `src/` or `tests/`, and CI never points `ruff format` at repo docs/README, so the new default behavior has no files to act on. |
| `noqa: F401` comments (5 sites, test files) vs. new `ruff: ignore`/`ruff: file-ignore` syntax | **Safe as-is** | `noqa` remains fully supported; the new syntax is purely additive, not a replacement requiring migration. |
| Any CI script parsing ruff JSON output | **N/A — not used** | Grepped CI workflows and `setup.cfg`; nothing invokes `--output-format json` or parses ruff output programmatically. |
| `target-version = "py312"` (pyproject.toml:41) | **Safe as-is** | Confirmed via `ruff check --show-settings`: `py312` resolves identically and is still a supported target in 0.16.0. |

Applying the case-studies discipline explicitly, per usage site: *"if I changed nothing in my code, could this dependency's new version make this line do something different than before?"* For every site above the answer is empirically no (Phase 3 verification below), and for the one bullet in the release notes that *could* have changed behavior with zero signature/config change on this repo's side — the default rule set jump — this repo's own `select` line was already the documented opt-out before 0.16.0 existed.

### Empirical verification (not just changelog inference)

Ran both the currently-floor-pinned version and the new target version against the real codebase, using the repo's own config, exactly as CI invokes them:

```
venv-old: ruff 0.3.4    venv-new: ruff 0.16.0   (also spot-checked ruff 0.15.12, the version
                                                  actually resolved today by requirements.txt's
                                                  unbounded >=0.3.4 floor, i.e. what CI already runs)

$ ruff check src tests          (old 0.3.4)  -> All checks passed!   (exit 0)
$ ruff check src tests          (new 0.16.0) -> All checks passed!   (exit 0)

$ ruff format --diff src tests  (old 0.3.4)  -> 33 files already formatted (exit 0, no diff)
$ ruff format --diff src tests  (new 0.16.0) -> 33 files already formatted (exit 0, no diff)
```

Also diffed the full set of rule codes matching prefixes `E4`/`E7`/`E9`/`F` (i.e. exactly what this repo's `select` activates) between the two ruff versions via `ruff rule --all`: **60 codes on each side, identical set, zero diff.** So not only does the codebase pass both versions, the actual rule surface being applied to it hasn't shifted underneath the config at all.

## Phase 4 — Coverage check

Nothing was flagged as other-than-"safe as-is" in Phase 3, so per the skill's own instruction ("for every usage site flagged as anything other than safe as-is") there is nothing here requiring a coverage-illusion check. For completeness: the only "test" of ruff's behavior in this repo is the CI `format`/`lint` tox jobs themselves (`setup.cfg:73,80`), and I ran the real equivalent of both jobs directly against both ruff versions rather than trusting a coverage/CI-green proxy — this is closer to "watched it pass" than "assumed it would."

## Phase 5 — Regression test

Not applicable. Phase 3/4 found no breaking change, no behavior drift, and no coverage gap — writing a regression test here would be manufacturing evidence for a non-issue, which the skill explicitly says not to do. Stopping here per Phase 5's guidance.

## Phase 6 — Fix what's fixable

Nothing to fix. No config changes, no code changes, no deprecation warnings observed with `ruff check`/`ruff format` under 0.16.0.

## Phase 7 — Would-be PR (dry-run — not opened)

Per the skill's dry-run instructions, this is the PR/comment content that *would* be produced, shown here instead of calling `gh pr create` or `gh pr comment`.

Given there is nothing to fix, the right artifact is a **comment on the originating Dependabot PR**, not a new PR (a PR with an empty diff would be noise) — this matches the skill's explicit guidance: "if a whole new PR would be empty, [use] a comment on the Dependabot PR."

---

**Would-be comment on Pirat83/pybroker#17:**

> ## Triaged — safe to merge, verified by execution
>
> Ran a full dependency-migration triage on this bump (`ruff>=0.3.4` → `>=0.16.0`) rather than relying on CI-green alone. Summary:
>
> **What was verified by actually executing something** (not inferred from changelogs):
> - Installed both `ruff==0.3.4` and `ruff==0.16.0` in isolated venvs and ran the exact commands CI runs (`ruff check src tests`, `ruff format --diff src tests`) against this repo's real `pyproject.toml` config on both. Both pass identically: `All checks passed!` / `33 files already formatted`, no diff either way.
> - Diffed the full set of rule codes matching this repo's `select = ["E4", "E7", "E9", "F"]` (pyproject.toml:53) between the two ruff versions via `ruff rule --all` — 60 codes on each side, byte-identical set.
>
> **What was inferred from ruff's own release notes** (not independently re-derived):
> - ruff 0.16.0's headline breaking change is expanding the *default* rule set from 59 to 413 rules. This repo is unaffected because it never relies on ruff's defaults — `pyproject.toml:53` already pins an explicit `select`, and per [ruff's own v0.16.0 blog post](https://astral.sh/blog/ruff-v0.16.0), `select = ["E4", "E7", "E9", "F"]` is the documented way to keep the pre-0.16.0 default behavior. This repo's config already matches that exact line.
> - The new default Markdown-formatting behavior doesn't apply: CI only ever runs `ruff format` against `src tests` (setup.cfg:73), and there are no `.md` files under either directory.
> - The other 0.16.0 breaking changes (new `ruff: ignore`/`ruff: file-ignore` suppression syntax, nullable JSON output fields) are additive or apply only to machine-readable output this repo doesn't consume — grepped CI workflows and `setup.cfg` for `--output-format`, no hits.
>
> No code or config changes needed. No regression test added — Phase 3/4 of the triage found nothing to guard against, and manufacturing a test for a non-issue would just be noise. Safe to merge as-is, or trust this and merge directly — your call on how deep to look yourself.

---

## Bottom line

**Nothing in this codebase needs to change because of this bump.** The one part of the 0.16.0 changelog that looked genuinely dangerous on paper — the 59→413 default rule set expansion — turns out to be exactly the scenario this repo's existing `pyproject.toml:53` config was already defending against (and matches ruff's own documented compatibility shim verbatim). This was confirmed by actually running both ruff versions against the real codebase and diffing both the lint/format output and the underlying rule-code set, not by trusting the Dependabot changelog blurb or CI's green checkmark alone.

# Triage: Pirat83/pybroker PR #17 — "Update ruff requirement from >=0.3.4 to >=0.16.0"

**Verdict: No code changes required. Safe to merge as-is.**

This is a dry-run — no `gh pr create` / `gh pr comment` was executed. Everything below
(including the draft PR comment at the end) is what would be produced, not what was posted.

## 1. What the PR actually does

- Author: `dependabot[bot]`, base branch `dev`, head `dependabot/pip/dev/ruff-gte-0.16.0`.
- Diff is exactly one line in `requirements.txt`:
  `ruff>=0.3.4` → `ruff>=0.16.0`.
- No other files touched. This is a version-floor bump, not a version pin — it widens
  the allowed range, it doesn't force any specific ruff release.

Jump size is large: ruff 0.3.4 → 0.16.0 spans ~13 minor releases, and 0.16.0's own
changelog calls out a genuinely breaking change worth checking seriously:

> Ruff now enables a much larger set of rules by default (413, up from 59).

That line is the reason this PR deserves more than a rubber stamp — normally a jump like
that would risk a pile of new lint failures on `ruff check` with no config changes.

## 2. Where this pin is actually consumed

Checked every place `requirements.txt` and ruff are referenced in this repo:

- `.readthedocs.yml` installs `requirements.txt` for the docs build (ruff isn't invoked
  there — irrelevant to lint/format behavior).
- `.github/actions/setup-pybroker/action.yml` hashes `requirements.txt` + `setup.cfg`
  only as a pip cache key — not a behavioral dependency.
- **The CI lint/format jobs do not consume this file at all.** `.github/workflows/main.yml`
  and `schedule.yml` both run `tox -e format` and `tox -e lint`. Those tox environments
  are defined in `setup.cfg`:

  ```ini
  [testenv:format]
  skip_install = True
  deps =
      ruff
  commands =
      ruff format {posargs:--diff src tests}

  [testenv:lint]
  skip_install = True
  deps =
      ruff
  commands =
      ruff check {posargs:src tests}
  ```

  Both envs declare `deps = ruff` with **no version constraint at all**. Tox builds an
  isolated venv for each and pip resolves to the latest ruff release available at run
  time — completely independent of the `ruff>=0.3.4` (or `>=0.16.0`) line in
  `requirements.txt`.

**Consequence:** this PR does not change what CI installs or runs. CI's lint/format
jobs have already been running against the newest ruff (likely already ≥0.16.0 today,
via `schedule.yml`'s daily cron) regardless of whether this PR merges. A green CI run
on this PR isn't new evidence either way — it was never gated by this file. The pin
bump only affects contributors who do `pip install -r requirements.txt` and then run
`ruff` directly outside of tox.

## 3. Ruff 0.16.0 breaking changes vs. this repo's actual config

`pyproject.toml` `[tool.ruff]`:

```toml
line-length = 79
indent-width = 4
target-version = "py312"

[tool.ruff.lint]
select = ["E4", "E7", "E9", "F"]
ignore = ["E402"]

[tool.ruff.format]
docstring-code-format = false
```

Checked each 0.16.0 breaking-change item against this config:

| 0.16.0 change | Applies here? | Why |
|---|---|---|
| Default rule set 59 → 413 | **No** | `select = ["E4","E7","E9","F"]` is explicit. Ruff only falls back to its built-in defaults when a project has no explicit `select`/`extend-select`. An explicit `select` fully replaces the default set, so the version-dependent default expansion is inert here — the enabled rule set is unchanged by version. |
| Markdown code-block formatting on by default | **No** | `testenv:format` runs `ruff format --diff src tests` — scoped to Python source dirs only, not the repo root, so `.md` files are never in scope regardless of version. `docstring-code-format = false` also means docstring snippets aren't reformatted. |
| `ruff: ignore` end-of-line / preceding-line comments | **No effect** | Purely additive syntax. Existing code uses standard `# noqa: F401` comments (5 occurrences, all in `tests/`) — untouched by this feature. |
| Fixes shown in `check`/`format --check` output; new JSON `null` fields | **No effect** | Output-format/cosmetic only. No tooling in this repo parses ruff's JSON output. |

## 4. Empirical verification

No pinned ruff was installed locally, so both the old floor and the new floor were
installed via `uvx` and run against the real repo tree (`src tests`, matching exactly
what CI's tox envs run):

```
$ uvx ruff@0.3.4  check src tests        → All checks passed!
$ uvx ruff@0.16.0 check src tests        → All checks passed!

$ uvx ruff@0.3.4  format --diff src tests → 33 files already formatted
$ uvx ruff@0.16.0 format --diff src tests → 33 files already formatted
```

Identical, clean results on both versions — zero new violations, zero formatting
diffs introduced by the jump from 0.3.4 to 0.16.0.

## 5. Residual observation (not a blocker for this PR)

The `deps = ruff` (unpinned) in `setup.cfg`'s `testenv:format`/`testenv:lint` means
lint/format CI results are not reproducible over time — a future ruff release could
introduce a real regression and the daily `schedule.yml` cron would be the first to
surface it, with no way to pin/roll back via `requirements.txt`. This is a
pre-existing gap this investigation surfaced, not something introduced by PR #17, and
is out of scope for this dependency bump. Worth a separate follow-up if the goal is
reproducible CI (e.g. pin `ruff==<version>` in the tox `deps` and let Dependabot bump
that instead of/in addition to `requirements.txt`).

## 6. Draft PR comment (not posted — dry-run)

> Verified this locally rather than relying on CI alone, since ruff 0.16.0 enables a
> much larger default rule set (413 vs. 59) and that's the kind of change that can
> silently break a project relying on ruff's defaults.
>
> Not a concern here: `pyproject.toml` sets an explicit
> `[tool.ruff.lint] select = ["E4", "E7", "E9", "F"]`, which fully overrides ruff's
> built-in defaults regardless of version — so the expanded default rule set never
> applies to this repo.
>
> Also note: this bump doesn't actually change what CI lints/formats against. The
> `format`/`lint` tox envs in `setup.cfg` already install an unpinned `ruff`
> (`deps = ruff`), independent of `requirements.txt` — so CI has already been running
> against latest ruff regardless of this PR.
>
> Ran both floors (`0.3.4` and `0.16.0`) directly against `src tests` to confirm:
> `ruff check` → "All checks passed!" on both; `ruff format --diff` → "33 files
> already formatted" on both, no diff. No code changes needed. LGTM to merge.

## Files referenced

- `/home/pirat/Projects/pybroker/pyproject.toml` (ruff config, lines 5-88)
- `/home/pirat/Projects/pybroker/requirements.txt` (the changed line, line 20)
- `setup.cfg` on `origin/dev` (tox `testenv:format` / `testenv:lint` definitions — not
  present on the currently checked-out branch's working tree at the repo root path
  checked; fetched via `git show origin/dev:setup.cfg`)
- `/home/pirat/Projects/pybroker/.github/workflows/main.yml`
- `/home/pirat/Projects/pybroker/.github/workflows/schedule.yml`
- `/home/pirat/Projects/pybroker/.github/actions/setup-pybroker/action.yml`
- `/home/pirat/Projects/pybroker/.readthedocs.yml`

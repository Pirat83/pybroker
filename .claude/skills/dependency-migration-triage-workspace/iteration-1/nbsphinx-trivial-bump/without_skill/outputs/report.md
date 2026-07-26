# Triage: Pirat83/pybroker PR #19 — Bump `nbsphinx` from `>=0.8.11` to `>=0.9.8`

**Mode:** dry run / calibration. No `gh pr create` or `gh pr comment` was executed against the real repo. Everything below is analysis only, produced by reading repo files and by actually building the Sphinx docs twice (old vs. new `nbsphinx`) in disposable local venvs.

## TL;DR

**Safe to merge.** The diff itself is a one-line, no-op-in-practice version-floor bump. The one thing worth registering as a separate finding: **none of the CI checks that passed on this PR exercise Sphinx/nbsphinx at all**, so "CI is green" is not evidence the docs still build — that had to be verified out-of-band, which I did (see "What I actually verified" below). Verdict: merge the PR, but treat the CI gap as a follow-up item, not a blocker for this PR.

## What the PR actually is

- Source: Dependabot, `dependabot/pip/dev/nbsphinx-gte-0.9.8`, base branch `dev`.
- Diff: `requirements.txt`, +1/-1 line: `nbsphinx>=0.8.11` → `nbsphinx>=0.9.8`.
- `setup.cfg`'s `docs` tox deps list `nbsphinx` **unpinned**, so no second file to change there.
- Status checks (all green): `format`, `lint`, `typecheck`, `test` (3.11/3.12/3.13), `build sdist`, ASV benchmarks. `mergeable` was reported `UNKNOWN` by the GitHub API at fetch time (normal — GitHub hasn't finished computing it, not a signal of conflict).

## The trap: CI passing tells you nothing about this dependency

I read `.github/workflows/main.yml` (the only workflow that runs on push and produced the green checks on this PR). Its jobs are `format` (ruff format), `lint` (ruff check), `typecheck` (mypy), `test` (pytest matrix), `build_source_dist`, `publish`. **None of them import Sphinx, run `sphinx-build`, or touch `nbsphinx` in any way.** `nbsphinx` isn't imported by any file under `src/` or `tests/` — it only appears in `requirements.txt` and `setup.cfg`'s docs deps.

The actual documentation build lives entirely outside this PR's gate:

- `.readthedocs.yml` builds `docs/source/conf.py` via Read the Docs on its own trigger — not part of the GitHub PR checks shown above, and not affected by this PR either way. I could not confirm from the repo alone whether this fork's `dev` branch has an active RTD project wired up (the README's RTD badge points at `pybroker.com`, which reads like the upstream project's domain carried over in the fork, not something verifiable without RTD account access).
- `setup.cfg` also defines a local `[testenv:docs]` tox env (`sphinx-apidoc` + `sphinx-build -n -W --keep-going -b html docs/ docs/_build/`). This is **not** in `tox.ini`'s `envlist`, is never invoked by CI, and — separately, pre-existing, unrelated to this PR — its source directory argument (`docs/`) doesn't actually contain `conf.py` (that lives in `docs/source/conf.py`), so running `tox -e docs` as currently written would fail immediately regardless of any nbsphinx version. Worth a follow-up ticket on its own, but not something this Dependabot PR introduced or needs to fix.

So the checks Dependabot shows you as "passing" are structurally incapable of catching an nbsphinx/Sphinx incompatibility. That has to be checked by hand.

## What I actually verified (real build, not just changelog reading)

I built the docs twice in disposable venvs, from this repo's actual `docs/source/`, using the same current-latest Sphinx (`9.1.0`, what an unbounded `Sphinx>=5.3.0` resolves to today) and the same `sphinx_rtd_theme` (`3.1.0`), varying only the `nbsphinx` version:

**Old (`nbsphinx==0.8.11`, the pre-PR floor) + Sphinx 9.1.0:**
Build reaches "writing output... 100%" for every page, then **hard-crashes** (exit code 2) during the HTML-collect-pages phase:

```
sphinx.errors.ExtensionError: Handler <function html_collect_pages at ...> for event
'html-collect-pages' threw an exception (exception: module 'sphinx.util' has no attribute
'status_iterator')
```

Full traceback pinpoints the failing line inside nbsphinx itself: `nbsphinx.py:2094, in html_collect_pages` → `status_iterator = sphinx.util.status_iterator`. `sphinx.util.status_iterator` was removed from Sphinx; nbsphinx 0.8.11 still calls it. Consequence: no `objects.inv`, no `searchindex.js`, no `search.html` — the site never finishes.

**New (`nbsphinx==0.9.8`, the PR's target) + same Sphinx 9.1.0:**
Build completes successfully — full HTML output, all 11 notebooks rendered, search index, object inventory all written. Exit code was 1 only because `-W --keep-going` upgrades warnings to a non-zero exit; the 37 warnings present are identical between both runs (autodoc import failures because `numpy`/`pandas` aren't installed in my throwaway venv, plus one pre-existing `benchmarking.rst` "not in any toctree" warning) — i.e., **zero new warnings introduced by the nbsphinx bump**, confirmed by diffing the warning sets (`comm` showed no lines unique to either side).

So this is a real, reproducible incompatibility between old nbsphinx and current Sphinx — not hypothetical. The bump is what keeps the docs buildable at all against a modern Sphinx.

## Why the PR is nonetheless close to a no-op today

`requirements.txt` has **no upper bound** on `nbsphinx` (`>=0.8.11` then `>=0.9.8`) and the repo has **no lock file** anywhere (checked for `*.lock`, `requirements*.txt`, `constraints*.txt` — none exist besides `requirements.txt` itself). Pip's resolver always picks the highest version satisfying all constraints when there's no ceiling, so a fresh `pip install -r requirements.txt` **already installs `nbsphinx` 0.9.8 today, before this PR merges** (confirmed via `pip index versions nbsphinx` showing `0.9.8` as latest, and it's what actually resolved in my test venv without pinning).

So in any environment built from scratch (CI runners, a fresh RTD build), merging this PR changes nothing about what gets installed. Its actual effect is defensive: it raises the manifest's stated floor so that (a) the requirement file documents accurately what's needed, and (b) a stale/cached environment that still has 0.8.11 installed will be forced to upgrade on the next `pip install -r requirements.txt`, rather than silently keeping the broken pairing shown above.

## Dependency-graph sanity check

Pulled current PyPI metadata for the packages in play:

- `nbsphinx==0.9.8` requires `sphinx!=8.2.0,!=8.2.1,>=1.8` (excludes only two known-buggy Sphinx point releases), `docutils>=0.18.1`, `nbconvert!=5.4,>=5.3`, `nbformat`, `traitlets>=5`.
- `sphinx-rtd-theme==3.1.0` (what `sphinx_rtd_theme>=2.0.0` resolves to unbounded) requires `sphinx<10,>=6`, `docutils<0.23,>0.18`.
- Both are jointly satisfiable at Sphinx 9.1.0 (what's actually resolved) — confirmed by the successful build above. No new transitive conflict introduced by this PR.
- `docs/source/conf.py` sets no nbsphinx-specific options (no `nbsphinx_prolog`, `nbsphinx_epilog`, `mathjax_config`/`mathjax3_config` overrides, `nbsphinx_allow_errors`, etc.), so none of the config-surface renames across the 0.8→0.9 nbsphinx line apply here — one less thing to worry about.

## Recommendation

1. **Merge PR #19.** It's a correctness fix for the manifest (matches what's already being installed) and, per the reproduction above, it's the difference between a docs build that completes and one that hard-crashes against current Sphinx.
2. **Don't rely on this PR's green CI as your signal** — it can't be, structurally. If you want an actual pre-merge gate for future Sphinx/nbsphinx/theme bumps, that means adding a docs-build job to `.github/workflows/main.yml` (or fixing and wiring up the existing `tox -e docs` env, which currently points at the wrong source directory and isn't in `tox.ini`'s `envlist`). Flagging this as a separate, non-blocking follow-up rather than acting on it, per the dry-run scope.
3. No action needed on `sphinx_rtd_theme`, `Sphinx`, or `docutils` pins — nothing here forces a companion bump.

## Evidence trail (for reproducibility)

- `gh pr view 19 --repo Pirat83/pybroker --json ...` — PR metadata, diff, status checks.
- `.github/workflows/main.yml` — confirms no docs/Sphinx job exists in CI.
- `.readthedocs.yml`, `docs/source/conf.py`, `setup.cfg` `[testenv:docs]` — confirms where/how docs are actually built (and that the local tox env is currently broken, pre-existing/unrelated).
- Two isolated venvs (`nbsphinx==0.8.11` vs `nbsphinx==0.9.8`, same latest `sphinx`/`sphinx_rtd_theme`), each run via `sphinx-build -n -W --keep-going -b html docs/source <out>` against this repo's real `docs/source/`. Old crashes (`sphinx.util.status_iterator` AttributeError inside `nbsphinx.py:2094`); new completes with only pre-existing, environment-caused warnings (identical warning set on both sides, diffed with `comm`).
- PyPI JSON metadata for `nbsphinx` (0.8.11 and 0.9.8) and `sphinx-rtd-theme` (latest) to confirm dependency bounds.

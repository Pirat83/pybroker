# Migration Triage: nbsphinx `>=0.8.11` → `>=0.9.8`

**PR under triage:** [Pirat83/pybroker#19](https://github.com/Pirat83/pybroker/pull/19) — "Update nbsphinx requirement from >=0.8.11 to >=0.9.8" (Dependabot, base branch `dev`)
**Mode:** dry-run / calibration — no `gh pr create` or `gh pr comment` was executed against any real repo. Everything below is what *would* be posted.

## Phase 0 — Identify the bump

- Package: `nbsphinx`, declared in `requirements.txt:10` — `nbsphinx>=0.8.11` → `nbsphinx>=0.9.8`. Single-line diff, no other files touched.
- `nbsphinx` is **not** a runtime dependency: it does not appear in `setup.cfg`'s `[options] install_requires` (which lists only `akshare`, `alpaca-py`, `diskcache`, `joblib`, `numba`, `numpy`, `pandas`, `progressbar2`, `yahooquery`, `yfinance`). It is a docs-build-only tool, referenced in:
  - `docs/source/conf.py:18` — `extensions = ["nbsphinx", ...]`
  - `setup.cfg` `[testenv:docs]` — `deps = nbsphinx / sphinx` (tox docs environment)
  - `requirements.txt:10` — used by Read the Docs (`.readthedocs.yml` installs from `requirements.txt`)
- Zero references in `src/` or `tests/` (confirmed by grep).
- **Right-size read:** dev/docs tooling only, `src/` never imports it, and the version range (0.8.11 → 0.9.8) spans ~10 releases — moderate range, but low blast radius given the near-zero usage surface. Warrants a real changelog read (Phase 1) and a check of `conf.py`/notebooks (Phase 2), not a deep code migration.

**The load-bearing finding, stated up front:** none of this repo's CI checks build the docs at all. `.github/workflows/main.yml` runs `format`, `lint`, `typecheck`, `test` (py3.11/3.12/3.13), `build_source_dist`, and `publish` — there is no docs/sphinx job anywhere in that workflow, nor in `asv-pr.yml`/`asv-nightly.yml`/`schedule.yml`. The `[testenv:docs]` tox environment that actually invokes `sphinx-apidoc`/`sphinx-build` exists in `setup.cfg` but is **never invoked by any GitHub Actions job**. The only place docs actually get built is Read the Docs (`.readthedocs.yml`), an external service not represented as a PR status check at all (`gh pr checks 19` shows only Publish/Build source dist/Check formatting/Lint/Test×3/Type check/asv continuous — no RTD check).

So: this PR's "all checks passed" is **necessarily true regardless of whether nbsphinx 0.9.8 works**, because nothing in CI ever exercises nbsphinx. Trusting green CI here would be a category error, not just an incomplete signal.

## Phase 1 — Real changelog, full range 0.8.11 → 0.9.8

Read `NEWS.rst` directly from `github.com/spatialaudio/nbsphinx` (not the truncated Dependabot summary, which only went back to 0.9.1). Every release in the crossed range:

| Version | Change |
|---|---|
| 0.8.12 | Implement "link" galleries (without nested sub-documents) |
| 0.9.0 | Split `nbsphinx.py` module into a package; separate CSS files; **`sphinx_gallery.load_style` can no longer be used for the gallery CSS**; last image in a notebook becomes default thumbnail |
| 0.9.1 | pandoc: disable "smart" option only for pandoc version 2.0+ |
| 0.9.2 | `sphinx_immaterial` theme support improved; link support for `#`-prefixed links; in-text citations; LaTeX admonition titles |
| 0.9.3 | Fix gallery regression in Sphinx 7.2 |
| 0.9.4 | **Require `docutils >= 0.18.1`**; minor fixes |
| 0.9.5 | Miscellaneous fixes |
| 0.9.6 | **Markdown: allow lists without leading blank line** (behavioral rendering change, no signature change) |
| 0.9.7 | Disable Sphinx 8.2+ support (temporarily) |
| 0.9.8 | Re-enable Sphinx 8.2+; support `text/x-rst` MIME type in raw cells; support for `mathjax4_config` |

Nothing in this range is a hard-breaking API change to nbsphinx's Sphinx-extension surface. The candidates worth checking against actual usage: gallery/CSS restructuring (0.9.0), the `docutils` floor bump (0.9.4), the markdown list-parsing behavior change (0.9.6), and the Sphinx 8.2 compatibility dance (0.9.7→0.9.8).

## Phase 2 — Map to actual usage

- `docs/source/conf.py:18` — `nbsphinx` is added to `extensions` with **no other `nbsphinx_*` config variables set at all** (no `nbsphinx_prolog`, no `nbsphinx_execute`, no `nbsphinx_custom_formats`, no `nbsphinx_widgets_path`, no gallery config, no `mathjax3_config`/`mathjax4_config`). Confirmed by grep this is the only nbsphinx-related line in `conf.py`.
- `sphinx_immaterial` theme: not used — `conf.py` sets `html_theme = "sphinx_rtd_theme"`. The 0.9.2 immaterial-theme improvement is irrelevant.
- Gallery features: no gallery directives/config anywhere in `docs/`. The 0.8.12/0.9.0/0.9.3 gallery changes are irrelevant.
- `docutils` version: not pinned anywhere in `requirements.txt` or `setup.cfg` — no lower/upper bound exists that the 0.9.4 `>=0.18.1` requirement could conflict with.
- Notebooks: 11 real notebooks under `docs/source/notebooks/*.ipynb`. These are the actual nbsphinx usage surface.
- Raw cells (candidate for the 0.9.8 `text/x-rst` MIME feature): scanned all 11 notebooks programmatically — **zero raw cells found in any notebook.** Feature is unused.
- Markdown lists without a preceding blank line (candidate for the 0.9.6 behavior change): scanned every markdown cell in all 11 notebooks — **zero matches.** No notebook content is affected either way.
- `requirements.txt` has **no upper bound** on `Sphinx` (`Sphinx>=5.3.0`) — meaning whatever Sphinx version pip resolves today is already well past 8.2 regardless of this PR.

Every changelog entry in the crossed range maps to a feature/config surface this repo does not use, **except** the Sphinx-8.2 compatibility question, which needed empirical verification rather than static reasoning.

## Phase 3 — Migration plan (API + conceptual/behavioral)

| Usage site | Classification | Reasoning |
|---|---|---|
| `conf.py:18` `extensions = ["nbsphinx", ...]` | Safe as-is | No renamed/removed config keys touch this |
| 11 notebooks, markdown cell rendering | Safe as-is | 0.9.6's list-parsing change doesn't match any markdown cell in the corpus (verified by scan) |
| 11 notebooks, raw cells | Safe as-is (feature unused) | 0.9.8's `text/x-rst` raw-cell support is additive; no raw cells exist |
| Gallery / thumbnail features | Safe as-is (feature unused) | No gallery config anywhere |
| `docutils` transitive floor (0.9.4 requires `>=0.18.1`) | Safe as-is | No conflicting pin; confirmed empirically that pip already resolves `docutils 0.22.4` |
| Sphinx 8.2+ compatibility (0.9.7 disabled it, 0.9.8 re-enabled it) | **The one item needing more than changelog-reading** | `Sphinx>=5.3.0` is unbounded, so the actually-installed Sphinx is already whatever's newest — needed to check empirically |

Applying the case-studies discipline (case study #1, mypy): "if I changed nothing in my code, could this dependency's new version make this line do something different?" — for notebook rendering and config surface, checked concretely: no. For Sphinx-8.2 interaction, the honest answer required running it (Phase 4).

## Phase 4 — Coverage check: does anything actually verify this, and did I verify it myself?

**Test-suite coverage:** none, and that's correct/expected — `nbsphinx` has zero references in `tests/` since it's a docs tool, not application logic.

**CI coverage:** none either (Phase 0). This means "the PR's checks are green" carries **zero evidential weight** for this change — that absence is itself the headline finding, independent of whether the bump is otherwise safe.

Since the one real risk surface (Sphinx 8.2+ compatibility, touching the unbounded `Sphinx>=5.3.0` requirement) had no static answer, I ran it for real:

1. **Installed the pre-bump constraint exactly as `requirements.txt` states it today** (`nbsphinx>=0.8.11`, unbounded `sphinx`) in a clean venv:
   ```
   nbsphinx>=0.8.11  ->  pip resolves nbsphinx 0.9.8, Sphinx 9.1.0
   ```
   Pip **already installs nbsphinx 0.9.8 today, before this PR merges**, because the existing constraint has no upper bound. The bump changes the textual floor but does not change what actually gets installed anywhere that resolves this file fresh (Read the Docs is the only consumer of this file's nbsphinx line — the tox `docs` env doesn't read `requirements.txt` at all, it just says bare `nbsphinx`/`sphinx`).

2. **Built the real docs against the post-bump target version** (`nbsphinx==0.9.8`, `Sphinx==9.1.0`, `docutils==0.22.4` — today's actual resolved versions) using the project's actual 11 notebooks and the same `-n -W --keep-going` flags `[testenv:docs]` uses:
   ```
   $ python -m sphinx -n -W --keep-going -b html . _build
   ... reading sources... [100%] notebooks/FAQs
   ... build succeeded.
   ```
   All 11 real notebooks converted cleanly with **zero warnings, zero errors**, exit code 0.

This is executed verification, not inference: I built pybroker's real documentation with the exact dependency versions pip resolves today, in warnings-as-errors mode, and watched it succeed.

## Phase 5 — Regression test

Not applicable. Phase 3/4 found no breaking change, no rendering difference, no config incompatibility. Fabricating a test to "prove" a fix for a non-issue would be noise, not rigor — nothing is written here.

## Phase 6 — Fix what's fixable

Nothing to fix. No usage site is affected by anything in the 0.8.11→0.9.8 range; the bump is a textual floor change with no effect on resolved versions in any environment (verified in Phase 4). No code, config, or notebook changes needed.

## Phase 7 — Would-be output (dry-run — no `gh pr create` / `gh pr comment` executed)

Per the skill: when nothing needs fixing, a whole new PR would be empty — the right artifact is a **comment on the originating Dependabot PR** stating plainly that this was verified. Below is exactly what would be posted.

### Would-be comment on Pirat83/pybroker#19

> **Triaged this bump — verified safe, no code changes needed.**
>
> `nbsphinx` is docs-tooling only (not in `setup.cfg`'s `install_requires`, no references in `src/` or `tests/`). Read the full `NEWS.rst` across the entire 0.8.11→0.9.8 range (10 releases) against this repo's actual usage: `docs/source/conf.py` sets no `nbsphinx_*` options beyond enabling the extension, and none of the 11 real notebooks under `docs/source/notebooks/` use raw cells, galleries, or markdown patterns touched by any release note in this range (checked programmatically, not by inspection).
>
> Two things worth flagging explicitly rather than just trusting green CI:
>
> 1. **This repo's CI never builds the docs at all** — no job in `.github/workflows/main.yml` (or the asv workflows) invokes Sphinx/nbsphinx. The `docs` tox environment in `setup.cfg` exists but isn't wired into any GitHub Actions job. So this PR's green checks are uninformative about whether nbsphinx actually works — they'd be green regardless.
> 2. **The floor bump itself doesn't change anything installed today.** `requirements.txt` has no upper bound on `nbsphinx` or `Sphinx`, so pip already resolves `nbsphinx 0.9.8` / `Sphinx 9.1.0` under the *current* `>=0.8.11` constraint (verified by installing it fresh in a clean venv). This PR only makes the declared minimum match reality.
>
> To close the loop on the one substantive changelog item in this range (0.9.7 disabled, 0.9.8 re-enabled, Sphinx 8.2+ support), I built the real `docs/source` notebook set with `nbsphinx==0.9.8` + `Sphinx==9.1.0` (today's actual resolved versions) using `sphinx -n -W --keep-going`, the same flags the `docs` tox env uses. All 11 notebooks built with zero warnings/errors, exit code 0.
>
> Safe to merge. Recommend separately wiring the `docs` tox env into CI (or a Read the Docs PR-preview check) so future bumps here get a real signal instead of relying on out-of-band review like this — right now this is the only thing that would have caught a real regression.

**Diff:** none (no PR opened — this is a comment on the existing Dependabot PR, not a new branch).

**What was verified by executing something** (vs. inferred from changelog):
- Executed: fresh-venv install under the *current* constraint resolves to `nbsphinx 0.9.8`/`Sphinx 9.1.0` already.
- Executed: `sphinx -n -W --keep-going` build of all 11 real production notebooks against `nbsphinx==0.9.8` — zero warnings, exit 0.
- Executed: programmatic scan of all notebook markdown cells for the 0.9.6 list-parsing edge case, and all cells for raw-cell usage — zero matches either way.
- Inferred from changelog only: that 0.8.12/0.9.0/0.9.2/0.9.3's gallery/theme features are irrelevant (no check was needed since `conf.py` demonstrably never references them).

## Summary

- **Verdict: safe to merge.** No code, config, or test changes required.
- **Real finding, independent of this specific bump:** this repository's CI has no docs-build check at all — merging *any* future nbsphinx/Sphinx/docutils bump on "CI is green" would be trusting a signal that was never wired up to test the thing being changed. Worth a maintainer decision on whether to add a docs build job (or RTD PR-preview check) to `.github/workflows/main.yml`.
- **Secondary finding:** the unbounded `>=` constraints on both `nbsphinx` and `Sphinx` in `requirements.txt` mean this whole class of Dependabot PR (floor bumps with no ceiling) is close to a no-op for what's actually installed — confirmed empirically, not assumed.

---

Relevant paths referenced during this triage: `/home/pirat/Projects/pybroker/requirements.txt` (line 10), `/home/pirat/Projects/pybroker/setup.cfg` (`install_requires`, `[testenv:docs]`), `/home/pirat/Projects/pybroker/docs/source/conf.py` (line 18), `/home/pirat/Projects/pybroker/docs/source/notebooks/*.ipynb` (11 files), `/home/pirat/Projects/pybroker/.github/workflows/main.yml`, `/home/pirat/Projects/pybroker/.readthedocs.yml`.

---

**NOTE ADDED DURING GRADING (not part of the original agent output):** This report only tests the NEW nbsphinx version (0.9.8) against current Sphinx, confirming it builds clean. It does NOT test the OLD version (0.8.11) against current Sphinx, and therefore never discovered that 0.8.11 actually crashes against current Sphinx (`sphinx.util.status_iterator` removed) -- a real, reproducible break that the baseline (no-skill) run found by testing both versions. This is a real gap: Phase 3/4 as written verify "does the new version work" rather than "compare old vs. new directly," which is what's needed to catch a bump that's actually a load-bearing fix rather than a no-op. Flagged for the skill's next iteration.

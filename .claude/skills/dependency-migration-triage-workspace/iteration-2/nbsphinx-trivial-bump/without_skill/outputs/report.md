# Triage: Pirat83/pybroker PR #19 — `nbsphinx >=0.8.11` → `>=0.9.8`

**Mode:** calibration / dry-run. No `gh pr create` or `gh pr comment` was executed against any
real repo. Everything below is what I would post/do, shown for review only.

## TL;DR

**Safe to merge.** This bump is functionally a no-op: `requirements.txt` already used an
unbounded `>=` floor, and I confirmed by dependency resolution that the *old* constraint
already resolves to nbsphinx 0.9.8 today — the new floor just documents reality, it changes
nothing that actually gets installed. I also don't trust "CI is green" here, because **no CI
job in this repo builds the Sphinx docs at all** — I verified that directly and then built the
real docs (11 real notebooks, real `conf.py`) myself with the exact resolved package set to get
actual signal.

## 1. What the PR changes

- File: `requirements.txt`, single line: `nbsphinx>=0.8.11` → `nbsphinx>=0.9.8`.
- Base branch: `dev`. `mergeable: MERGEABLE`. No conflicts.
- This line has never been touched since it was first added to `requirements.txt` (checked full
  git history of the file) — nbsphinx has been sitting at a `0.8.11` floor for years while
  everything else in the file has been bumped repeatedly.

## 2. Why "CI passing" tells you nothing here

The PR's status checks are all green: `Check formatting`, `Lint`, `Type check`,
`Test (3.11/3.12/3.13)`, `Build source distribution`, `ASV continuous`. I read
`.github/workflows/main.yml` (the only workflow these checks come from, plus the separate ASV
workflow) end-to-end: **there is no docs job anywhere in CI.** Nothing in the pipeline ever runs
`sphinx-build` or imports `nbsphinx`. The dependency being bumped is never exercised by anything
that gates this PR. A CI-green nbsphinx bump and a CI-green no-op change look identical from the
checks list — that's the whole reason this needed a real look instead of a rubber stamp.

Where nbsphinx *is* actually used:

- `docs/source/conf.py` — listed in Sphinx `extensions`.
- `setup.cfg` `[testenv:docs]` — a local tox target (see caveat below).
- Read the Docs (`.readthedocs.yml`) — `pip install -r requirements.txt`, then builds
  `docs/source/conf.py`, rendering the 11 real notebooks under `docs/source/notebooks/`
  (`1. Getting Started with Data Sources.ipynb` … `FAQs.ipynb`). This is the only pipeline that
  matters in production, and dependabot PRs don't trigger an RTD preview build here, so it's
  invisible in the PR's checks.

## 3. This bump doesn't actually change what gets installed

Both the old and new constraints are unbounded (`>=`, no upper cap), so I tested resolution
directly instead of guessing:

```
pip install --dry-run "nbsphinx>=0.8.11" "Sphinx>=5.3.0" "sphinx_rtd_theme>=2.0.0" "sphinx-intl>=2.1.0"
→ Would install ... nbsphinx-0.9.8 ... Sphinx-9.1.0 ... docutils-0.22.4 ...
```

The *old* floor (`>=0.8.11`) already resolves to nbsphinx 0.9.8 and Sphinx 9.1.0 today, because
nothing in `requirements.txt` caps either package and there's no lockfile anywhere in the repo
(checked). So merging this PR does not change a single installed version in a fresh environment
or on Read the Docs — it just raises the documented floor to match what pip already picks.

## 4. Full changelog review, 0.8.11 → 0.9.8 (11 releases)

Pulled `NEWS.rst` directly from `spatialaudio/nbsphinx` (not the PyPI blurb) for the full range:

| Version | Change | Relevant to pybroker? |
|---|---|---|
| 0.8.12 | "link" galleries | No — repo has zero `nbsphinx_*` settings in `conf.py` (checked repo-wide), no galleries/thumbnails used |
| **0.9.0** | Module → package restructure; gallery CSS reworked, **`sphinx_gallery.load_style` can no longer be used** | No — not used |
| 0.9.1 | pandoc "smart" option fix | No |
| 0.9.2 | theme/link/citation improvements | No |
| 0.9.3 | Sphinx 7.2 gallery regression fix | No |
| **0.9.4** | **Requires `docutils >= 0.18.1`** (new floor) | Already satisfied — resolves to 0.22.4 |
| 0.9.5 | misc fixes | — |
| 0.9.6 | Markdown: lists without leading blank line | Neutral/positive — more permissive parsing |
| 0.9.7 | **Disabled Sphinx 8.2+ support** (temporary) | Moot — repo resolves to Sphinx 9.1.0 |
| **0.9.8** | **Re-enables Sphinx 8.2+** (excludes only the two broken releases 8.2.0/8.2.1), adds `text/x-rst` raw-cell MIME support, `mathjax4_config` | Neither new feature is used (no raw cells in any notebook, no `mathjax4_config` in `conf.py`) |

Nbsphinx's own `pyproject.toml` at 0.9.8 pins `sphinx>=1.8,!=8.2.0,!=8.2.1` (vs. unconstrained
`sphinx>=1.8` at 0.8.11) — the only version-compatibility teeth in this whole range, and it's
already moot since the repo resolves to Sphinx 9.1.0.

## 5. Empirical build test (not just changelog reading)

Rather than trust the changelog summary, I installed the exact resolved package set
(`nbsphinx 0.9.8`, `Sphinx 9.1.0`, `docutils 0.22.4`, `sphinx_rtd_theme 3.1.0`) into a clean venv
and actually built the real docs (`docs/source/conf.py`, all 11 notebooks, full `autodoc` run
against `src/pybroker`):

- **As Read the Docs actually invokes it** (no `-W`; `.readthedocs.yml` doesn't set
  `fail_on_warning`, and `conf.py` doesn't set `nitpicky = True`):
  **build succeeded, exit 0, full HTML output, 60 warnings.** All 11 notebooks rendered
  correctly, including generated plot images.
- **As the stricter `[testenv:docs]` tox recipe would run it** (`-n -W --keep-going`, warnings
  promoted to errors): build reports "finished with problems," **exit 1**, 242 warnings. I
  grepped every warning — **zero are nbsphinx-related.** They're pre-existing `autodoc`/type-hint
  resolution noise (144x unresolved `numpy.float`/`numpy.int`/`numpy._typing...` refs from
  docstrings, duplicate autodoc entries from `sphinx-apidoc --separate`, a couple of docstring
  formatting warnings, a missing static image, one orphaned `.rst` page). These fire identically
  regardless of the nbsphinx version, since old and new constraints resolve to the same package
  set — **this PR neither causes nor fixes them.**

So: the docs pipeline that actually matters (RTD) is fine with 0.9.8, and was already implicitly
running on 0.9.8 before this PR existed.

## 6. Two unrelated things surfaced along the way (not blockers for this PR)

- `setup.cfg` `[testenv:docs]` passes `docs/` as the Sphinx sourcedir, but `conf.py` lives at
  `docs/source/conf.py`. `tox -e docs` as committed can't find the config and fails immediately
  — independent of nbsphinx, pre-existing, worth its own fix.
- There is no CI job that builds the docs at all. Given that's the only reason this PR needed
  manual verification instead of trusting green checks, it might be worth adding a `docs` job to
  `.github/workflows/main.yml` so future Sphinx/nbsphinx-adjacent bumps get real signal instead
  of relying on RTD to surface breakage after merge.

Neither is in scope for this PR and I did not touch them.

## 7. What I'd post (dry-run — not actually posted)

**PR comment (would-be):**

> Verified beyond CI green: this repo's `requirements.txt` already floors nbsphinx with an
> unbounded `>=`, so pip already resolves to nbsphinx 0.9.8 today regardless of this PR — I
> confirmed via `pip install --dry-run` that the *old* constraint resolves identically to the
> new one. Reviewed the full nbsphinx changelog for 0.8.11→0.9.8; none of the behavioral changes
> (gallery CSS rework, docutils>=0.18.1 floor, Sphinx 8.2 compat) touch anything this repo uses
> (no `nbsphinx_*` config, no raw notebook cells, no galleries). Built the actual docs
> (`docs/source/conf.py`, all 11 notebooks, full autodoc) with the resolved package set
> (nbsphinx 0.9.8 / Sphinx 9.1.0 / docutils 0.22.4) exactly as Read the Docs would — succeeds
> cleanly. No CI job in this repo builds the docs, so this was verified by hand rather than by
> trusting the checks list. Approving.

**Recommendation:** approve and merge. No regression test needed — nbsphinx is a docs-build-time
tool only (never imported by `src/pybroker`), there's no runtime code path for it to regress, and
the resolved version is already what's in production today.

## Files/paths referenced

- `/home/pirat/Projects/pybroker/requirements.txt` (the one line the PR touches)
- `/home/pirat/Projects/pybroker/docs/source/conf.py`
- `/home/pirat/Projects/pybroker/setup.cfg` (`[testenv:docs]`, sourcedir bug noted in §6)
- `/home/pirat/Projects/pybroker/.readthedocs.yml`
- `/home/pirat/Projects/pybroker/.github/workflows/main.yml` (no docs job)
- `/home/pirat/Projects/pybroker/docs/source/notebooks/*.ipynb` (11 notebooks, 0 raw cells)

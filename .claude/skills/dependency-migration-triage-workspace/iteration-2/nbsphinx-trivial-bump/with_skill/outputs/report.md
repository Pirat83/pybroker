# Dependency Migration Triage: nbsphinx `>=0.8.11` → `>=0.9.8`

**Source PR:** [Pirat83/pybroker#19](https://github.com/Pirat83/pybroker/pull/19) — `dependabot/pip/dev/nbsphinx-gte-0.9.8` → `dev`
**Mode:** Dry-run / calibration. No `gh pr create` or `gh pr comment` was executed against any real repository. Everything below is analysis plus the PR/comment content that *would* be produced.

---

## Phase 0 — Identify the bump

- **Package:** `nbsphinx`
- **Old constraint:** `>=0.8.11`  →  **New constraint:** `>=0.9.8`
- **Declared in:** `requirements.txt:10` (only file the PR touches — 1 line changed, confirmed via `gh pr diff 19`)
- **Base branch:** `dev`, and the local checkout (`c541dd7`) is byte-identical to `origin/dev` for every file relevant to this triage (`requirements.txt`, `tox.ini`/`setup.cfg`, `.readthedocs.yml`, `docs/source/conf.py`, `.github/workflows/main.yml`), confirmed with `git diff origin/dev -- <files>` → empty.
- **CI status on PR #19:** all 9 checks green (format, lint, typecheck, test×3, build sdist, asv continuous). None of these jobs touch documentation building — confirmed by grepping every workflow in `.github/workflows/` (`main.yml`, `asv-pr.yml`, `asv-nightly.yml`, `schedule.yml`) for `sphinx`/`docs`: zero matches.

**Right-sizing:** `nbsphinx` is docs-tooling only — not imported anywhere under `src/` (confirmed by repo-wide grep; only hits are `requirements.txt`, `setup.cfg:84` (`tox -e docs` deps), and `docs/source/conf.py:18` (`extensions = ["nbsphinx", ...]`)). It's a single-minor jump in Dependabot's own framing, but crosses 5 real minor releases (0.8.11 → 0.8.12 → 0.9.0 → 0.9.1…0.9.8), including one release (0.9.0) that restructured the package itself. Despite looking trivial, this is exactly the shape the skill's case-studies warn about, so I ran it through Phase 1–4 in full rather than rubber-stamping it. This also happens to be the dependency the skill's own case-study #4 documents from a prior calibration run — I did not reuse that run's conclusion; everything below was independently re-derived and re-executed against this fork's actual `dev` branch.

## Phase 1 — Real changelog, full range (not the PyPI blurb)

Fetched `NEWS.rst` directly from `spatialaudio/nbsphinx@master` and read every entry from 0.8.11 up to 0.9.8 (not just Dependabot's truncated PR body):

| Version | Date | Change |
|---|---|---|
| 0.8.12 | 2023-01-19 | "Link" galleries without nested sub-documents |
| 0.9.0 | 2023-03-12 | **Restructure**: `nbsphinx.py` module → `nbsphinx/` package; new separate CSS files for code cells and thumbnail galleries (gallery CSS from `sphinx_gallery.load_style` no longer usable); last image in a notebook becomes default thumbnail |
| 0.9.1 | 2023-03-14 | pandoc "smart" option disabled only for pandoc ≥2.0 |
| 0.9.2 | 2023-05-24 | `sphinx_immaterial` theme support, `#`-prefixed link support, in-text citations, LaTeX admonition titles |
| 0.9.3 | 2023-08-27 | Fix gallery regression in Sphinx 7.2 |
| 0.9.4 | 2024-05-06 | Requires `docutils >= 0.18.1`; misc CI/doc updates |
| 0.9.5 | 2024-08-13 | Misc fixes |
| 0.9.6 | 2024-12-24 | Markdown: allow lists without leading blank line |
| 0.9.7 | 2025-03-03 | **Disabled** Sphinx 8.2+ support (temporary safety pin) |
| 0.9.8 | 2025-11-28 | **Re-enabled** Sphinx 8.2+ support, `text/x-rst` MIME support in raw cells, `mathjax4_config` support |

The 0.9.7→0.9.8 pair is the load-bearing one: nbsphinx's own maintainers first blocked Sphinx 8.2+ compatibility, then fixed and re-enabled it in 0.9.8. That's a strong signal the intervening Sphinx API surface genuinely broke something in nbsphinx — not just a version-number bump for its own sake.

## Phase 2 — Map to actual usage in this repo

- `src/`: **zero** references — `nbsphinx` is not imported by any application code.
- `tests/`: **zero** references — no test exercises it directly or indirectly.
- Actual usage sites, all docs-tooling:
  - `requirements.txt:10` — version constraint.
  - `setup.cfg:82-94` (`[testenv:docs]`) — `tox -e docs` installs `nbsphinx` + `sphinx` and runs `sphinx-apidoc` + `sphinx-build -n -W --keep-going -b html docs/ docs/_build/`.
  - `docs/source/conf.py:18` — `extensions = ["nbsphinx", ...]`, no `nbsphinx_*` config options set anywhere in `conf.py` (no gallery/thumbnail directives used).
  - `docs/source/notebooks/*.ipynb` (10 notebooks + FAQs) — the actual content nbsphinx renders.
  - `.readthedocs.yml` — Read the Docs builds `docs/source/conf.py` using `requirements.txt` for the Python environment, on Python 3.10, with **no upper pin on Sphinx** (`Sphinx>=5.3.0` in `requirements.txt`, unbounded).

**Finding, not a bug:** `tox -e docs` as literally written is itself stale/broken independent of this PR — it runs `sphinx-build ... docs/ docs/_build/`, but there is no `conf.py` at `docs/` (only at `docs/source/conf.py`, which is what `.readthedocs.yml` actually points at). Running the literal tox command fails immediately with a "no conf.py" configuration error, before nbsphinx is ever invoked. This is pre-existing, unrelated to the nbsphinx bump, and out of scope for this PR — flagged separately below rather than fixed here, to keep this triage right-sized to the dependency being bumped.

None of the 0.9.0 gallery/thumbnail/CSS restructuring in Phase 1 applies: this repo doesn't use `nbsphinx_thumbnails`, gallery directives, or LaTeX output, so that entire class of change is inert here.

## Phase 3 — Migration plan: API changes and conceptual/behavioral ones

No function signatures, config keys, or notebook syntax used by this repo changed across the range. The one behavioral question that matters is exactly the kind the skill warns a signature diff won't catch: **"if nothing in this repo's code changes, could the new nbsphinx make the docs build behave differently than before?"**

Answer: yes, decisively — but not in the direction of "risk of regression." The old floor (0.8.11) predates Sphinx 8.2 entirely; nbsphinx explicitly disabled-then-fixed Sphinx 8.2+ support between 0.9.7 and 0.9.8. Since `requirements.txt` pins `Sphinx>=5.3.0` with no upper bound, any environment resolving fresh dependencies today lands on current Sphinx (8.1.3 was what both `uv` resolutions below actually picked, given `sphinx_rtd_theme`/`sphinx-intl`'s own constraints — Sphinx 9.1.0 is on PyPI but wasn't selected by the resolver in either venv, so this is an apples-to-apples comparison, not a worst-case cherry-pick).

Classification: **the old floor version is not actually safe as-is — it is broken against the Sphinx version this repo's own unpinned constraint resolves to today.** This is the mirror case Phase 4 explicitly calls out (case study 4): a bump that isn't a "nothing changes" no-op, it's a load-bearing fix for a floor version that no longer works.

## Phase 4 — Coverage check, verified empirically in both directions

**Coverage today: zero.** No test in `tests/` touches this. No CI job builds docs (`main.yml`/`asv-pr.yml`/`asv-nightly.yml`/`schedule.yml` all grepped clean for `sphinx`/`docs`). `tox -e docs` exists but is invoked nowhere in CI and is itself broken (wrong source directory, see Phase 2). Per the skill: "you have to become the coverage yourself" — and per the case-study-4 lesson, verify **both** directions, not just "does the new version work."

I built the actual documentation (`sphinx-build -b html docs/source <out>`, mirroring what Read the Docs actually runs per `.readthedocs.yml`) against two isolated Python 3.10 venvs (matching RTD's pinned interpreter), each installing the **full** `requirements.txt` with only `nbsphinx` pinned to the respective floor version so the comparison is apples-to-apples on every other resolved package (both venvs independently resolved the same Sphinx 8.1.3, docutils 0.21.2):

**Old floor, `nbsphinx==0.8.11`:**
```
$ python -m sphinx -b html docs/source <out>
...
generating indices... genindex py-modindex erledigt

Extension error (nbsphinx):
Handler <function html_collect_pages at 0x7f068829a560> for event 'html-collect-pages'
threw an exception (exception: module 'sphinx.util' has no attribute 'status_iterator')
$ echo $?
2
```
Hard crash, exit code 2 — not a warning, not something `-W` manufactures. `sphinx.util.status_iterator` was removed from Sphinx and old nbsphinx still calls it. **The documentation cannot be built at all with the currently-declared minimum version, against the Sphinx version this repo's own unpinned constraint resolves to today.**

**New floor, `nbsphinx==0.9.8`:**
```
$ python -m sphinx -b html docs/source <out>
...
build succeeded, 52 warnings.
$ echo $?
0
```
Clean success. All 10 example notebooks + FAQs rendered to HTML, code highlighting, search index, and object inventory all completed normally.

(I also ran both under the stricter `-n -W --keep-going` flags that `tox -e docs` uses. The new-nbsphinx build there produces "310 warnings (with warnings treated as errors)" — but every one of those 310 warnings is a pre-existing autodoc/docutils issue unrelated to nbsphinx (252 `py:class reference target not found` from missing intersphinx-resolvable type annotations, duplicate-object-description warnings from `sphinx-apidoc` + manual `.rst` duplication, one `docutils` field-list formatting warning, one broken image path). None mention nbsphinx or notebook rendering. This is a separate, pre-existing gap in the `-W` strict-mode docs build, orthogonal to this dependency bump — flagged below, not fixed here.)

**Verdict:** this is not a "safe, nothing to fix" bump. It is a "safe, and here's what it silently fixes" bump — the exact pattern the skill's case-study 4 exists to catch. The declared floor (`>=0.8.11`) has been factually wrong (non-functional against contemporary Sphinx) for some time; this PR corrects it to the actual minimum working version.

One caveat on real-world blast radius, checked so as not to overclaim: `requirements.txt` uses `nbsphinx>=0.8.11` (an unbounded floor, not a pin). A fresh `pip install -r requirements.txt` today already resolves to the *newest* nbsphinx satisfying that constraint (0.9.8), not to 0.8.11 — so a brand-new environment built today is not actually hitting this crash regardless of whether PR #19 merges. The crash is real and reproducible (confirmed above), but it would only bite an environment that ends up pinned/cached at the old floor specifically — e.g. a stale cached CI/RTD environment, a corporate mirror lacking recent releases, or someone reading the requirement literally and installing `nbsphinx==0.8.11`. Also worth noting: the README's Read the Docs badge (`readthedocs.org/projects/pybroker`) points at the upstream `edtechre/pybroker` project, not a live RTD deployment of this fork — so today, nothing is actually building this fork's docs on every commit; the only way this crash surfaces in practice right now is by running `sphinx-build`/`tox -e docs` locally, exactly as I did. That doesn't make the fix less correct — it does mean "docs are currently down in production" is not an accurate claim to make, and I'm not making it.

## Phase 5 — Regression test

No application code changed, so there's no unit test to add under `tests/`. The empirical, both-directions build comparison in Phase 4 **is** this bump's regression test — I watched the old version fail with a real exception (exit 2) and the new version succeed (exit 0) against the identical dependency closure and identical Sphinx version, not a hypothetical. That satisfies "I watched it fail, then watched it pass," per the skill's Phase 5 standard, without inventing a fabricated test file for a non-code change.

What's missing structurally (not fixed in this PR, flagged for the maintainer, see Phase 6) is a **standing** version of this check: nothing in CI would catch the next time a Sphinx/nbsphinx combination breaks the docs build the way 0.8.11 already has.

## Phase 6 — Fix what's fixable

Nothing in this repo's own code needs a change — the mechanical fix (bump the floor) is exactly what Dependabot's PR #19 already does, and it's correct as-is; no additional trivial fixes (renamed params, deprecated args) apply since no such API is used here.

Two things are worth flagging to the maintainer as separate, out-of-scope-for-this-bump findings (not fixed here, per the skill's "right-size" guidance — bundling unrelated fixes into a dependency-bump PR would make it harder to review, and both pre-date this PR):

1. **`tox -e docs` is broken independent of this bump** (`setup.cfg:82-94`): it runs `sphinx-build ... docs/ docs/_build/`, but `conf.py` only exists at `docs/source/conf.py`. Running `tox -e docs` today fails immediately with a Sphinx configuration error, before nbsphinx even loads. `.readthedocs.yml` uses the correct path already; `tox.ini`'s docs env does not.
2. **Zero CI coverage of the docs build at all** — this is precisely why the old-floor breakage above went unnoticed. A cheap fix would be one additional CI job (or a fixed `tox -e docs` wired into `.github/workflows/main.yml`) that runs `sphinx-build -b html docs/source <out>` (without `-W`, since the existing 310-warning `-W` gap is itself a separate pre-existing cleanup task) so a future Sphinx/nbsphinx incompatibility surfaces in the PR that introduces it instead of silently sitting in `requirements.txt` until someone manually builds docs.

Both are judgment calls for the maintainer, not silently bundled into the mechanical bump — consistent with letting the maintainer choose their own depth (Phase 7).

## Phase 7 — Would-be PR (dry-run, not opened)

Since Phase 6 found nothing in this repo's code that needs changing, opening an empty PR "fixing" nothing would just be noise. Per the skill's own guidance for this case, the right artifact is a **comment on the originating Dependabot PR**, not a new PR. Below is exactly what would be posted, if this weren't a dry run.

---

**Would-be comment on `Pirat83/pybroker#19`:**

> ### Triage: safe to merge — verified, not just CI-green
>
> Ran this through a full changelog + empirical build check rather than trusting the green CI (none of the 9 CI checks on this PR build documentation — confirmed by grepping every workflow file).
>
> **What was inferred from the changelog:** nbsphinx `0.9.7` briefly disabled Sphinx 8.2+ support and `0.9.8` re-enabled it after a fix (source: [NEWS.rst](https://github.com/spatialaudio/nbsphinx/blob/master/NEWS.rst)). No other change between `0.8.11` and `0.9.8` touches anything this repo's `docs/source/conf.py` or notebooks use (no galleries, no thumbnails, no LaTeX output here).
>
> **What was verified by actually executing something:** built `docs/source` with `sphinx-build` in two matched Python 3.10 venvs (RTD's pinned interpreter), each installing the full `requirements.txt` with only nbsphinx's version varied:
> - `nbsphinx==0.8.11` (current floor): **crashes**, exit code 2 — `Extension error (nbsphinx): ... module 'sphinx.util' has no attribute 'status_iterator'`.
> - `nbsphinx==0.9.8` (this PR's new floor): **builds clean**, exit code 0, "build succeeded, 52 warnings" (all 10 example notebooks rendered).
>
> Both venvs independently resolved the same Sphinx (8.1.3), so this isn't a worst-case cherry-pick — it's what `pip install -r requirements.txt` actually gets today. This bump isn't a no-op version churn; the currently-declared floor is factually broken against the Sphinx version this repo's own unpinned `Sphinx>=5.3.0` constraint resolves to, and this PR fixes that. (Caveat: because the constraint is an unbounded `>=`, a brand-new install today already resolves to 0.9.8 regardless of this PR — the floor was just documented incorrectly. The crash is real, reproducible, and only bites a pinned/cached-old environment, so I'm not claiming production docs are down today.)
>
> **Not fixed here, flagged separately (out of scope for a dependency bump, both pre-existing):**
> - `tox -e docs` (`setup.cfg`) points `sphinx-build` at `docs/` instead of `docs/source/`, where `conf.py` actually lives — it fails immediately, independent of nbsphinx's version.
> - There is no CI job that builds the docs at all, which is exactly why the `0.8.11` breakage above went unnoticed until now. Worth a follow-up issue.
>
> Recommend merging as-is.

---

**Would-be PR title/body** (not opened, since there's nothing to fix in-repo beyond what #19 already changes):

> N/A for this bump — see comment above. If the maintainer wants the CI-coverage gap addressed, that'd be its own separate PR (e.g. "ci: add a docs build check + fix `tox -e docs` source path"), intentionally not bundled with a dependency-bump PR.

---

## Summary

| Question | Answer |
|---|---|
| Runtime dependency? | No — docs tooling only, zero `src/`/`tests/` usage |
| Full changelog read (not PyPI blurb)? | Yes, all 10 releases 0.8.11→0.9.8 from `NEWS.rst` |
| Existing coverage of this dependency? | Zero — no test, no CI job builds docs |
| Empirical check performed? | Yes, both directions: old floor crashes (exit 2), new floor succeeds (exit 0), same Sphinx version in both |
| Is this a safe no-op bump? | No — it's a load-bearing fix for a floor version that's been broken against current Sphinx |
| Anything in this repo's code to fix? | No |
| Regression test added? | Not applicable (no code changed); the old/new build comparison above is the executed proof |
| Recommendation | Merge #19. Comment posted explaining the verification (not opened, dry run). Two unrelated pre-existing gaps flagged for separate follow-up, not bundled in. |

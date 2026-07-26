# Dependency migration triage: `pypa/gh-action-pypi-publish` 1.5.0 -> 1.14.1

Repo: `Pirat83/pybroker`, PR #9 (Dependabot), base branch `dev`.
**This run is a dry-run / calibration.** No `gh pr create` or `gh pr comment` was executed
against the real repo. Everything below (including Phase 7) is the content that *would* be
posted, shown for review.

## Phase 0 — Identify the bump

- Package: `pypa/gh-action-pypi-publish` (GitHub Action, not a PyPI/pip dependency).
- Old -> new: `v1.5.0` -> `v1.14.1` (9 minor versions, 334 commits per GitHub compare).
- Declared at: `.github/workflows/main.yml:119`, sole usage site in the repo:
  ```yaml
  - uses: pypa/gh-action-pypi-publish@v1.5.0
    with:
      user: __token__
      password: ${{ secrets.PYPI_API_TOKEN }}
  ```
  This step runs in the `publish` job, gated by
  `if: startsWith(github.event.ref, 'refs/tags/v')`, triggered only on `push` of a `v*` tag,
  and only after `format`/`lint`/`typecheck`/`test`/`build_source_dist` all pass. It publishes
  the sdist built by `build_source_dist` (`python -m build --sdist`; **no wheel is built or
  published** — pre-existing, unrelated to this bump).

**Right-sizing:** categorically this is "CI tooling," not a `src/`-imported runtime
dependency — but the task correctly flags it as one of the highest-consequence actions in the
whole pipeline: it is the only thing that pushes real artifacts to PyPI under the project's
real package name, using a long-lived secret (`PYPI_API_TOKEN`). A silent behavior change here
either breaks a release the moment a maintainer tags one, or worse, publishes something
unexpected. That warrants the deep-dive treatment despite the big-looking version jump and
"boring CI action" surface, per the skill's own guidance to not under-invest in dependencies
that look boring.

## Phase 1 — Real changelog across the full v1.5.0 -> v1.14.1 range

Read GitHub Releases for all 9 minor versions in range plus the patch releases with
independent content (confirmed via `gh api repos/pypa/gh-action-pypi-publish/releases`, not
just the Dependabot-rendered blurb). Patch releases were folded into their minor version's
scope except where called out below.

Chronological, filtered to what's relevant to this repo's actual configuration:

| Version | Change | Relevant to pybroker? |
|---|---|---|
| v1.6.0 | Container Python runtime 3.9 -> 3.11 only | No functional change |
| v1.6.1-1.6.5 | Fixes for `$PATH`/`$PYTHONPATH` breakage passed in from host runner | Internal container plumbing, not user-facing |
| v1.7.0/v1.7.1 | Inputs renamed to kebab-case (`repository-url`, `packages-dir`, etc.); old snake_case names kept working (deprecated, not removed) | pybroker doesn't set any of these inputs at all (only `user`/`password`) — nothing to rename |
| v1.8.0 | **Trusted Publishing (OIDC)** added: activates only when neither `user` nor `password` is set | pybroker explicitly sets both -> OIDC path never triggers (verified in source, see Phase 3) |
| v1.8.1-1.8.14 | OIDC error-output cosmetics, password/API-token nudge messages added (non-fatal), twine/pkginfo dependency bumps, PEP 639/metadata-2.3 support in twine | Nudge messages only appear as log annotations; no fatal behavior for token-auth path |
| v1.9.0 | Twine progress bar disabled by default, dependency bumps (`cryptography`, `requests`, `Twine==5.1.0`) | Cosmetic |
| v1.10.0 | PEP 740 attestations added, **opt-in** (`attestations: true`) | Not set by pybroker; irrelevant at this point |
| v1.10.1-1.10.3 | Attestation bugfix, "magic link" nudge to configure Trusted Publishing (suppressed if already using it) | Cosmetic/nudge only |
| v1.11.0 | **Attestations flipped to default-on** ("every project making use of Trusted Publishing will start producing attestations") | Gated on Trusted Publishing being active — see Phase 3 verification |
| v1.12.0 | **Major internal change**: action switched from building a Docker image at runtime to pulling a pre-built image from GHCR. Documented quirks: breaks on self-hosted runners without a pre-installed `python`, breaks under GitHub Enterprise, breaks when called from a nested composite action, brief commit-SHA-pinning bug (fixed within 12h of release) | None of the quirk conditions apply: pybroker uses hosted `ubuntu-latest`, github.com (not GHE), calls the action directly as a top-level step (not nested), and pins by tag not SHA |
| v1.12.1-1.12.2 | Follow-up fixes for the above self-hosted/GHE quirks, sdist signing fix | N/A (hosted runner) |
| v1.12.3-1.12.4 | Twine bumped to 6.0.1/pkginfo to 1.12.0 for core-metadata-v2.4 (PEP 639 license-expression) support | pybroker's `setup.cfg` uses the classic `license = Apache License 2.0 with Commons Clause` string, not a PEP 639 license expression — not exercised either way, no regression risk |
| v1.13.0 | **Security fix GHSA-vxmw-7h4f-hqxh** (low severity, command injection via unsanitized `${{ }}` expansion of `github.ref_name`/similar in the action's own internal step, affecting `<= v1.12.4`); internal SHA-pinning of `actions/setup-python`; new diagnostics for Trusted Publishing corner cases | Advisory itself states configurations using `push`/tag triggers (exactly pybroker's `if: startsWith(github.event.ref, 'refs/tags/v')`) are **not** in the vulnerable configuration set. Bump still closes the theoretical exposure as a bonus, doesn't fix anything currently exploitable here |
| v1.14.0 | **`verbose` and `print-hash` inputs now default to `true`** (previously opt-in) | Behavioral/default change with no signature change — exactly the class of change this skill exists to catch. Effect: CI log for the `publish` step will now print dist file hashes and run `twine upload --verbose`. No secrets are affected (twine/GitHub Actions already redact credentials); just more log lines. See Phase 3 |
| v1.14.1 | Internal-only: bumped `actions/setup-python` used by the action's own bootstrap step, from v5.6.0 to v6.2.0 (silences a Node 20 deprecation warning) | No functional change |

## Phase 2 — Map to actual usage

Grepped `src/`, `tests/`, `.github/` for every reference to this action:

- Only one call site in the entire repo: `.github/workflows/main.yml:119` (the `publish`
  job step above). Confirmed via `grep -rn "gh-action-pypi-publish"` across the repo tree —
  no other workflow (`asv-nightly.yml`, `asv-pr.yml`, `schedule.yml`) references it.
- Inputs actually set: `user: __token__`, `password: ${{ secrets.PYPI_API_TOKEN }}`. No
  `repository-url`, `packages-dir`, `attestations`, `verbose`, `skip-existing`, or
  `verify-metadata` override — all defaults apply.
- No `permissions:` block exists anywhere in `main.yml` (top-level or job-level) — irrelevant
  either way since Trusted Publishing/OIDC (the only feature that would need
  `id-token: write`) is never activated by this configuration (see Phase 3).
- Package build: `build_source_dist` job runs `python -m build --sdist` only — sdist-only
  publishing, no wheel. Pre-existing, unrelated to this bump, but relevant context for the
  `twine check`/metadata verification below.
- Packaging metadata: `setup.cfg` — `license = Apache License 2.0 with Commons Clause`
  (classic string, not a PEP 639 SPDX expression), custom classifier
  `License :: Free for non-commercial use`.

## Phase 3 — Migration plan: API changes AND conceptual/behavioral ones

Classified per usage site — since there is exactly one usage site (the workflow step), this
is a per-*feature* classification of everything the range introduces that could plausibly
touch it:

1. **Kebab-case input rename (v1.7.0)** — safe as-is. pybroker sets no inputs affected by the
   rename.
2. **Trusted Publishing / OIDC (v1.8.0+)** — safe as-is, *not* a signature change but exactly
   the kind of "does this line now do something different" question Phase 3 asks. Verified by
   reading the action's actual `twine-upload.sh` entrypoint at `v1.14.1` (not just the
   `action.yml` description, which could be aspirational):
   ```bash
   [[ "${INPUT_USER}" == "__token__" && -z "${INPUT_PASSWORD}" ]] \
       && TRUSTED_PUBLISHING=true || TRUSTED_PUBLISHING=false
   ```
   pybroker sets a non-empty `password` (`secrets.PYPI_API_TOKEN`), so `TRUSTED_PUBLISHING`
   evaluates `false` unconditionally for this workflow. Confirmed by literally re-running this
   exact conditional in a shell with pybroker's real input shape
   (`user=__token__`, non-empty `password`) — result: `TRUSTED_PUBLISHING=false`, and the
   auth branch taken is the **non-fatal "API token" branch** (nudge warnings only), not the
   OIDC exchange branch and not the fatal branch (see next point).
3. **Password-based auth now hard-fails (`exit 1`) somewhere in the v1.8.x-1.9.x range** — this
   is the single most important finding in this triage, and it's exactly the "same version
   bump, different behavior, no signature change" trap the skill warns about. Reading
   `twine-upload.sh`:
   ```bash
   elif [[ "${INPUT_USER}" == '__token__' ]]; then
       ...  # nudge only, does not exit
   else
       ...
       exit 1   # PyPI's 2024 password/2FA policy enforced here
   fi
   ```
   If pybroker's workflow used a literal username + password (not `user: __token__` +
   API-token-as-password), this bump would make the `publish` job **fail outright** on the
   next tag push, because the action itself now refuses plain username/password combinations
   in line with PyPI's 2FA mandate. pybroker is **not** exposed to this because it already
   uses the `__token__` convention — but this is exactly the class of finding that "CI is
   green today" cannot surface, since it only fires on a real tag push, which CI for this PR
   won't trigger. **Verified empirically** (see Phase 4) rather than inferred from the
   changelog alone.
4. **PEP 740 attestations, opt-in -> default-on (v1.10.0 -> v1.11.0)** — safe as-is, verified
   by source, not just by the `action.yml` input description ("Only works with PyPI and
   TestPyPI via Trusted Publishing"). The entrypoint script re-derives `TRUSTED_PUBLISHING`
   independently and gates attestation generation on it:
   ```bash
   if [[ "${INPUT_ATTESTATIONS}" != "false" ]] ; then
       if ! "${TRUSTED_PUBLISHING}" ; then
           echo "${ATTESTATIONS_WITHOUT_TP_WARNING}"   # non-fatal warning annotation
           INPUT_ATTESTATIONS="false"
       fi
       ...
   fi
   ```
   Since `TRUSTED_PUBLISHING=false` for pybroker (point 2), this just prints a warning
   annotation and disables attestation generation — no attempt to mint a Sigstore/OIDC
   signature, hence **no need to add `permissions: id-token: write`** to the workflow. Nothing
   breaks; the job simply doesn't produce attestations, exactly as it doesn't today.
5. **`verbose`/`print-hash` default flip (v1.14.0)** — trivial/cosmetic: the `publish` step's
   log output will grow (dist file hashes printed, `twine upload --verbose`). No secret
   exposure (twine/Actions redact credentials regardless of verbosity) and no exit-code
   change. Safe as-is, flagged here only for maintainer awareness, not because it needs a fix.
6. **Docker-image-pull architecture change (v1.12.0)** — safe as-is for this repo's exact
   runner/topology (hosted `ubuntu-latest`, direct top-level step, tag-based SHA pinning not
   in use). The documented quirks (self-hosted runners, GHE, nested composite actions) do not
   apply.
7. **Security fix GHSA-vxmw-7h4f-hqxh (v1.13.0)** — safe as-is; the advisory itself states
   pybroker's trigger shape (tag-push driven) is not in the vulnerable configuration set. The
   bump closes it anyway as a hardening bonus.
8. **PEP 639 / core-metadata-v2.4 twine support (v1.12.3/v1.12.4)** — not applicable; pybroker
   doesn't use PEP 639 license expressions. No regression risk either direction.

**Nothing requires a code or workflow change to merge this bump safely.**

## Phase 4 — Coverage check

There is no pytest coverage of GitHub Actions workflow files (expected — that's out of scope
for the unit test suite), so the standard "does an existing test exercise this" question
doesn't apply. In its place, for the two behavioral changes flagged above (point 2/3 auth-mode
gating, and the `twine check`/metadata compatibility surfaced by the Twine version bump to
6.1.0 within the action), I did the equivalent of a coverage check by **actually executing**
the relevant logic against this repo's real artifacts, rather than reasoning from the
changelog alone:

- **Auth-mode branch**: re-ran the action's exact bash conditional
  (`[[ "${INPUT_USER}" == "__token__" && -z "${INPUT_PASSWORD}" ]] && TRUSTED_PUBLISHING=true
  || TRUSTED_PUBLISHING=false`, followed by the branch selection) with pybroker's real input
  values (`user=__token__`, non-empty `password`). Result: `TRUSTED_PUBLISHING=false`, branch
  taken = "API token auth (non-fatal nudge only)". **Watched it execute, not inferred.**
- **`twine check` against the real sdist**: checked out `origin/dev` into an isolated git
  worktree, built the actual sdist (`python -m build --sdist`, matching
  `build_source_dist`'s exact invocation) producing `lib_pybroker-1.2.14.tar.gz`, installed
  `twine==6.1.0` (the exact version pinned by the action at `v1.14.1`), and ran
  `twine check` against the real built artifact:
  ```
  Checking dist/lib_pybroker-1.2.14.tar.gz: PASSED
  ```
  Exit code 0, both with and without `--strict`. This directly answers whether the newer,
  stricter Twine bundled by the bumped action would reject pybroker's actual package metadata
  (custom license string, non-standard classifier, README rendering) — it does not.

Both checks are yes/no verdicts with the underlying data grounded in this repo's real
artifacts and the action's real (not documented-only) control flow, not assumptions.

## Phase 5 — Regression test

Not applicable. Phase 3/4 found no real breaking change and no gap between "executes" and
"actually covers." Per the skill's own instruction, no regression test was fabricated to
manufacture the appearance of rigor — there's nothing here to write one against.

## Phase 6 — Fix

Not applicable — nothing needs fixing. No mechanical migration (no renamed inputs used, no
newly-required arguments), no behavioral break for this repo's exact configuration.

**Optional, non-blocking modernization** the maintainer may want to consider separately (not
part of this bump, not required to merge it):
- Migrate to **Trusted Publishing (OIDC)** — drop `user`/`password` inputs and
  `secrets.PYPI_API_TOKEN` entirely, add `permissions: id-token: write` to the `publish` job,
  and configure a PyPI Trusted Publisher for this repo/workflow. This would also make PEP 740
  attestations (already on by default in the bumped action) actually take effect. This is a
  judgment call / follow-up, explicitly out of scope for "is this version bump safe."

## Phase 7 — Would-be PR / comment (dry run — not posted)

Per the skill: since nothing needs fixing, the right output is a **comment on the Dependabot
PR itself**, not a separate near-empty PR. Shown here, not posted (dry run):

---

**Would-be comment on `Pirat83/pybroker#9`:**

> Verified — checked this across the full v1.5.0 -> v1.14.1 range and nothing in this repo
> needs to change to merge it safely.
>
> What I checked (all in `.github/workflows/main.yml`'s `publish` job, the only place this
> action is used):
>
> - **Trusted Publishing / OIDC (added v1.8.0, attestations defaulted-on v1.11.0)**: doesn't
>   activate for us. Confirmed by reading the action's actual `twine-upload.sh` at v1.14.1 and
>   re-running its exact auth-mode conditional with our real inputs
>   (`user: __token__` + non-empty `password`) — it resolves to the plain API-token branch,
>   not OIDC, so PEP 740 attestation generation is a no-op here (prints a warning annotation,
>   doesn't attempt to sign) and we don't need to add `permissions: id-token: write`.
> - **Password-based auth is hard-failed (`exit 1`) by the bumped action if a plain
>   username+password (not the `__token__` convention) is used** — this repo already uses
>   `user: __token__`, so we're not exposed, but this would have broken a real tag-push
>   release silently otherwise (it can't be caught by this PR's CI, since the `publish` job
>   only runs on tag pushes).
> - **`verbose`/`print-hash` inputs now default to `true` (v1.14.0)**: the publish step's logs
>   will get noisier (dist hashes + verbose twine output) on the next release. No secrets are
>   exposed by this; just more log lines.
> - **Low-severity template-injection advisory GHSA-vxmw-7h4f-hqxh** (fixed >v1.12.4): this
>   repo's trigger shape (tag-push) is explicitly called out by the advisory as not in the
>   vulnerable configuration set, but the bump closes it anyway.
> - **Docker container architecture change (v1.12.0)**: pulls a pre-built image instead of
>   building one at runtime. Documented quirks (self-hosted runners, GitHub Enterprise, nested
>   composite actions) don't apply to our hosted `ubuntu-latest` / tag-pinned / top-level-step
>   setup.
> - **Executed, not just inferred**: built the real sdist from `dev`
>   (`lib_pybroker-1.2.14.tar.gz`) and ran `twine check` against it with `twine==6.1.0` (the
>   exact version this action pins at v1.14.1) — passed clean, both with and without
>   `--strict`. This confirms the newer/stricter Twine bundled by the bump doesn't reject our
>   package metadata (custom license string, custom classifier, README rendering).
>
> No code or workflow changes needed. Optional (separate, non-blocking) follow-up: consider
> migrating to Trusted Publishing to drop the long-lived `PYPI_API_TOKEN` secret — that's a
> maintainer judgment call, not required for this bump.
>
> Safe to merge as-is.

---

No `gh pr create` or `gh pr comment` was executed. The above is the full would-be content for
review.

## Note on repository state

Unrelated to this task: `/home/pirat/Projects/pybroker`'s current branch changed from
`validation/v3-parallel-backends` (as recorded at session start) to `dev` during this session,
per `git reflog` timestamps that predate/overlap this investigation — consistent with a
concurrent process (this skill's own eval harness appears to run multiple variants against
this same checkout, per the `iteration-1/.../with_skill` and `without_skill` directories
present here) rather than anything this task's commands did. This task only read files, used
`gh api`/`gh pr view`/`gh pr diff` (read-only), and did its build/`twine check` verification in
a separate, already-cleaned-up git worktree under the scratchpad directory — no file in the
main checkout was modified or staged by this task.

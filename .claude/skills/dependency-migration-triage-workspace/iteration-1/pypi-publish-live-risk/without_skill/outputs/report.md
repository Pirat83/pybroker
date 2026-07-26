# Dependabot PR #9 Risk Assessment: `pypa/gh-action-pypi-publish` 1.5.0 → 1.14.1

**Repo:** Pirat83/pybroker
**PR:** https://github.com/Pirat83/pybroker/pull/9
**Base branch:** `dev`
**Diff:** one line, in `.github/workflows/main.yml`

```diff
-      - uses: pypa/gh-action-pypi-publish@v1.5.0
+      - uses: pypa/gh-action-pypi-publish@v1.14.1
```

This is a dry run / calibration. No `gh pr comment` or `gh pr create` was executed against the real repo — this file and the text response are the only output.

## Verdict: Safe to merge, but the change is untested by CI and matters most on the *next real release tag*

All gated CI checks on PR #9 pass (format, lint, typecheck, test x3 Python versions, build, asv). The **`Publish package`** job itself shows `skipping` — it's gated by `if: startsWith(github.event.ref, 'refs/tags/v')`, and this run was a branch push, not a tag push. That means **the new action version has never actually executed in this repo.** Merging is low-risk for CI health, but the real validation only happens the next time someone pushes a `vX.Y.Z` tag and a live release goes to PyPI.

## What changed in the action between 1.5.0 and 1.14.1

Pulled `action.yml` for both tags directly from GitHub and diffed them, plus read the current README and cross-checked known incident issues on `pypa/gh-action-pypi-publish`.

1. **Execution model changed from `docker` to `composite`.** v1.5.0 built a Docker image locally from a Dockerfile at run time. v1.14.1 is a composite action that generates a trampoline action and pulls a pre-built image from `ghcr.io/pypa/gh-action-pypi-publish:<sha>` at run time.
   - Requires outbound network access to `ghcr.io` during the job. Pybroker's runner is a plain `ubuntu-latest` with no `step-security/harden-runner` or other egress firewall, so this is not a concern here (egress-blocking was the root cause of real incidents in other repos — see below).
   - Requires Python 3 preinstalled on the runner to bootstrap the trampoline action. Standard GitHub-hosted `ubuntu-latest` images ship Python; this only broke for self-hosted runners without Python (issue #289, fixed in v1.12.1).
   - **Known past incident (Nov 2024, v1.12.0):** short/invalid handling of Git-SHA-pinned refs caused `Unable to find image ... locally` failures (issue #290), fixed in v1.12.1. Pybroker's PR pins by tag (`@v1.14.1`), not a raw commit SHA, so this failure mode does not apply.
   - **Known past incident:** repos with `step-security/harden-runner` egress allowlists needed to add `ghcr.io:443` (and sometimes a Sigstore rekor CDN host) after this change, because the image pull moved from "before the job's steps" to "at the point this step runs." Pybroker has no such firewall, so not applicable — flag this only if a hardened runner is ever adopted later.

2. **Inputs renamed to kebab-case, snake_case kept as deprecated aliases.** `user`, `password`, `repository_url`->`repository-url`, `packages_dir`->`packages-dir`, `verify_metadata`->`verify-metadata`, `skip_existing`->`skip-existing`, `print_hash`->`print-hash`. Pybroker's workflow only sets `user` and `password`, both still valid, unchanged names. **No action needed**, though migrating to kebab-case (not used here) would be forward-looking since the snake_case aliases are marked "TODO: Remove in v3+".

3. **`password` input is no longer `required`** (was `required: true` in 1.5.0). This reflects Trusted Publishing (OIDC) becoming the supported passwordless flow. Pybroker still explicitly supplies `password: ${{ secrets.PYPI_API_TOKEN }}`, so behavior is unchanged.

4. **`verbose` and `print-hash` now default to `true`** (were `false`). Effect: the publish log will now print upload progress and the SHA-256 hash of every uploaded file. This is strictly more log output, not a behavior change — file hashes are not secrets. No action needed.

5. **New `attestations` input, default `true`.** This is the one item worth double-checking carefully, since it's new in this version range (PEP 740 / Sigstore digital attestations). Per the action's own README (checked directly): "Support for generating and uploading digital attestations is currently limited to Trusted Publishing flows using PyPI or TestPyPI" and "requires authentication with a trusted publisher." Pybroker authenticates with a plain API token (`user`/`password`), not Trusted Publishing (no `id-token: write` permission, no OIDC), so attestation generation simply has no OIDC identity to work with and is a no-op — it does not fail the job. This matches the documented gating and is not expected to break the release.

6. **No new required permissions for the token-auth path.** Trusted Publishing requires `id-token: write`, but that's irrelevant here since pybroker doesn't use it. A separate documented gotcha (`contents: read` needed for **private** repos using Trusted Publishing, issue #237) also doesn't apply — `Pirat83/pybroker` is a public repo, and the workflow has no restrictive `permissions:` block anyway (default token permissions apply).

## Does the workflow need changes to merge this safely?

No blocking changes required. The one-line bump as submitted by Dependabot is functionally compatible with the current `user`/`password` (API token) authentication pattern.

## Optional follow-ups (not required for this PR)

- **Migrate to PyPI Trusted Publishing (OIDC).** This is the direction the action's docs now steer everyone toward, would let `attestations: true` actually produce signed provenance, and removes the long-lived `PYPI_API_TOKEN` secret entirely (replaced by a `pypi` GitHub Environment + `id-token: write` permission scoped to the `publish` job only). Bigger change, own PR, not blocking.
- **Exercise the new pin before the next real release.** Because `Publish package` is always skipped on non-tag CI runs, v1.14.1 has zero execution history in this repo. Consider a manual dry run against TestPyPI (temporary workflow_dispatch job with `repository-url: https://test.pypi.org/legacy/` and a TestPyPI token) before or immediately after the next production tag push, so a genuine live-publish regression doesn't surface for the first time on a real release.
- **Consider pinning third-party actions by full 40-character commit SHA** for supply-chain hardening, since this action publishes real releases to PyPI. Not done currently (repo already pins by tag consistently for all actions, which is a reasonable, lower-maintenance baseline) — flagging only as a defense-in-depth option, not a requirement.

## Sources checked

- `pypa/gh-action-pypi-publish` `action.yml` at tags `v1.5.0` and `v1.14.1` (raw diff)
- `pypa/gh-action-pypi-publish` `README.md` at `v1.14.1` (Trusted Publishing and attestations sections)
- `pypa/gh-action-pypi-publish` issues #289, #290, #237 (real-world incident reports and their root causes/fixes)
- PR #9 metadata, diff, and CI check results (`gh pr view/diff/checks 9`)
- Current `.github/workflows/main.yml` in `Pirat83/pybroker`
- `Pirat83/pybroker` repo visibility (public)

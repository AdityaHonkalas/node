# Fix Validation Report Workflows — Plan

## Top-Level Overview

**Goal:** Create new GitHub Actions workflow files that demonstrate the *corrected* (fixed) version of each existing error-injection workflow, and append a "Generate Report" step to each that produces a Markdown summary artifact. Each workflow is triggered exclusively via `workflow_dispatch` and targets the `main` branch, making it safe to run in fork repositories.

**Scope:** 5 new workflow files — one per requested error category:
1. **Dependency error** (base: `tools_dependency-error.yaml` → `tools.yml`)
2. **Incorrect package version error** (base: `coverage-linux_authentication-error.yaml` → `coverage-linux.yml`)
3. **Syntax error** (base: `linters_syntax-error.yaml` → `linters.yml`)
4. **Command error** (base: `test-linux_test-failure.yaml` → `test-linux.yml`)
5. **Other controlled error** (base: `build-tarball_image-build-failure.yaml` → `build-tarball.yml`)

> The remaining 3 existing error workflows (`test-linux_timeout-error.yaml`, `test-linux-build-error.yaml`, `linters_permission-error.yaml`) are covered by the 5 categories above; not duplicated.

**Approach:**
- Each file is a **standalone `workflow_dispatch`-triggered workflow**.
- It reuses the full steps of its base workflow (same actions, same env vars, same pinned SHAs).
- The injected error from the original file is **removed/corrected**.
- A final `generate-report` job runs `if: always()` after all other jobs and writes a Markdown report to a file, then uploads it as an Actions artifact.

**Design Constraints:**
- `workflow_dispatch` only — no `pull_request` or `push` triggers.
- Compatible with fork repositories (no `secrets.*` references except those already in base workflows that are acceptable for forks).
- Report generation must not depend on external services (pure shell + `actions/upload-artifact`).
- Follow existing naming conventions: `{base-workflow-name}_fix-validation-{error-type}.yaml`.
- All pinned action SHAs are preserved verbatim from their base workflows.
- No new composite actions are introduced.

---

## Sub-Tasks

---

### Sub-Task 1 — Dependency Error Fix Validation Workflow

**Intent:** Create a `workflow_dispatch`-only workflow derived from `tools_dependency-error.yaml` (which breaks because `pip install gcovr==0.0.0` fails). The fix is to remove that invalid pip install step. A report job summarizes which dependency-update matrix jobs ran and marks the fix as validated.

**Expected Outcomes:**
- File: `.github/workflows/tools_fix-validation-dependency-error.yaml`
- Trigger: `workflow_dispatch` only, no branch restriction (fork-safe)
- `workflow_dispatch` input: `id` (choice: all, acorn, ada, libuv, undici, zlib) — same as original
- `tools-deps-update` job: matrix of 5 dependency update tasks (acorn, ada, libuv, undici, zlib)
  - The broken `Install required Python dependencies` step (`pip install gcovr==0.0.0`) is removed
  - All other steps preserved from base
- `generate-report` job: runs after `tools-deps-update` with `if: always()`, writes a Markdown file summarizing the fix, uploads as artifact named `fix-validation-report-dependency-error`

**Todo List:**
1. Copy the structure of `tools_dependency-error.yaml` as the starting point.
2. Remove the `# ERROR INJECTION` step (`pip install gcovr==0.0.0`).
3. Change the trigger to `workflow_dispatch` only (keep the existing `inputs` block).
4. Add `generate-report` job after `tools-deps-update`:
   - `needs: tools-deps-update`
   - `if: always()`
   - `runs-on: ubuntu-latest`
   - Steps:
     a. Checkout
     b. Write Markdown report using `echo` shell commands to `fix-validation-report.md`, documenting: error type, base workflow, what was fixed, result status from `${{ needs.tools-deps-update.result }}`
     c. Upload artifact using `actions/upload-artifact@043fb46d1a93c77aae656e7c1c64a875d1fc6a0a` (v7.0.1)
5. Remove `if: github.repository == 'nodejs/node'` guard from job condition so fork repos can run it.
6. Add header comment block consistent with existing error workflow conventions.

**Relevant Context:**
- Base error workflow: [`.github/workflows/tools_dependency-error.yaml`](.github/workflows/tools_dependency-error.yaml)
- Original clean workflow: [`.github/workflows/tools.yml`](.github/workflows/tools.yml)
- `actions/upload-artifact` pinned SHA already used in `build-tarball.yml`: `043fb46d1a93c77aae656e7c1c64a875d1fc6a0a`

**Status:** `[x] done`

---

### Sub-Task 2 — Incorrect Package Version Fix Validation Workflow

**Intent:** Create a `workflow_dispatch`-only workflow derived from `coverage-linux_authentication-error.yaml`. The error there is an incorrect/invalid token (`INVALID_CODECOV_TOKEN_FOR_CONTROLLED_CI_FAILURE`) for the Codecov upload step. The fix: remove the hard-coded invalid token and replace the `Upload` step with a local report-only step (since Codecov upload requires a real secret, we replace it with a step that just prints coverage summary — this is a realistic fix for a fork environment). Additionally, the original error also represents an "incorrect package version" scenario by referencing `gcovr==7.2` which is a specific pinned version — we demonstrate the fix by also validating that `gcovr==7.2` installs successfully without constraining to a non-existent version.

**Expected Outcomes:**
- File: `.github/workflows/coverage-linux_fix-validation-incorrect-package-version.yaml`
- Trigger: `workflow_dispatch` only
- `coverage-linux` job: identical to base but `Upload` step replaced with a local summary step (no external token needed)
- `generate-report` job: runs after `coverage-linux` with `if: always()`, generates Markdown artifact

**Todo List:**
1. Copy structure of `coverage-linux_authentication-error.yaml`.
2. Remove the hard-coded invalid `token: INVALID_CODECOV_TOKEN_FOR_CONTROLLED_CI_FAILURE` from the `Upload` step.
3. Replace the `Upload` (codecov-action) step with a local step that writes a plain-text coverage summary to a file (e.g., `cat coverage/*.xml | head -20 || echo "Coverage files ready for upload"`).
4. Change trigger to `workflow_dispatch` only.
5. Add `generate-report` job (`needs: coverage-linux`, `if: always()`):
   - Writes Markdown report documenting: error type (incorrect package version / bad auth token), base workflow, what was fixed, job result status
   - Uploads artifact `fix-validation-report-incorrect-package-version`
6. Add header comment consistent with conventions.

**Relevant Context:**
- Base error workflow: [`.github/workflows/coverage-linux_authentication-error.yaml`](.github/workflows/coverage-linux_authentication-error.yaml)
- Original clean workflow: [`.github/workflows/coverage-linux.yml`](.github/workflows/coverage-linux.yml)

**Status:** `[x] done`

---

### Sub-Task 3 — Syntax Error Fix Validation Workflow

**Intent:** Create a `workflow_dispatch`-only workflow derived from `linters_syntax-error.yaml`. The error is an unclosed shell command substitution `$(` in the `Lint JavaScript files` step. The fix: restore the correct shell syntax (close the subshell or remove the erroneous `$(`). The report job documents the fix.

**Expected Outcomes:**
- File: `.github/workflows/linters_fix-validation-syntax-error.yaml`
- Trigger: `workflow_dispatch` only
- All jobs from `linters_syntax-error.yaml` preserved: `lint-addon-docs`, `lint-cpp`, `lint-js-and-md`, `lint-py`
- In `lint-js-and-md`: the broken `run:` block in "Lint JavaScript files" is replaced with the correct syntax from `linters.yml` (remove the stray `$(`)
- Note: `linters_syntax-error.yaml` also has a corrupted `setup-python` action SHA in `lint-cpp` (`@a309ff8b426b58ec0e2a45f0f anthropic-provider-0f869d46889d02405`) — this must also be corrected to the valid SHA `@a309ff8b426b58ec0e2a45f0f869d46889d02405`
- `generate-report` job: runs after all lint jobs with `if: always()`, collects per-job results, writes Markdown, uploads artifact `fix-validation-report-syntax-error`

**Todo List:**
1. Copy structure of `linters_syntax-error.yaml`.
2. Fix the corrupted `setup-python` action SHA in `lint-cpp` job (restore the correct SHA).
3. Fix the `Lint JavaScript files` step in `lint-js-and-md`: replace the broken run block (with `$(`) with the correct version from `linters.yml`.
4. Change trigger to `workflow_dispatch` only (remove `pull_request` and `push` triggers).
5. Add `generate-report` job (`needs: [lint-addon-docs, lint-cpp, lint-js-and-md, lint-py]`, `if: always()`):
   - Runs on `ubuntu-slim`
   - Writes Markdown table with each job name and its result (`${{ needs.lint-addon-docs.result }}`, etc.)
   - Uploads artifact `fix-validation-report-syntax-error`
6. Add header comment.

**Relevant Context:**
- Base error workflow: [`.github/workflows/linters_syntax-error.yaml`](.github/workflows/linters_syntax-error.yaml)
- Original clean workflow: [`.github/workflows/linters.yml`](.github/workflows/linters.yml) — lines 108–133 contain the correct `Lint JavaScript files` step
- Correct `setup-python` SHA: `a309ff8b426b58ec0e2a45f0f869d46889d02405`

**Status:** `[x] done`

---

### Sub-Task 4 — Command Error Fix Validation Workflow

**Intent:** Create a `workflow_dispatch`-only workflow derived from `test-linux_test-failure.yaml`. The error is a deliberately broken test command (passing a `--globals-list /nonexistent/globals.txt` flag followed by `exit 1`). The fix: restore the clean `make test-ci` command from `test-linux.yml` without the invalid flags or forced exit.

**Expected Outcomes:**
- File: `.github/workflows/test-linux_fix-validation-command-error.yaml`
- Trigger: `workflow_dispatch` only
- `test-linux` job: matrix `[ubuntu-24.04, ubuntu-24.04-arm]`, same env, same steps as `test-linux.yml`
- The broken `Test` step replaced with the clean version: `make test-ci -j1 V=1 TEST_CI_ARGS="-p actions --measure-flakiness 9"`
- Additional steps from clean `test-linux.yml` also restored: "Ensure running tests did not cause any change in the tree" and "Re-run test in a folder whose name contains unusual chars"
- `generate-report` job: runs after `test-linux` with `if: always()`, writes Markdown artifact `fix-validation-report-command-error`

**Todo List:**
1. Copy structure of `test-linux_test-failure.yaml`.
2. Replace the broken `Test` step (`--globals-list /nonexistent/globals.txt` + `exit 1`) with the clean version from `test-linux.yml`.
3. Add back the two steps that were present in `test-linux.yml` but absent from the error workflow: "Ensure running tests did not cause any change in the tree" and "Re-run test in a folder whose name contains unusual chars".
4. Change trigger to `workflow_dispatch` only.
5. Remove the `if: github.event.pull_request.draft == false` job condition (irrelevant for `workflow_dispatch`).
6. Add `generate-report` job (`needs: test-linux`, `if: always()`):
   - Runs on `ubuntu-latest`
   - Writes Markdown report
   - Uploads artifact `fix-validation-report-command-error`
7. Add header comment.

**Relevant Context:**
- Base error workflow: [`.github/workflows/test-linux_test-failure.yaml`](.github/workflows/test-linux_test-failure.yaml)
- Original clean workflow: [`.github/workflows/test-linux.yml`](.github/workflows/test-linux.yml) — lines 89–101 contain the correct test steps

**Status:** `[x] done`

---

### Sub-Task 5 — Other Controlled Error (Image Build Failure) Fix Validation Workflow

**Intent:** Create a `workflow_dispatch`-only workflow derived from `build-tarball_image-build-failure.yaml`. The error is a `docker build` step that references a non-existent base image (`node:99999-nonexistent`). The fix: remove the entire broken Docker build step. The rest of the workflow (make tarball → upload → test tarball) runs unmodified.

**Expected Outcomes:**
- File: `.github/workflows/build-tarball_fix-validation-image-build-failure.yaml`
- Trigger: `workflow_dispatch` only
- `build-tarball` job: identical to base but without the "Build Docker image for distribution verification" step
- `test-tarball-linux` job: identical to base (depends on `build-tarball`)
- `generate-report` job: runs after both jobs with `needs: [build-tarball, test-tarball-linux]`, `if: always()`, writes Markdown artifact `fix-validation-report-image-build-failure`

**Todo List:**
1. Copy structure of `build-tarball_image-build-failure.yaml`.
2. Remove the "Build Docker image for distribution verification" step (the `docker build` step with the `node:99999-nonexistent` Dockerfile).
3. Change trigger to `workflow_dispatch` only.
4. Remove the `if: github.event.pull_request.draft == false` job conditions.
5. Add `generate-report` job (`needs: [build-tarball, test-tarball-linux]`, `if: always()`):
   - Runs on `ubuntu-latest`
   - Writes Markdown report documenting: error type (image build failure), base workflow, fix applied, job results
   - Uploads artifact `fix-validation-report-image-build-failure`
6. Add header comment.

**Relevant Context:**
- Base error workflow: [`.github/workflows/build-tarball_image-build-failure.yaml`](.github/workflows/build-tarball_image-build-failure.yaml)
- Original clean workflow: [`.github/workflows/build-tarball.yml`](.github/workflows/build-tarball.yml)
- `actions/upload-artifact` SHA: `043fb46d1a93c77aae656e7c1c64a875d1fc6a0a` (v7.0.1)
- `actions/download-artifact` SHA: `3e5f45b2cfb9172054b4087a40e8e0b5a5461e7c` (v8.0.1)

**Status:** `[x] done`

---

## Report Artifact Format

Each `generate-report` job produces a `fix-validation-report.md` file uploaded as a named artifact. The Markdown file must contain:

```markdown
# Fix Validation Report — {Error Type}

**Workflow Run:** ${{ github.run_id }}
**Repository:** ${{ github.repository }}
**Branch:** ${{ github.ref_name }}
**Triggered by:** ${{ github.actor }}
**Run date:** (via `date -u`)

## Error Type
{Error type name and description}

## Base Workflow
{Name of the error-injection workflow this is derived from}

## Fix Applied
{Description of what was corrected}

## Validation Result

| Job | Result |
|-----|--------|
| {job-name} | {needs.job-name.result} |

## Status
{PASSED / FAILED based on job results}
```

---

## File Naming Convention

| Error Type | New Workflow File |
|---|---|
| Dependency error | `tools_fix-validation-dependency-error.yaml` |
| Incorrect package version | `coverage-linux_fix-validation-incorrect-package-version.yaml` |
| Syntax error | `linters_fix-validation-syntax-error.yaml` |
| Command error | `test-linux_fix-validation-command-error.yaml` |
| Image build failure (other) | `build-tarball_fix-validation-image-build-failure.yaml` |

All files land in `.github/workflows/`.

---

## Notes for Implementation

- The `if: github.event.pull_request.draft == false` condition on each job **must be removed** in the fix-validation workflows since there is no pull_request event — only `workflow_dispatch`. Without removing this, jobs will be skipped on manual dispatch (the expression evaluates to false when there is no PR context).
- For Sub-Task 4 (command error / test-linux), the build and test steps are very long-running in practice. The `generate-report` job uses `if: always()` so the report is produced even if build/test steps fail due to environment constraints.
- For Sub-Task 2 (coverage), the Codecov upload is replaced with a local step because the real upload requires a repository secret that forks do not have. The fix demonstrates the *pattern* of fixing an authentication error (removing the hardcoded bad token and gracefully handling the upload in a fork-safe way).
- Pinned action SHAs must not be altered unless they are the injected error (as in `linters_syntax-error.yaml` where the SHA was corrupted as part of the error injection).

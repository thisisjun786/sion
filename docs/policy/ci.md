# Continuous Integration

The [CI workflow](../../.github/workflows/ci.yml) classifies a change, runs the applicable checks, and reports one required result: `foundation`.

## Execution

```text
selection ----+---- docs -------+---- foundation
              +---- automation-+
```

`selection` records the exact base and candidate commits, changed paths, selected checks, and the selection reason. It also checks Git whitespace errors. `docs` and `automation` run independently after selection. `foundation` only evaluates their results; it never reruns their commands.

| Event | Coverage |
| --- | --- |
| PR into `dev` | Classify the diff from the target base to GitHub's merge candidate. |
| Push to `dev` | Classify the integrated commit against the previous dev head. This produces the exact-commit evidence used for release. |
| Manual dispatch | Run every current check, regardless of changed paths. |
| PR into `main` | Reject the target. A metadata-only job also attempts to close the PR; fork token restrictions may prevent closure. |

New runs cancel obsolete runs for the same PR or branch. Each job has a bounded timeout. Short checks share one job and setup; independent checks run in parallel. Add platforms only for an actual support commitment, not as an empty matrix.

## Change classification

The executable path map is [ci-scope.mjs](../../.github/scripts/ci-scope.mjs).

| Change | Selection |
| --- | --- |
| Only explicitly listed root/PR documents or Markdown directly under `docs/policy/` or `docs/design/` | `docs`; no automation installation or test run. |
| Workflow, issue-template, verification-script, or shared repository configuration | Both checks. |
| Empty or unavailable diff | Both checks; never assume documentation-only. |
| An unmapped path in the changed paths or candidate tree | Both checks, but `foundation` refuses to pass until its verification is registered. |

Deleted paths and both sides of a rename participate. A `.md` suffix alone does not make a file prose: prompts, executable examples, and test inputs need their consumer's checks.

## Current checks

- **docs:** validate local file targets in all tracked Markdown and `LICENSE`, including links from unchanged documents to deleted targets. This does not validate external URLs, heading anchors, or the meaning of prose.
- **automation:** check JavaScript syntax, run the Node behavioral suite, and validate Actions syntax and shell blocks with the pinned actionlint release. Release tests execute the workflow's actual shell steps against disposable Git repositories and explicit GitHub-response fixtures; they never publish real releases. The pinned actionlint archive is cached by version and checksum; every run re-verifies the checksum before extraction.

The scripts use Node 20 or newer and built-in modules. Release fixtures also use Bash, Git, and jq. Action versions and the actionlint checksum live in the workflow.

## Timing and budget

| mode | baseline run | event | cache | wall s | runner s | wall bound s | runner bound s |
| --- | --- | --- | --- | ---: | ---: | ---: | ---: |
| full | 36749604362 | push | miss | 29 | 22 | 50 | 40 |
| docs | 36749682569 | push | not used (automation skipped) | 36 | 20 | 60 | 40 |

The bound is U(x) = 10 * ceil(max(1.5x, x + 15) / 10) seconds, applied to wall and runner seconds; wall is measured from run creation to the last non-skipped job completion of attempt 1, and runner is the sum of non-skipped job durations.

A run above its bound is investigated before the bound is changed; a bound changes only by PR with the new measurement.

## Passing and failing

[ci-gate.mjs](../../.github/scripts/ci-gate.mjs) always runs after the selected jobs. Selection must succeed, every selected job must report `success`, and an unselected job must report `skipped`. Missing, failed, cancelled, malformed, or unexpectedly skipped results fail the gate. Do not use workflow-level path filters that prevent the required result from appearing.

The Actions summary names the candidate, selection reason, and job results. A green `foundation` means the registered checks passed, not that unimplemented product behavior or model quality was established.

## Local verification and extension

```sh
node .github/scripts/check-docs.mjs
node --test .github/scripts/*.test.mjs
actionlint
git diff --check
```

Use the actionlint version pinned in the workflow. During iteration, run the affected test file rather than repeating the entire suite. Pure prose needs reading and link/format checks, not tests pinning its wording.

Introduce a new component together with its real verification commands, path mapping, and gate expectations. Tests must exercise the behavior being changed; bug fixes need a regression that fails without the fix. Build/install checks should use the produced artifact. Share commands between local and CI execution, and avoid running the same suite again in an aggregator. Cache downloads and reproducible build inputs, not previous pass/fail results.

## Upstream tree

SION's product code is the upstream ClawSweeper tree under `upstream/` and SION's additions to it ([source layout](../design/sion.md#source-layout)). The [dependency policy](dependencies.md) governs its packages.

### Toolchain

The `foundation` gate builds and tests the tree with upstream's own toolchain, read from `upstream/package.json`:

- Node.js at the major version that upstream's `engines` field requires: Node 24.
- pnpm at the exact version that upstream's `packageManager` field names, activated with corepack.
- `pnpm install --frozen-lockfile` in `upstream/`, which fails when upstream's lockfile and manifest disagree.

### Registration and path mapping

The PR that brings in the upstream tree registers it in [ci-scope.mjs](../../.github/scripts/ci-scope.mjs). Until then its paths remain unmapped and `foundation` refuses to pass. A change to any path under `upstream/` selects the `upstream` job.

Every path under `upstream/` belongs to the tree, including Markdown and upstream's own `.github/`. SION's docs check skips Markdown under `upstream/`; upstream's own docs check covers it as part of `pnpm run check`. SION's own TypeScript, the generated protocol types and the conformance fixtures register their commands and paths in the same way when they are added.

### Job

`upstream` runs after `selection`, in parallel with `docs` and `automation`. `foundation` evaluates it like the others: when selected it must report `success`, and when not selected it must report `skipped`.

- **upstream:** sets up the toolchain above, installs with `pnpm install --frozen-lockfile` in `upstream/`, and runs `pnpm run check` there, the check that upstream's own CI runs. It has no secrets and no write token. Checks that call a model or need a provider credential run separately, as described under Boundaries and releases.

The job caches the pnpm store by the hash of upstream's lockfile and workspace file, never `node_modules` or results. Adding the job changes the full-mode run, so the PR that registers the tree records a new full-mode baseline and bound in the timing table.

```sh
corepack enable
cd upstream
pnpm install --frozen-lockfile
pnpm run check
```

## Boundaries and releases

Verification uses read-only tokens and disposable runners. Only the metadata-only PR-closing job has pull-request write permission; it does not check out contributor code. Do not expose account secrets, personal data, or production services to PR code. Actual model/provider calls and subjective quality evaluations run separately in explicitly authorized environments.

The [release workflow](releases.md) requires successful dev push CI for its exact source commit. A PR result for a different merge candidate is not a substitute. The [PR policy](pull-requests.md) governs integration; CI success does not authorize publication or deployment.

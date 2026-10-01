# SION

SION (Sweeping Inspector Over Noise) is a repository maintainer. It reviews every issue and pull request, repairs what it can, merges what is ready, and tells maintainers what comes next. This contract fixes what SION does, who else may do the same work, how SION is built on ClawSweeper and follows its upstream, how it runs on GitHub alone, which commands and stages it offers, and how it connects to LINA. It is normative: implementations must follow it, and any change to it goes through a pull request against this file. Finishing this document does not mean any part of SION runs; runtime proof belongs to the [roadmap](https://github.com/thisisjun786/sion/issues/1) stages that consume it.

## Scope

This contract covers:

- SION's four jobs and the roles it shares with LINA and maintainers
- the ClawSweeper base and the rules for following upstream
- the GitHub Actions runtime: source, operator and target repositories, the GitHub App identity, model credentials, the `state` and `mailbox` branches, concurrency and the status dashboard
- commands, the internal repair loop and per-repository stages
- the LINA link

It does not define the envelope, the protocol JSON Schema, the supported-combination table or the conformance fixtures; LINA's [host protocol](https://github.com/thisisjun786/lina/blob/dev/docs/design/host-protocol.md) defines them. The sibling-compatibility conditions are defined in LINA's [product families](https://github.com/thisisjun786/lina/blob/dev/docs/design/product-families.md). Contribution, CI, branch and release rules live in this repository's policies (see Repository policy). Product principles live in the [manifesto](../../MANIFESTO.md).

## What SION does

- **Sweep.** SION finds duplicates, items the default branch already fixed, and questions nobody answered. It closes an item only when the evidence is clear and the repository's policy allows that close. Otherwise it says what is missing and leaves the item open.
- **Inspect.** SION reviews every open issue and pull request: the code, the tests, the checks, and the gap between what the description promises and what the diff does. A review is a proposal.
- **Operate.** On pull requests a maintainer opted in, SION addresses review findings, repairs failing checks, refreshes stale branches and merges what is ready.
- **Navigate.** Every item gets a next step: what is missing before merge, who should act, and why.

SION keeps one record per item and one comment per item, edited in place. Every write is checked against live repository state immediately before it happens; a stored judgment never authorizes a write on its own.

SION is open source and works without LINA. Every installation is self-hosted: the people who run it own its GitHub App, its model credentials and its records.

## Roles

No role is exclusive. Merging, fixing, closing and opening pull requests may be done by SION, by LINA Core or by a person, whoever the repository's rules allow. Two things are enforced for every actor:

- **Pre-merge safety.** Every merge passes the repository's required checks and protection rules, and the merged head and the protected refs match what was verified. SION's own merges also pass SION's merge gates (see Commands).
- **Records.** Every result is recorded, whoever acted. SION records each write it makes in its action ledger and records what it observes others do on the items it tracks. In a repository connected to LINA, LINA reads SION's result as a sibling record and applies it to its own canon (see LINA link).

Default responsibilities:

| Actor | Usually does |
| --- | --- |
| SION | Reviews issues and pull requests in its allowed repositories with its built-in Codex review; sweeps and closes items with clear evidence; fixes and merges opted-in pull requests; leaves the next step on every item |
| LINA Core | Sends Codex workers on work LINA owns and opens their pull requests; may fix or merge them itself under the same two rules |
| Klotho (LINA's planning module) | Supplies judgments from LINA's goals and plans, such as whether an item is needed and what comes first, as review input for SION |
| Maintainers | Choose repositories, stages and policy; call SION with comment commands |

SION's code review is its built-in Codex review; SION integrates no other review service. Pull requests opened for LINA go through SION's review, fix and merge like any other pull request.

In a repository whose PR policy requires the owner to authorize each merge, SION merges only pull requests the owner opted in with `/sion automerge`. That opt-in is the owner's authorization for that pull request until it is withdrawn. This repository is such a repository ([PR policy](../policy/pull-requests.md)), and so is LINA's.

## ClawSweeper base

SION is built on [ClawSweeper](https://github.com/openclaw/clawsweeper) (MIT, TypeScript), the conservative maintenance bot of the OpenClaw repositories, and follows its upstream. SION is TypeScript on Node.js, like upstream, because it inherits upstream's code instead of translating it.

SION keeps from upstream, unchanged:

- review, sweep, repair and merge behavior and the lanes that carry it: review, apply and repair
- the safety model: review is proposal-only; Codex never holds a credential that can write to a target repository; a deterministic executor performs every GitHub write after rechecking live state; secret-scanning admission runs before a model sees repository content; an exact-head review precedes every repair push and every merge
- record formats, close reasons, repository profiles and the runner interface

SION adds, and keeps thin:

- installation settings: which repositories, which stage, who may call which command
- SION naming: the `/sion` command prefix, the `sion:` label prefix, comment markers and the App name
- the LINA link
- the GitHub-only runtime

The GitHub-only runtime is the one large divergence. Upstream keeps canonical records and its work queue in a Cloudflare Worker with Durable Objects, and keeps action ledgers and assets in R2. SION keeps records, ledgers, queue leases and assets on the operator repository's `state` branch, serializes work per item with Actions concurrency, and publishes its status dashboard with GitHub Pages. No capability those services provide is dropped. All storage access sits behind one storage adapter boundary, so the divergence lives in one place in the code. Upstream code that serves OpenClaw's own deployment, such as the profiles of OpenClaw's repositories, private inference routing and hosted fleet tooling, stays in the tree unchanged and is not enabled by SION's settings.

Following upstream:

- [Third-party notices](../../THIRD-PARTY-NOTICES.md) record the upstream commit SION contains, upstream's license notice and a summary of SION's modifications.
- An upstream sync is a pull request into `dev` that merges one named upstream commit with a merge commit. Conflicts are resolved at the storage adapter boundary or in the SION-only additions, never by rewriting upstream behavior. The sync passes `foundation` with upstream's tests and SION's tests.
- A fix that is not specific to SION is also offered upstream. SION drops its own copy once upstream has it.
- SION does not reformat, rename or reorganize upstream files.
- Each SION release names the upstream commit it contains.

## Runtime

SION is a GitHub Actions bot. It runs no server of its own and uses no store outside GitHub.

```text
SION source     this repository: code, reusable workflows, the target dispatcher, releases
Operator repo   one per installation: workflows pinned to a SION release tag, settings,
                secrets, the state branch, the mailbox branch and the Pages dashboard
Target repo     a repository SION serves: one dispatcher workflow and the SION GitHub App
```

### Operator repository

Each installation has exactly one operator repository. It holds:

- workflows that call SION's reusable workflows pinned to one SION release tag. Moving to another SION release is a pull request in the operator repository that sets the new tag. Release tags are immutable ([release policy](../policy/releases.md)).
- settings on the default branch: the allowed target repositories and, for each one, its stage, repository profile, command permissions, LINA link switch and limits. SION never acts on a repository the settings do not list.
- Actions secrets: the SION App private key and the model API key.
- the `state` branch (see State branch) and the Pages site built from it.
- the `mailbox` branch, when the LINA link is on (see LINA link).
- rulesets that keep each branch to its writer: only the SION App updates `state`, and only the operator repository's maintainers update `mailbox`. No identity may force-push or delete these branches.

Installation state lives only in the operator repository. SION's source repository and LINA's repository hold none. The operator repository must be no more visible than the most restricted target it serves, because records quote target content.

Operator workflows cover scheduled scans, event intake, item workers for the review, apply and repair lanes, state publication and the dashboard build. Each job declares the smallest workflow token it needs: `pages: write` and `id-token: write` only in the Pages deployment, `actions: write` only in jobs that queue follow-up runs. A job that pushes `state` does so with a SION App installation token narrowed to the operator repository and `contents: write`, because the `state` ruleset admits only the SION App.

### Target dispatcher

Each target repository has exactly one dispatcher workflow, shipped with every SION release and pinned to the same tag as the operator workflows. It listens to `issues`, `issue_comment` and `pull_request_target` events and forwards each one to the operator repository with `repository_dispatch`.

- It uses `pull_request_target` so it works for pull requests from forks, and it never checks out or runs pull request code.
- It forwards identifiers only: repository, item number, event, action, comment ID and head SHA. It forwards no event body; the operator workflows read everything again from GitHub.
- A dispatch is a wake-up, not an authorization. The operator workflows check the item, the commenter's live permission and the repository's stage themselves, so a forged or replayed dispatch causes at most a re-read.
- The dispatcher uses the SION App, as upstream does. The App's private key is a target-repository Actions secret, used only to mint a short-lived installation token narrowed to the operator repository and the permission `repository_dispatch` requires, for that one call.

Scheduled scans in the operator repository cover what the dispatcher cannot: dropped events, items that predate the installation, and repositories without a dispatcher.

### Identity

SION acts on target repositories only as its GitHub App, the SION App. Every write appears under the App's name. SION never acts with a personal access token.

- The SION App has no webhook endpoint. Events reach SION only through the dispatcher and scheduled scans.
- Every job mints a short-lived installation token narrowed to one repository and to the permissions of that repository's stage and of the write it performs (see Stages).
- No job holds a token that can write to a target repository while Codex or target-repository code runs in it. A job that runs Codex reads with a read-only token and emits a hash-bound artifact. A separate executor job, with no model process, verifies the artifact and the live state, then writes.
- The SION App holds the Workflows write permission, as upstream's App does, so SION rebases and repairs pull requests that change files under `.github/workflows/` like any other pull request. SION never requests the Administration or Secrets permissions.
- Raising the App's permissions for a higher stage requires the installation owner to accept them on GitHub.

### Model credentials

SION's reviews and repairs run Codex. The model API key is an operator-repository Actions secret. In each job that runs Codex, a local Responses proxy on localhost holds the key; Codex sends its requests to the proxy and never receives the key. No App private key reaches the Codex process or anything Codex runs.

### State branch

The `state` branch of the operator repository holds everything SION remembers. It is never merged into the default branch. Its ruleset lets only the SION App update it.

```text
records/    one record per item: decision, evidence, next step, GitHub snapshot hash
ledger/     immutable action events: the intent and the outcome of every write
queue/      intakes for events and commands, leases, dead letters
assets/     published assets that records refer to
status/     data the dashboard is built from
jobs/, results/, notifications/
            upstream's operational state
outbox/     envelopes SION writes for LINA, and SION's declaration (LINA link)
```

Directories that also exist upstream keep upstream's names and formats.

- **One head, one writer at a time.** Every write to `state` is a fast-forward push. A rejected push means another writer moved first; the writer fetches, reapplies its change and pushes again. The branch head is the compare-and-swap point, so no writer overwrites another.
- **Write-ahead ledger.** Ledger events are new files named by event ID and are never edited. SION writes the intent of a GitHub write to the ledger before performing it and the outcome after. An intent without an outcome is unknown. SION resolves it by reading live state and never repeats the write until live state shows the write did not happen.
- **Durable intake.** The intake job writes every forwarded event and every command to `queue/` before any worker acts on it.
- **Leases.** A lease is a queue entry naming its owner run, attempt and expiry, acquired by a fast-forward push. A worker verifies it still holds its lease before running the model and again before each write. Another worker may take an expired lease.
- **Per-item serialization.** Item workers run in an Actions concurrency group keyed by repository and item number, so at most one run acts on an item at a time. Actions keeps only the newest pending run in a group; that only merges wake-ups, because every event and command is already durable in `queue/`.
- **Retry artifacts.** Review artifacts that serve only retries are hash-bound Actions artifacts, not branch content.

### Status dashboard

An operator workflow builds a static status dashboard from the `state` branch and publishes it with GitHub Pages. It shows the queue, running and recent jobs, per-repository status, recent actions, failures and automerge progress.

- The dashboard is observability only. It never starts, steers or authorizes work, and the page makes no requests to GitHub.
- A Pages site is public unless the account provides Pages access control. The build therefore includes only public target repositories unless the operator repository's Pages site is access-controlled, and it refuses to publish otherwise.

### Costs of the GitHub-only runtime

- Scheduled runs can start late or be skipped. Scans resume from cursors on `state` and never assume that a run happened.
- Command acknowledgement waits for an Actions run to start, because there is no webhook endpoint.
- Writes to `state` serialize at one branch head, which bounds write throughput.
- Upstream syncs conflict at the storage adapter boundary. That is the one place SION accepts recurring conflicts.
- Actions minutes, artifact storage and Pages count against the operator's account.

## Commands

Maintainers call SION with a comment whose first line starts with `/sion`. SION does not answer `@sion`, because that mentions the GitHub user `sion`.

| Command | On | Who may call | Effect |
| --- | --- | --- | --- |
| `/sion review` | issue or pull request | the item's author, or anyone with write permission | Fresh review of the item at its current head. Updates SION's one comment and writes nothing else. |
| `/sion fix` | pull request | write permission | Runs the repair loop on the current head. Ends when the review is clean and required checks pass, when the round bound is reached, or when SION stops for a person. Never merges. |
| `/sion autofix` | pull request | write permission | Opts the pull request in with `sion:autofix`. Runs the repair loop whenever a new head, review finding or failing check appears. Never merges. |
| `/sion automerge` | pull request | write permission, narrowed by settings | Opts the pull request in with `sion:automerge`. Works as `autofix`, then merges when every merge gate passes. A draft pull request is fix-only until it is marked ready. |

- Write permission means the commenter's live `admin`, `maintain` or `write` permission on the target repository, read when the command runs. Settings may narrow who may call each command per repository; a repository whose policy requires the owner to authorize merges restricts `/sion automerge` to the owner.
- A command the repository's stage does not allow is refused with a reply that names the stage. At the `report` stage the refusal is recorded only.
- Replies to commands go into one marker-backed status comment per item, edited in place.
- Labels stop SION. Removing `sion:autofix` or `sion:automerge` withdraws the opt-in. The hold label `sion:hold`, set by SION when it stops for a person or by a maintainer, stops every automatic write on the item until a maintainer removes it. SION reads the labels again before every write.

### Repair loop

Repair is the internal loop that `/sion fix`, `/sion autofix` and `/sion automerge` run. It has no command of its own.

1. Review the pull request at its exact head.
2. If the review or the required checks show actionable problems, Codex refreshes the branch, addresses the findings, fixes failing checks and runs the validation. It does this in a job without write tokens and returns a repair artifact.
3. The executor reads the live head again. If the head moved or the pull request closed, it requeues instead of pushing. Otherwise it pushes the artifact's commits.
4. Review the new head at its exact head, then wait for the required checks.
5. Repeat until the result is clean, the round bound is reached, or a stop condition holds.

The merge gates for `/sion automerge` are: a clean exact-head review of the final head, passing required checks, GitHub reporting the pull request mergeable, protection rules satisfied, the opt-in label present, no hold label, and the repository profile allowing the merge. A security-sensitive finding is repaired only under an explicit opt-in, and the pull request does not merge until a later exact-head review is clean.

## Stages

A stage is the level of automation enabled for one target repository in the operator settings. Each stage includes the one before it. The operator sets each repository's stage directly.

| Stage | SION may | Target-repository token permissions |
| --- | --- | --- |
| `report` | Review items and keep records on `state`. Write nothing to the target repository. | Read: metadata, contents, issues, pull requests, checks, commit statuses, actions |
| `review` | Post and edit one comment per item; reply to commands. | `report` plus write: issues, pull requests |
| `sweep` | Apply advisory labels; close items with clear evidence under the repository profile; reopen an item it closed wrongly. | Same as `review` |
| `operate` | Run `/sion fix`, `/sion autofix` and `/sion automerge` on opted-in pull requests. | `review` plus write: contents, workflows |

The App's installation permissions are those of the highest stage in use. Each job's token is narrowed further, as described in Identity. Closing and merging are also limited by the repository profile's close reasons and merge gates.

## LINA link

SION and LINA are siblings. Each has its own repository, canon and voice, and each keeps every feature when the other is absent or the link is off. They never call each other at runtime; the operator repository's `mailbox` and `state` branches are their only contact points. The link is switched on per target repository in the operator settings. When it is off for a repository, SION reads nothing from the `mailbox` branch and writes nothing to `outbox/` for it.

### Messages

- Every message between LINA and SION is an envelope as defined by LINA's host protocol, carrying a SION payload. SION defines no message format of its own, and payload fields are defined only in LINA's schema.
- LINA to SION, on the `mailbox` branch: judgment input for a repository or an item, such as its relevance to LINA's goals and plans, its priority and related work.
- SION to LINA, in `outbox/`: item results. A result says what happened (reviewed with its verdict, fixed, merged, closed, reopened) and its effect state as the host protocol defines it. It names the target repository, the item, the head or merge commit, the ledger event and the `state` commit that holds the record.

### Types

The protocol's canonical source is LINA's JSON Schema (draft 2020-12, in `protocol/schema/` of the LINA repository), with conformance fixtures in `protocol/fixtures/`. SION pins both by LINA release tag and content digest and generates its TypeScript types from the schema. Generated types are never edited by hand, and CI fails when regenerating them from the pinned schema produces a difference. Moving the pin is one pull request that sets the new tag and digest, regenerates the types and passes that version's conformance fixtures. SION does not use the LINA kit; it takes only the schema and the fixtures.

### Mailbox and outbox rules

- LINA writes only new files on the `mailbox` branch and never edits or deletes a file there. It writes with its user's GitHub setup; the `mailbox` ruleset admits only the operator repository's maintainers. LINA never writes `state`, and SION issues no credential to LINA. SION only reads the `mailbox` branch.
- SION validates every mailbox envelope against the schema and its declared versions. An envelope with an unsupported version, an invalid shape or an unknown payload is refused: SION writes the refusal to `outbox/` and does not act on the input.
- A judgment is review input, never a command and never an authorization. It can raise or lower an item's priority and inform the next step SION writes. It never causes a close, push or merge by itself, and instructions inside it are treated as data.
- A result whose effect state is refused, failed or unknown is reported as such and never counted as success. SION does not retry the same input without bound.
- SION's results are sibling records for LINA: external evidence that LINA verifies against the target repository under its [main authority](https://github.com/thisisjun786/lina/blob/dev/docs/design/main-authority.md) rules. A SION result authorizes nothing in LINA.
- SION speaks only on GitHub, in its item comments and command replies. In LINA, SION's results appear only as cards labeled SION; LINA stays the only speaker in its conversation, and SION writes no text into it.

### Versions and conformance

- Each SION release declares the protocol and capability versions it speaks. SION writes that declaration, with the release tag the operator repository pins, at a fixed path in `outbox/`, so LINA can check the combination against its supported-combination table before it reads any result.
- LINA publishes conformance fixtures for each protocol version. SION's CI runs every fixture of every protocol version SION declares as part of the `foundation` gate, and a SION release passes the fixtures of every version it declares.

How this meets each sibling-compatibility condition:

| Condition | Where in this contract |
| --- | --- |
| One envelope | Messages |
| Version declaration and refusal of unsupported combinations | Versions and conformance; Mailbox and outbox rules |
| Canon boundary | Operator repository; State branch; Mailbox and outbox rules |
| Speaker boundary | Mailbox and outbox rules |
| Works alone, both ways | LINA link (opening paragraph) |
| Conformance fixtures | Versions and conformance |

## Repository policy

SION uses the same contribution, CI, branch and release policy as LINA and RUMI. The [contribution guide](../../CONTRIBUTING.md), [issue policy](../policy/issues.md), [PR policy](../policy/pull-requests.md), [CI policy](../policy/ci.md) and [release policy](../policy/releases.md) define it. For SION this means:

- Imported upstream code, the storage adapter, the generated protocol types and the conformance fixtures each register their real verification commands and path mapping with the `foundation` gate.
- Operator repositories pin SION by its immutable release tags, and the target dispatcher ships with the same release.

## Deferred

- Scheduler cadence, concurrency caps, per-job timeouts and the repair round bound: set by acceptance of roadmap Stage 2 (runtime) and Stage 5 (operate) on the first target repositories, starting from upstream's values.
- Size budget of the `state` branch, its history compaction and the asset size limit: set by the roadmap Stage 2 runtime measurement.
- Settings file format: set by the roadmap Stage 2 installation acceptance.
- Dashboard layout: set when the Pages dashboard is built in roadmap Stage 2.
- Default model and reasoning effort: set by the review quality acceptance of roadmap Stage 3.

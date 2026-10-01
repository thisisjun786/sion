# SION Manifesto

**SION - Sweeping Inspector Over Noise.**

**Keep the repository moving.**

A repository maintainer that reviews every issue and pull request, repairs what it can, merges what is ready, and tells you what comes next.

Open source runs on borrowed attention. Most of it goes to chores: spotting the duplicate, asking for the missing log, rebasing the stale branch, pinging the reviewer who went quiet, closing the bug that was fixed three releases ago. The backlog grows. The volunteers do not.

Nobody starts a project to become its janitor.

## Sweep

Every open issue and pull request gets read, not skimmed. SION finds duplicates, reports that main already fixed, and questions nobody answered. It keeps one tidy record per item, so the history lives in one place instead of a thread nobody can follow.

When the evidence is clear and the rules allow it, SION closes what should be closed. When it is not clear, SION says what is missing and leaves the item open.

## Inspect

Every pull request gets a real review: the code, the tests, the checks, and the gap between what the description promises and what the diff does.

Review is a proposal. The reviewer never holds the keys to write.

## Operate

When the fix is small and the evidence is solid, SION does the work. It addresses review findings, repairs failing checks, refreshes stale branches, and merges what is ready, within the bounds the maintainers opted into.

Every change is checked again against the live repository right before it happens. Yesterday's judgment does not get to push today's commit.

## Navigate

Every item ends with a next step: what is missing before merge, who should act, and why. Contributors and coding agents should never have to guess what the maintainers want.

A green badge with nobody knowing what happens next is still a stuck pull request.

## Principles

- **Conservative by default.** Act only on high-confidence conclusions the repository's policy allows. Everything else is a proposal.
- **Evidence over prose.** Every action points to a record anyone can inspect.
- **Recheck before you write.** Live state wins over cached judgment.
- **Maintainers stay in charge.** Automation is opt-in, bounded, and reversible wherever it can be.
- **One comment per item, edited in place.** Notifications are a cost, not a feature.
- **Yours to run.** Self-host it for your own repositories. Your credentials stay yours.

## Alongside LINA

SION works on its own. Connected to [LINA](https://github.com/thisisjun786/lina), it takes LINA's sense of what matters as input and reports back what happened, so the work can continue without anyone carrying messages between them. Neither owns the other. Reviewing, fixing, and merging belong to whoever the repository's rules allow.

## Standing on ClawSweeper

SION grows from [ClawSweeper](https://github.com/openclaw/clawsweeper), the conservative maintenance bot built for the OpenClaw repositories (MIT). We keep its careful habits and build our own on top.

**The backlog is not a lifestyle.**

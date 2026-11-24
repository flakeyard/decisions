---
id: FY-0043
title: Queue a release pull request behind commits that release nothing
status: accepted
date: 2026-11-24
supersedes: []
superseded_by: []
related: [FY-0001, FY-0007, FY-0030, FY-0041]
affects: ["*:release-please-config.json"]
review_by:
---

## Context

[FY-0041](0041-merge-through-the-merge-queue.md) adds a release pull request to
the merge queue only when no other pull request of its repository is in the
queue, the parent of its head is the tip of the base branch and the required
check passed on its head. The second condition is too strict.

release-please opens its release pull request with one commit on top of the tip
of the base branch that it saw, on its branch
`release-please--branches--<base>`. On every later push to the base branch it
builds the release again, and it updates the pull request only when the release
changes. A commit that releases nothing, such as a Renovate `chore(deps)` or
`ci(deps)` update, does not change the release, so the pull request stays as it
is: the parent of its head falls behind the tip, and FY-0041 never lets it into
the queue.

The head cannot be moved by hand. GitHub's update of the branch by rebase writes
an unsigned commit, which the rule `required_signatures` refuses; this was tried
on [workflow-release#126][workflow-release-126], whose head is now such an
unsigned commit.

Whether a commit releases something is decided by release-please, as pinned by
`workflow-release` ([FY-0030](0030-actions-workflows-and-release-bot.md)):

- `workflow-release` pins `googleapis/release-please-action` v5.0.0, whose lock
  file resolves release-please 17.6.0 and its changelog preset
  `conventional-changelog-conventionalcommits` 6.1.0.
- release-please skips a release whose changelog would be empty: "No user facing
  commits found since ... - skipping" (`src/strategies/base.ts`).
- The changelog keeps a commit when its type has a section that is not hidden,
  when it carries a breaking change, written as `!` or a `BREAKING CHANGE`
  footer, of any type, or when it has a `Release-As` footer. A type without a
  section and a message that is not a conventional commit are dropped
  (`writer-opts.js` of the preset).
- The sections come from `changelog-sections` of the repository's
  `release-please-config.json`. Without it, the preset's defaults apply: `feat`,
  `feature`, `fix`, `perf` and `revert` are shown; `docs`, `style`, `chore`,
  `refactor`, `test`, `build` and `ci` are hidden. A repository may show more,
  for example a section for `deps`.
- No `release-please-config.json` of the organization sets `changelog-sections`
  today, and the preflight of `workflow-release` does not reject it. A Renovate
  `fix(deps)` update has the type `fix` and releases; a `chore(deps)` or
  `ci(deps)` update does not.

## Decision

A release pull request may be added to the queue behind commits that release
nothing. It is added only when no other pull request of its repository is in the
queue, the required check passed on its current head, and no releasing commit
lies between the parent of its head and the tip of the base branch, that is
among the commits that the tip has and the parent has not.

A commit is releasing when release-please would put it into a release: its type
has a section that is not hidden in the `changelog-sections` of the repository's
release-please configuration, or in release-please's defaults when the
configuration has none; or it carries a breaking change; or it has a
`Release-As` footer.

When a releasing commit lies in between, release-please rebuilds the pull
request on the push of that commit, and the pull request is added after the
rebuilt head passed its check.

This is safe:

- The queue tests and merges one pull request at a time on top of the tip
  (FY-0041, `max_entries_to_merge` 1), so the release pull request is checked on
  the tree it will merge into, and nothing else lands with it.
- The merge commit of the release pull request is still the commit of its own
  push, so the release workflow finds it with `merged_here` and takes the
  release as due (FY-0030, FY-0041).
- The version and the changelog stay correct. They were built from the commits
  up to the parent of the head; the commits after it release nothing, so
  release-please would build the same version and changelog at the tip. Its next
  run starts from the merge commit of the release, which contains them.

The branch of a release pull request belongs to release-please. It is never
rebased, updated or pushed to by hand or through GitHub's update of the branch;
only release-please changes it.

### Amendments

This decision amends [FY-0041](0041-merge-through-the-merge-queue.md); the rest
of it stays in force. Its rule that the parent of the head of a release pull
request is the tip of the base branch is replaced by the rule above that no
releasing commit lies between them.

## Constraints

- A release pull request MUST be added to the queue only when no other pull
  request of its repository is in the queue, the required check passed on its
  current head, and no releasing commit lies between the parent of its head and
  the tip of the base branch.
- A commit MUST be taken as releasing when its type has a section that is not
  hidden in the `changelog-sections` of the repository's release-please
  configuration, or in release-please's defaults when it sets none, when it
  carries a breaking change, or when it has a `Release-As` footer.
- The branch of a release pull request MUST NOT be rebased, updated or pushed to
  by anyone but release-please.

## Consequences

- (+) A release is no longer stuck behind Renovate updates and other commits
  that release nothing.
- (+) Every commit on `main` stays signed: the release branch is never rewritten
  by GitHub's update of the branch.
- (−) Deciding whether a release pull request may be added needs the commit
  messages between its parent and the tip and the release-please configuration
  of the repository, not only two SHAs.
- (±) A release can include commits that landed after release-please built it;
  they release nothing and have no entry in the changelog, as they would have if
  release-please had rebuilt it.

## Alternatives considered

- **Keeping the parent at the tip.** The release waits for the next releasing
  commit, which may not come for a long time.
- **Updating the branch through GitHub.** The rebased head is unsigned and
  `required_signatures` refuses it.
- **Making release-please rebuild the pull request on every push.** The rebuilt
  pull request would need a new check on every push to the base branch, for no
  change of the release.
- **Closing the pull request so that release-please opens a new one.** Manual
  work on the release branch for each stalled release, and it still needs a push
  to the base branch to trigger release-please.

[workflow-release-126]: https://github.com/flakeyard/workflow-release/pull/126

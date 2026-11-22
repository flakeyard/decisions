---
id: FY-0041
title: Merge through the merge queue
status: accepted
date: 2026-11-22
supersedes: []
superseded_by: []
related: [FY-0001, FY-0007, FY-0011, FY-0029, FY-0030, FY-0033, FY-0034]
affects:
  [
    ".github:org/**",
    "*:.github/workflows/test.yaml",
    "workflow-check:.github/workflows/check.yaml",
  ]
review_by:
---

## Context

[FY-0007](0007-branching-and-releases.md) merges a pull request only when its
branch is up to date with `main`, and
[FY-0034](0034-check-every-repository-with-workflow-check.md) enforces that with
`strict_required_status_checks_policy` on the required check `check / required`.
Every merge to `main` therefore makes every other open pull request of the
repository stale: it has to be updated and checked again before it can be
merged. A measure of the CI runs of flakeyard showed that about 28% of the runs
on pull requests were such updates of a branch, with no change of the pull
request itself. Renovate opens many small pull requests at once, so a batch of
updates waits for one full check per pull request and per merge before it.

FY-0007 said that merge queues are not available on the GitHub Free plan. They
are: an organization on Free can use the merge queue in its public repositories,
and only through the rule `merge_queue` of a branch ruleset. Every flakeyard
repository is public.

A probe repository with the real reusable workflows of `workflow-check` and
`workflow-release` and the rulesets applied by `org/apply.sh`
([FY-0011](0011-settings-as-code.md)) showed:

- With `max_entries_to_merge` 1, every pull request is one push to `main` with
  its own push run. With a larger value, the queue merged two pull requests in
  one push, and the release workflow then misses a release pull request: it
  takes a release as due only when the push merged it, that is when the merge
  commit of the release pull request is the commit of the push
  ([FY-0030](0030-actions-workflows-and-release-bot.md)).
- The check run `check / required` of a `merge_group` run, by GitHub Actions
  (`integration_id` 15368), satisfies the required status check of the ruleset.
- A release pull request opened with `GITHUB_TOKEN` can be added to the queue,
  and its `merge_group` run needs no approval. The push of the queue triggers
  the release workflow, which finds the merged release pull request, and the
  approval of the environment `release` works as before.
- The queue squashes with the pull request title as the message and no body, and
  GitHub signs the commit. `github-merge-queue[bot]` pushes it to `main`.
  `merged_by` of the pull request is the account that added it to the queue.
- The head of a pull request in the queue cannot be pushed: GitHub rejects the
  push with `GH006`. Editing the title keeps the entry. A failing check removes
  the entry and marks it `UNMERGEABLE`.
- The rule `merge_queue` rejects patterns of ref names: a ruleset that includes
  `refs/heads/release/v*` with the queue fails with "Wildcard ref names are not
  supported". `~DEFAULT_BRANCH` is accepted.
- A Free organization runs at most 20 jobs at once, shared by x64 and arm64
  runners and code scanning.

## Decision

Pull requests to the default branch are merged only through the merge queue. The
queue tests each pull request on top of the tip of the branch, so a pull request
no longer has to be up to date before it is merged.

### Rulesets

The branch ruleset of the default branch, `~DEFAULT_BRANCH`, keeps the rules it
has and adds the rule `merge_queue` with:

- `merge_method` `SQUASH`;
- `max_entries_to_merge` 1;
- `max_entries_to_build` 2;
- `grouping_strategy` `ALLGREEN`;
- `min_entries_to_merge` 1 and `min_entries_to_merge_wait_minutes` 0;
- `check_response_timeout_minutes` 60.

Its required status check is still `check / required` with `integration_id`
15368, now with `strict_required_status_checks_policy` off: the queue already
tests against the tip.

Since the queue cannot cover `release/v*`, a second branch ruleset protects
`refs/heads/release/v*` with the same rules except `merge_queue`, and with
`strict_required_status_checks_policy` on. Pull requests to `release/vN` are
merged as before, when they are up to date and checked.

Neither ruleset has bypass actors. A new repository therefore gets its signed
initial commit on `main` first, and `org/apply.sh` applies the rulesets after
it.

### Check workflow

The `test.yaml` of every repository with CI triggers on `merge_group` as well.
The `merge_group` run executes the whole check, and its `check / required`
satisfies the ruleset. A `merge_group` run is not cancelled by a newer run.
`workflow-check` has to accept the event; that change and its details belong to
its [ADR-0001][workflow-check-adr-0001].

### Merging

A pull request is merged by adding it to the queue with the GraphQL mutation
`enqueuePullRequest` and `expectedHeadOid` set to the head that was reviewed and
checked. Only a pull request that GitHub reports as mergeable is added. The
merge button, `gh pr merge` and updates of the branch are no longer used for the
default branch. Renovate's automerge stays off.

A pull request in the queue cannot be updated. To update it, for example when
Renovate rebases it, it is first removed from the queue.

Release pull requests go through the queue as well, one at a time per
repository: a release pull request is added only when no other pull request of
its repository is in the queue, the parent of its head is the tip of the base
branch and the required check passed on its head. The rule that skipped the
other pull requests of a repository with an open release pull request is
dropped. Repositories are still released in the order of their dependencies.

### Security

A `merge_group` run executes the code of the pull request head, workflows
included, with the trust of a `pull_request` run from a branch of the same
repository. The environments `release` and `publish` are deployable only from
`main` and `release/v*`, so their secrets are not exposed to it. For a pull
request from a fork, this is more than its own `pull_request` run gets; adding
it to the queue is the decision to merge it and follows the review of its code.

### Rollout

1. `workflow-check` releases a version that accepts `merge_group`.
2. Every repository with CI adds the trigger to `test.yaml` through its usual
   update.
3. `org/apply.sh` applies the rulesets to one repository first, then to the
   rest.

Until a repository gets the new rulesets, FY-0007 and FY-0034 apply to it
unchanged.

### Rollback

To return to the previous model: set `strict_required_status_checks_policy` back
on in the ruleset of the default branch, remove its rule `merge_queue` and run
`org/apply.sh`. The trigger `merge_group` may stay in `test.yaml`; it never
fires without a queue.

### Amendments

This decision amends other records; the rest of each stays in force.

- [FY-0007](0007-branching-and-releases.md): a pull request to the default
  branch is merged through the merge queue instead of being up to date with the
  branch, and `release/v*` is protected by the same rules as `main` except the
  queue, in a ruleset of its own. Its statement that merge queues are not
  available is corrected in place.
- [FY-0034](0034-check-every-repository-with-workflow-check.md):
  `strict_required_status_checks_policy` is off in the ruleset of the default
  branch and stays on for `release/v*`; `test.yaml` triggers on `merge_group`.
- [FY-0033](0033-custody-of-the-release-bot-key-and-audit-of-releases.md):
  `merged_by` of a pull request merged through the queue is the account that
  added it to the queue, which the release audit checks as before.

## Constraints

- The branch ruleset of the default branch MUST include the rule `merge_queue`
  with `merge_method` `SQUASH`, `max_entries_to_merge` 1, `max_entries_to_build`
  2, `grouping_strategy` `ALLGREEN`, `min_entries_to_merge` 1,
  `min_entries_to_merge_wait_minutes` 0 and `check_response_timeout_minutes` 60.
- `max_entries_to_merge` MUST NOT exceed 1: the merge commit of a release pull
  request MUST be the commit of its own push.
- The required status check of the default branch MUST be `check / required`
  with `integration_id` 15368 and `strict_required_status_checks_policy` set to
  `false`.
- Branches `release/v*` MUST be protected by a separate branch ruleset with the
  same rules as the default branch except `merge_queue`, and with
  `strict_required_status_checks_policy` set to `true`.
- These rulesets MUST NOT have bypass actors. A new repository MUST get its
  signed initial commit before `org/apply.sh` applies them.
- `test.yaml` of every repository with CI MUST trigger on `merge_group` and MUST
  NOT cancel a `merge_group` run in progress.
- Pull requests to the default branch MUST be merged only by adding them to the
  merge queue with `expectedHeadOid` set to the reviewed head. A pull request
  that GitHub does not report as mergeable MUST NOT be added.
- A pull request MUST be removed from the queue before its branch is updated.
- A release pull request MUST be added to the queue only when no other pull
  request of its repository is in the queue, the parent of its head is the tip
  of the base branch and the required check passed on its head.
- A pull request from a fork MUST NOT be added to the queue before its code has
  been reviewed.

## Consequences

- (+) A merge no longer makes the other pull requests stale; the runs that only
  updated a branch, about 28% of the runs on pull requests, disappear.
- (+) Every commit on `main` was checked on top of the commit before it.
- (+) A batch of Renovate pull requests is merged without updating each one by
  hand.
- (−) Every merge waits for a `merge_group` run, also when the pull request is
  already up to date; up to two run at once per repository, which counts against
  the 20 concurrent jobs of the organization.
- (−) A pull request in the queue cannot be pushed to; Renovate's rebase of such
  a pull request fails until it is removed from the queue.
- (−) Release pull requests are serialized with the other pull requests of their
  repository.
- (±) `merged_by` of a pull request is the account that added it to the queue,
  and `github-merge-queue[bot]` pushes to `main`. Push runs on `main`, including
  the release workflow, are triggered by that bot.
- (±) The push run on `main` repeats the check of the `merge_group` run that
  tested the same tree. Dropping it is a later decision.
- (±) `release/v*` keeps strict checks, since the queue cannot cover it; it
  receives few pull requests.

## Alternatives considered

- **Keeping strict checks without a queue.** Keeps the churn of updated
  branches, which is the reason for this decision.
- **Turning strict checks off without a queue.** No updates, but a pull request
  is merged after a check against an old base and can break `main`.
- **Merging several pull requests per push.** Fewer runs, but the release
  workflow misses a release pull request merged in the same push as another one.
- **One ruleset for the default branch and `release/v*`.** GitHub rejects the
  rule `merge_queue` with patterns of ref names.
- **A bypass actor for the initial commit or for urgent merges.** A bypass
  weakens every merge; the initial commit can be made before the rulesets.
- **Renovate merging its own pull requests through the queue.** Possible with
  its platform automerge; left for a later decision.

[workflow-check-adr-0001]:
  https://github.com/flakeyard/workflow-check/blob/main/docs/adr/records/0001-check-pull-requests-and-pushes.md

---
id: FY-0007
title: Branching and releases
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0002, FY-0003, FY-0006]
affects:
  [
    ".github:org/repository/**",
    "*:.github/workflows/**",
    "*:release-please-config.json",
    "*:.release-please-manifest.json",
    "*:CHANGELOG.md",
    "workflow-*:**",
  ]
review_by:
---

## Context

[FY-0006](0006-versioning-and-support.md) requires semantic versions, immutable
tags `vX.Y.Z`, a moving tag `vX` and security fixes for the previous major
version. Consumers update through automated pull requests and read changelogs to
decide whether to merge them. Most repositories are small and change in one
direction at a time; the organization has many of them and few maintainers.

The version of a release has to follow from the changes it contains, and both
the history and the changelog have to be produced without manual work in every
repository.

The organization is on the GitHub Free plan: rulesets are configured per
repository and merge queues are not available. Organization settings already
allow only squash merges with the pull request title as the commit message,
require signed commits on `main` and restrict tags `v*` to the `flakeyard-bot`
GitHub App.

## Decision

### Branches

Development is trunk-based. `main` is the only long-lived branch; every change
reaches it through a pull request merged with a squash merge, so every commit on
`main` is one pull request, signed by GitHub. Branches are deleted after the
merge. A pull request is merged only when its branch is up to date with `main`
and the required checks pass.

When major version `N + 1` is first released, a branch `release/vN` is created
from the last release of major version `N`. Security fixes for that version are
made on `main` first and cherry-picked into `release/vN` through a pull request.
The branch is protected like `main` and archived when support for `vN` ends.

### Commit messages

The title of a pull request follows
[Conventional Commits](https://www.conventionalcommits.org/) and becomes the
message of its squash commit. The type decides the next version: `fix` a patch,
`feat` a minor version, `!` after the type or a `BREAKING CHANGE` footer a major
version; other types (`docs`, `chore`, `ci`, `refactor`, `test`, `build`,
`perf`, `revert`) do not produce a release by themselves. The title is checked
in CI.

### Releases

Which repositories are released follows from
[FY-0006](0006-versioning-and-support.md):

- `atom-*`, `action-*`, `workflow-*` and `image-*` are released: consumers pin
  them by commit or digest and need changelogs.
- `template-*` are released: `copier update` selects versions by git tags.
- `renovate-config` is released: presets are referenced by tag.
- `.github` is not released: it is applied from `main`, nothing is pinned.
- `decisions` is not released: records are versioned by their ids and statuses.

Each released repository uses release-please with a single component. It keeps a
release pull request with the next version and the changelog up to date. The
maintainer merges it when a release is due; release-please then creates the tag
`vX.Y.Z` and its GitHub release, which become immutable. Until `v1.0.0`,
releases are `0.y.z`, breaking changes raise the minor version and releases are
marked as pre-releases.

After the release, the same workflow moves `vX` to the new tag. Both steps run
with a token of `flakeyard-bot`, the only actor allowed to create and move tags
`v*`. The release workflow is a flakeyard reusable workflow, shared by every
released repository.

## Constraints

- Changes MUST reach `main` and `release/v*` only through pull requests merged
  with a squash merge. Merge commits and rebase merges MUST be disabled.
- A pull request MUST be up to date with its base branch and pass the required
  checks before it is merged.
- Pull request titles MUST follow Conventional Commits with the types listed in
  this decision, and CI MUST check them.
- A breaking change of the public API MUST be marked with `!` or a
  `BREAKING CHANGE` footer.
- Branches `release/vN` MUST be created only when major version `N + 1` is
  released, MUST receive only fixes cherry-picked from `main`, and MUST be
  protected by the same ruleset as `main`.
- Released repositories MUST use release-please with one component and the
  shared release workflow. Tags `v*` MUST be created and moved only by
  `flakeyard-bot`.
- Releases MUST be created only from a merged release pull request.
- The moving tag `vX` MUST NOT have a GitHub release of its own.
- `.github` and `decisions` MUST NOT have release tooling.

## Consequences

- (+) Every commit on `main` is a reviewed, signed pull request, and the history
  reads as the changelog.
- (+) Versions and changelogs follow from pull request titles without manual
  work.
- (+) The maintainer decides when a release happens by merging the release pull
  request.
- (+) Only the bot can create or move release tags, so a stolen maintainer token
  cannot rewrite a release.
- (−) A wrong type in a pull request title produces a wrong version; review of
  the title is part of review of the change.
- (−) Without a merge queue, a pull request has to be updated and checked again
  after every merge to `main`.
- (±) Fixes for the previous major version are cherry-picked by hand, which is
  acceptable for security fixes only.

## Alternatives considered

- **Git flow with `develop` and release branches for every release.** More
  branches and merges than small repositories need.
- **Merge commits or rebase merges.** History with intermediate commits that
  were not reviewed one by one, and messages that cannot drive versions.
- **Releasing every merge to `main` automatically.** Consumers would get a
  version and an update pull request for every change, including several in a
  row for one piece of work.
- **semantic-release or changesets.** semantic-release releases on every merge;
  changesets needs a file per change and targets JavaScript packages.
  release-please works from commit messages and batches changes into one release
  pull request.
- **Release branches for every major version from the start.** Extra branches to
  maintain before anyone needs them.
- **Moving `vX` with a separate tool or by hand.** Another place to forget it;
  the release workflow already holds the bot token.

---
id: FY-0022
title: Bootstrap order and temporary exceptions
status: accepted
date: 2026-10-28
supersedes: []
superseded_by: []
related:
  [
    FY-0007,
    FY-0009,
    FY-0012,
    FY-0016,
    FY-0018,
    FY-0019,
    FY-0020,
    FY-0029,
    FY-0030,
    FY-0034,
    FY-0035,
    FY-0038,
  ]
affects:
  [
    "*:.github/workflows/**",
    "*:README.md",
    ".github:org/repository/rulesets/**",
  ]
review_by: 2027-04-30
---

## Context

Accepted decisions require every repository to check pull request titles in CI
([FY-0007](0007-branching-and-releases.md)), to lint workflows with actionlint
and zizmor and to run OpenSSF Scorecard ([FY-0018](0018-supply-chain.md)), to
run its checks on runners with Nix installed by `action-setup-nix`
([FY-0009](0009-ci-on-runners.md)), and to be developed in the dev container
([FY-0012](0012-develop-in-container.md)). These checks are delivered by
flakeyard repositories themselves: `workflow-release`, `action-setup-nix`,
`image-devcontainer`, `workflow-check`, `workflow-scorecard` and the atoms. The
first of them cannot be checked or released by products that do not exist yet.

## Decision

flakeyard is built in this order, each repository released before the next one
depends on it:

1. `workflow-release`, which releases every repository;
2. `action-setup-nix`, which installs Nix in CI;
3. `image-devcontainer`, the development environment;
4. `renovate-config`, whose presets every `renovate.json` extends and
   `workflow-check` checks;
5. `workflow-check`, then `workflow-scorecard`;
6. `action-flake-lock`;
7. the push to the binary cache in `workflow-check`;
8. the atoms, starting with `atom-workflows`;
9. the templates;
10. a decision about a second organization.

[FY-0035](0035-extract-logic-that-has-two-users.md) amends this order: it puts
`action-shell-lib` before the steps that use it and lets duplicated Nix modules
stay until their atoms are released. [FY-0038](0038-roll-out-the-atoms.md) sets
the order of the atoms inside step 8 and answers the questions below about the
atoms.

Steps 6 to 10 were not designed when this decision was made. Each of them gets
an accepted ADR, in this repository or in the repository of the product, before
its code is written. These ADRs answer at least:

- `action-flake-lock`: where its tools come from. Checking `flake.lock` is not
  an API step, so [FY-0030](0030-actions-workflows-and-release-bot.md) does not
  allow the runner's `bash`, `gh` and `jq` for it. Answered in
  [action-flake-lock ADR-0001][action-flake-lock-adr-0001]: `jq` and `gh` come
  from the action's own flake, built with the Nix that is already on `PATH`.
- The atoms: how the last minor release before a major one emits the warning of
  [FY-0016](0016-pin-form.md), where that is recorded, who sets it, and how
  release-please keeps it.
- The cache push: which job pushes, which repositories get the environment
  `cache`, and what pull requests rebuild without it, measured as
  [FY-0019](0019-binary-cache.md) requires.
- The templates: how an update reaches a project generated from a template,
  since the Renovate app of Mend ([FY-0029](0029-renovate-mend.md)) does not run
  the tasks of templates and the presets turn Copier off: who runs
  `copier update`, who opens the pull request, and with which credential it
  changes workflow files.
- A second organization: whether it brings its own App, Mend installation, cache
  and preset, or flakeyard products take the owner as a parameter.

Until a repository's checks can come from released flakeyard products, the
missing ones are replaced:

- workflow linting, formatting, spelling and the other checks of its atoms run
  locally, on the host or in a disposable container, with the same tools, before
  every pull request;
- pull request titles are checked by the reviewer;
- rulesets require no status checks until `workflow-check` is released;
- OpenSSF Scorecard is added from `workflow-scorecard` together with
  `workflow-check`, so `workflow-scorecard` is released right after it.

Every repository under an exception says so in a "Bootstrap" section of its
README, naming the missing checks and the product that will provide them. An
exception ends when that product is released: the repository adopts it in its
next pull request and removes the section.

## Constraints

- Repositories MUST be bootstrapped in the order of this decision.
- Each step from `action-flake-lock` on MUST have an accepted ADR, in this
  repository or in the repository of the product, before its code is written.
- A repository MAY omit a check required by another `FY` decision only while the
  flakeyard product that provides it is unreleased, and MUST then run the same
  tools locally before every pull request.
- Every such repository MUST list its omitted checks and their providers in a
  "Bootstrap" section of its README.
- A repository MUST adopt a check in the first pull request after its provider
  is released.
- This decision MUST be deprecated once every repository has adopted every
  check.

## Consequences

- (+) The first repositories can be released without waiting for products that
  depend on them.
- (+) Every gap is visible in the repository it affects and has an end.
- (−) Until `workflow-check` exists, broken workflows and wrong pull request
  titles are caught only by people.
- (±) The first releases of flakeyard are `0.y.z` pre-releases, so a missed
  defect breaks no compatibility promise.

## Alternatives considered

- **Temporary hand-written checks in every repository.** Would close the gaps
  earlier, but each would be thrown away once the products exist, and they would
  use tools without the pinning the decisions require.
- **Building everything before the first release.** No exceptions, but nothing
  could be tested in real use until the end.

[action-flake-lock-adr-0001]:
  https://github.com/flakeyard/action-flake-lock/blob/main/docs/adr/records/0001-check-flake-lock.md

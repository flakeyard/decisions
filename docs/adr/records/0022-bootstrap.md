---
id: FY-0022
title: Bootstrap order and temporary exceptions
status: accepted
date: 2026-10-28
supersedes: []
superseded_by: []
related: [FY-0007, FY-0009, FY-0012, FY-0018, FY-0020]
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
`image-devcontainer`, `workflow-check` and the atoms. The first of them cannot
be checked or released by products that do not exist yet.

## Decision

flakeyard is built in this order, each repository released before the next one
depends on it:

1. `workflow-release`, which releases every repository;
2. `action-setup-nix`, which installs Nix in CI;
3. `image-devcontainer`, the development environment;
4. `workflow-check`, the atoms and `renovate-config`.

Until a repository's checks can come from released flakeyard products, the
missing ones are replaced:

- workflow linting, formatting, spelling and the other checks of its atoms run
  locally, on the host or in a disposable container, with the same tools, before
  every pull request;
- pull request titles are checked by the reviewer;
- rulesets require no status checks until `workflow-check` is released;
- OpenSSF Scorecard is added together with `workflow-check`.

Every repository under an exception says so in a "Bootstrap" section of its
README, naming the missing checks and the product that will provide them. An
exception ends when that product is released: the repository adopts it in its
next pull request and removes the section.

## Constraints

- Repositories MUST be bootstrapped in the order of this decision.
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

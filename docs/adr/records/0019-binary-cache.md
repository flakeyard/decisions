---
id: FY-0019
title: Binary cache
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0005, FY-0009, FY-0017, FY-0018]
affects:
  [
    "*:.github/workflows/**",
    "workflow-*:**",
    "action-setup-nix:**",
    "image-devcontainer:**",
    ".github:org/**",
  ]
review_by: 2027-10-22
---

## Context

CI reads from public binary caches everywhere and pushes to the flakeyard cache
only from protected branches ([FY-0009](0009-ci-on-runners.md)). Which cache,
what to push and whether it is worth it were left open until measured.

Probes with an empty store and only `cache.nixos.org` as a substituter
established:

- Building the dev container image fetches about 300 MiB from `cache.nixos.org`
  and builds 13 derivations of its own, 10 MiB per architecture, almost all of
  it the layer with the Nix database.
- The checks of a project built from atoms fetch about 316 MiB of tools and
  build outputs of 7 KiB in total: configuration files and empty success
  markers.
- `nix flake check` skips a check whose output can be substituted, without
  downloading it or any of its tools. With the markers in a cache, the same
  checks fetched 113 MiB instead of 316 MiB; the rest was the formatting check,
  which treefmt-nix makes unsubstitutable and [FY-0017](0017-atom-contract.md)
  makes substitutable.
- Cachix does not store paths that `cache.nixos.org` has, evicts the least
  recently used paths at its limit, and offers 5 GB for open source projects.
  Its GitHub action pushes only paths built in the job.
- The free tier of `magic-nix-cache` ended in 2025;
  `nix-community/cache-nix-action` stores the Nix store in the GitHub Actions
  cache, which holds 10 GB per repository and evicts entries unused for 7 days.

A marker in the cache is an assertion that a check passed: whoever can push can
make a failing check pass for everyone who trusts the cache.

## Decision

flakeyard uses one public Cachix cache, `flakeyard`, on the open source plan.

CI of flakeyard repositories substitutes from `cache.nixos.org` and from
`flakeyard`, and pushes what it built for checks and packages on `main` and
`release/v*` after all checks pass, as [FY-0009](0009-ci-on-runners.md)
requires. The main gain is the markers of checks: a pull request that does not
touch a domain substitutes its check and downloads none of its tools. Paths of
the dev container image are not pushed: the image is published to the container
registry, and its own derivations rebuild in seconds.

The GitHub Actions cache through `nix-community/cache-nix-action` may be used
for reruns of pull requests. Whether it pays for its save and restore time is
measured on the first real workflows before any workflow requires it.

Consumers do not gain from the flakeyard cache today: their checks have their
own sources, and their tools come from `cache.nixos.org`. The cache stays opt-in
for them ([FY-0009](0009-ci-on-runners.md)); `action-setup-nix` and the image
document how to enable it.

This decision is revisited when flakeyard builds toolchains that
`cache.nixos.org` does not have, when the cache exceeds half of its limit, or
when the terms of the Cachix plan change.

## Constraints

- flakeyard MUST use the Cachix cache `flakeyard` as its only binary cache
  besides `cache.nixos.org`.
- Only outputs of checks and packages built by the repository MAY be pushed.
  Paths of the dev container image MUST NOT be pushed.
- Check outputs MUST be pushed only by the push job of
  [FY-0009](0009-ci-on-runners.md), after all checks of the same commit passed.
- The GitHub Actions cache, if used, MUST use `nix-community/cache-nix-action`,
  allowed by organization settings, and MUST NOT be restored by release jobs.
- Workflows MUST NOT require the GitHub Actions cache until its benefit has been
  measured and recorded.
- The cache size MUST be reviewed when it exceeds half of its limit.

## Consequences

- (+) Pull requests check only the domains they change and download only the
  tools of those domains.
- (+) The cache stays far below its limit: markers have no size, and the image
  is not pushed.
- (−) The push token can turn any failing check into a passing one for flakeyard
  CI; it is protected only by the branch-restricted environment of
  [FY-0009](0009-ci-on-runners.md).
- (−) The open source plan is a third-party offer that may change.
- (±) Consumers get no benefit until flakeyard ships builds that are not in
  `cache.nixos.org`.

## Alternatives considered

- **No binary cache.** Nothing to trust or maintain, but every pull request runs
  and downloads every check.
- **Attic or another self-hosted cache.** No third-party limits, but a server
  and storage to run and secure.
- **A Nix cache in the container registry.** Free storage next to the image, but
  tooling for it is experimental.
- **FlakeHub Cache or magic-nix-cache.** The former is paid, the free tier of
  the latter has ended.
- **Pushing the image's paths too.** Saves seconds of rebuild at the cost of 10
  MiB per architecture and release.

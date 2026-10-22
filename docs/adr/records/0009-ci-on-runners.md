---
id: FY-0009
title: Run CI on hosted runners
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0004, FY-0005, FY-0007, FY-0008]
affects:
  [
    "*:.github/workflows/**",
    "action-setup-nix:**",
    "workflow-*:**",
    ".github:org/**",
  ]
review_by:
---

## Context

Developers work in the dev container image
([FY-0005](0005-dev-container-image.md)), where the Nix sandbox is off and the
store lives in the container. CI has different needs: builds must be isolated,
results must be reproducible on every run, and only what a check needs should be
downloaded. Atoms already separate tools for checks from tools for people:
checks carry their own dependencies, and developer tools live only in
development shells ([FY-0004](0004-atom-contract.md)).

GitHub-hosted runners are free for public repositories on Linux x86_64 and
arm64. Jobs in a container on these runners run without the privileges the Nix
sandbox needs.

Builds that are not in `cache.nixos.org` are rebuilt on every run unless they
are cached. A binary cache signs what it stores with one key: whoever can push
can place any store path under that key, and everyone who trusts the key
downloads it.

## Decision

### Runners and Nix

CI runs directly on GitHub-hosted runners, not in the dev container image. Nix
is installed by `action-setup-nix` in multi-user mode with the sandbox on and no
fallback, with the same Nix implementation and minor version as the image. Every
system a repository claims is built natively on a runner of that system; there
is no emulation.

CI runs only checks and apps: `nix flake check` or builds of `checks.<system>`,
and `nix run .#<app>`. It never enters a development shell, so it downloads only
what checks and apps depend on.

### Caching

There are three levels of caching:

- **Public binary caches, read everywhere.** Every run substitutes from
  `cache.nixos.org` and, in flakeyard repositories, from the flakeyard Cachix
  cache. Consumers opt in to the flakeyard cache; by default `action-setup-nix`
  trusts only `cache.nixos.org`.
- **Push to Cachix only from protected branches.** After all checks pass on
  `main` or `release/v*`, a separate job pushes the paths the repository built
  itself. The push token is a secret of a GitHub environment that only these
  branches can deploy to, and only the push step sees it.
- **GitHub Actions cache per branch.** Pull request runs save and restore the
  store paths they built, so repeated runs on one branch do not rebuild them.
  The cache is scoped by GitHub: a pull request reads its own and its base
  branch's entries and writes only its own. Release and publishing jobs restore
  no GitHub Actions cache.

### Required checks

Every repository ends its CI with one job, `required`, that depends on all other
jobs and fails if any of them failed or was cancelled. It is the only required
status check, so organization settings name the same check in every repository
regardless of its matrix.

## Constraints

### Environment

- CI MUST run on GitHub-hosted runners and MUST NOT run inside the dev container
  image, except for the image's own smoke tests.
- Nix in CI MUST be installed by `action-setup-nix` with `sandbox = true` and
  `sandbox-fallback = false`, and with the Nix implementation and minor version
  of the image.
- Every system a repository claims MUST be built natively. CI MUST NOT emulate
  other architectures.
- CI MUST run only checks and apps and MUST NOT use `nix develop` or development
  shells.
- CI MUST NOT set `accept-flake-config`.

### Binary cache

- Substituters other than `cache.nixos.org` MUST be opt-in for consumers.
- Pushing to the binary cache MUST happen only from `main` and `release/v*`,
  after all checks pass, in a separate job.
- The push token MUST be a secret of a GitHub environment restricted to `main`
  and `release/v*`, and MUST be visible only to the push step.
- Only store paths built by the repository MAY be pushed; paths substituted from
  `cache.nixos.org` MUST NOT be pushed.
- Pull requests MUST NOT have access to the push token.

### Branch cache

- Pull request runs MAY save and restore store paths in the GitHub Actions
  cache.
- Release and publishing jobs MUST NOT restore the GitHub Actions cache.

### Status checks

- Every repository with CI MUST have a final job `required` that depends on all
  other jobs and fails when any of them did not succeed.
- `required` MUST be the only required status check of the branch rulesets.

## Consequences

- (+) Builds in CI are isolated, while developers keep a container without
  sandbox privileges.
- (+) CI downloads only what checks need; developer tools never reach it.
- (+) Pull requests reuse everything built on `main` and their own previous
  runs, without any way to write into the shared cache.
- (+) One required check name works for every repository.
- (−) Paths changed by a pull request are rebuilt in every new pull request
  until they reach `main`.
- (−) The GitHub Actions cache is limited to 10 GB per repository and evicts
  entries after 7 days without use.
- (−) Each claimed system needs runners of its own, and systems without hosted
  runners cannot be claimed.
- (±) CI and the image must be updated together to keep the same Nix version.

## Alternatives considered

- **CI in the dev container image.** The same environment as developers, but no
  sandbox, a pulled image on every job and development tools in CI.
- **Emulated builds of other architectures.** No extra runners, but slow and
  different from what users run.
- **Push to Cachix from every branch.** Faster pull requests, but unreviewed
  code would hold the key that consumers trust.
- **A second binary cache for pull requests.** Shares builds between pull
  requests without the main key, at the cost of another cache, key and trust
  configuration; not needed while builds of flakeyard are small.
- **Required checks per job.** Every matrix change would need a settings change
  in every repository.

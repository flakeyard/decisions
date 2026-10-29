---
id: FY-0026
title: Follow a stable nixpkgs release
status: accepted
date: 2026-10-29
supersedes: []
superseded_by: []
related: [FY-0005, FY-0009, FY-0016, FY-0017, FY-0023, FY-0024]
affects:
  [
    "image-devcontainer:flake.nix",
    "image-devcontainer:flake.lock",
    "action-setup-nix:versions.sh",
    "workflow-*:flake.nix",
    "action-*:flake.nix",
    "atom-*:dev/**",
    "template-*:**",
    "renovate-config:**",
  ]
review_by:
---

## Context

flakeyard pins nixpkgs in the dev container image, in the tool flakes of its
actions and workflows, in the development and test partitions of its atoms, and
in the projects its templates generate ([FY-0016](0016-pin-form.md)). Atoms
themselves take every tool from the consumer's nixpkgs
([FY-0017](0017-atom-contract.md)).

`nixos-unstable` advances when a small set of tests passes; other packages may
stay broken there for weeks. Building the image showed it: `sbomnix`, needed for
the image's SBOM, failed to build on `nixos-unstable` from 2026-10-16 on, which
would have blocked every update of the image, security fixes included.

A stable branch, `nixos-YY.MM`, receives only fixes and security updates. A new
one is released every May and November and is supported for about seven months,
until the end of the following June or December. `nixos-26.05` contains
`nixVersions.nix_2_35`, the Nix minor version of the image and of
`action-setup-nix` ([FY-0009](0009-ci-on-runners.md),
[FY-0023](0023-setup-nix.md)), and builds every tool the image needs on both
architectures.

## Decision

### Branch

Every nixpkgs pin of flakeyard follows the same stable branch, written in the
form of FY-0016 with the branch as `ref`:

```nix
url = "git+https://github.com/NixOS/nixpkgs?ref=nixos-26.05&rev=<commit>&shallow=1";
```

Renovate proposes the new commits of that branch. Templates generate the same
branch for new projects; projects may follow another one.

### Switching

All repositories move to the next stable branch within one month after its
release and before the current one reaches its end of life. Each repository
records the end of life of its branch, and a weekly job that is not a required
check warns from 30 days before that date and fails after it.

A switch is a deliberate pull request in each repository, released by the level
of its effect on the public API ([FY-0024](0024-versioning-and-support.md)). It
keeps the Nix minor version when the new branch still has it; a new Nix minor
version is a separate change.

### Nix minor version

The image and `action-setup-nix` move to a new Nix minor version only when the
stable branch the image follows contains it: first `action-setup-nix` is
released with it, then the image. Between the two releases, consumers may run
different minor versions in CI and in the container until both updates are
merged.

### Broken tools

When a tool flakeyard needs is broken on the stable branch, the repository takes
that tool alone from a second, separately pinned nixpkgs input, says so in its
README and removes the input once the branch is fixed.

## Constraints

- Every nixpkgs pin in flakeyard repositories MUST follow the same stable
  `nixos-YY.MM` branch in the form of FY-0016. Templates MUST generate that
  branch.
- Repositories MUST switch to the next stable branch within one month after its
  release and before the current branch reaches its end of life.
- Every repository that pins nixpkgs MUST record the end of life of its branch
  and run a weekly check, not a required one, that warns from 30 days before it
  and fails after it.
- A branch switch MUST be its own pull request and MUST NOT change the Nix minor
  version in the same change.
- `action-setup-nix` and the image MUST adopt a new Nix minor version only when
  the stable branch of the image contains it, `action-setup-nix` first.
- A second nixpkgs input MAY be used only for a tool that is broken on the
  stable branch, MUST be documented in the README and MUST be removed once the
  branch is fixed.

## Consequences

- (+) Updates within a branch are fixes; a broken unrelated package rarely
  blocks security updates.
- (+) Every repository and generated project uses the same package set.
- (−) Two switches a year in every repository, each checked by its CI.
- (−) Tool versions are older than on `nixos-unstable`, and a new Nix minor
  version arrives only with a stable branch that has it.
- (±) A window in which CI and the container run different Nix minor versions
  opens with every Nix minor update.

## Alternatives considered

- **`nixos-unstable`.** Newest packages, but any broken package flakeyard
  depends on blocks every update until it is fixed upstream.
- **`nixos-unstable` with a second input for tools.** Keeps the image current
  and isolates tool breakage, but doubles the pins to maintain everywhere and
  still breaks when the image's own packages fail.
- **An older `nixos-unstable` commit.** Works until the next update needs a
  commit where everything builds again.

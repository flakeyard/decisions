---
id: FY-0036
title: Check the end of life of nixpkgs in flake.lock
status: accepted
date: 2026-11-07
supersedes: [FY-0026]
superseded_by: []
related: [FY-0009, FY-0016, FY-0017, FY-0024, FY-0030, FY-0034, FY-0035]
affects:
  [
    "image-devcontainer:flake.nix",
    "image-devcontainer:flake.lock",
    "action-setup-nix:versions.sh",
    "action-flake-lock:**",
    "workflow-check:.github/workflows/check.yaml",
    "*:.github/workflows/test.yaml",
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
one is released every May and November. `nixos-YY.05` is supported until the end
of 31 December of the same year and `nixos-YY.11` until the end of 30 June of
the next year, as the announcement of NixOS 26.05 and endoflife.date state; the
`-small` variant of a branch ends with it. `nixos-26.05` contains
`nixVersions.nix_2_35`, the Nix minor version of the image and of
`action-setup-nix` ([FY-0009](0009-ci-on-runners.md), [action-setup-nix
ADR-0001][action-setup-nix-adr-0001], [image-devcontainer
ADR-0001][image-devcontainer-adr-0001]), and builds every tool the image needs
on both architectures.

[FY-0026](0026-stable-nixpkgs.md) made every repository record the end of life
of its branch in a file `nixpkgs-eol` and run a weekly job of its own that
checks it. That is the same logic in every repository, which
[FY-0035](0035-extract-logic-that-has-two-users.md) forbids, and the file can
disagree with the branch that `flake.lock` actually pins. The branch alone gives
the date, and `action-flake-lock` already reads `flake.lock` in every repository
with a flake.

This record replaces FY-0026. What changes is where the end of life comes from
and which job checks it; everything else is unchanged.

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
release and before the current one reaches its end of life.

A switch is a deliberate pull request in each repository, released by the level
of its effect on the public API ([FY-0024](0024-versioning-and-support.md)). It
keeps the Nix minor version when the new branch still has it; a new Nix minor
version is a separate change.

### End of life

`action-flake-lock` derives the end of life from the branch that `flake.lock`
records for each root input that is nixpkgs, as designed in [action-flake-lock
ADR-0002][action-flake-lock-adr-0002]:

- `nixos-YY.05` and `nixos-YY.05-small`: the end of 31 December 20YY, UTC;
- `nixos-YY.11` and `nixos-YY.11-small`: the end of 30 June of the year after
  20YY, UTC;
- any other branch: a notice that its end of life is not known.

From 30 days before that date, the action reports a warning annotation and the
check passes. After the date, it reports a violation only in runs of the
`schedule` event; in other events it reports a warning, so the date never blocks
a pull request. The end of life stays a weekly check, not a required one.

`workflow-check` runs `action-flake-lock` ([workflow-check
ADR-0003][workflow-check-adr-0003]). Both have the input `nixpkgs-eol`, on by
default, that turns the check of the end of life off.

The support policy of NixOS is part of the code of `action-flake-lock`. When it
changes, a release of the action follows it.

### Weekly run

Every repository with a flake runs `workflow-check` on a weekly `schedule`. A
repository that checks a lock outside it runs `action-flake-lock` itself on that
schedule, as `image-devcontainer` does and as `action-flake-lock` does for its
own root lock.

The file `nixpkgs-eol` and the separate weekly job of FY-0026 are no longer
required. They are removed once every repository runs a release of
`workflow-check` with this rule.

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

### Branches and Nix

- Every nixpkgs pin in flakeyard repositories MUST follow the same stable
  `nixos-YY.MM` branch in the form of FY-0016. Templates MUST generate that
  branch.
- Repositories MUST switch to the next stable branch within one month after its
  release and before the current branch reaches its end of life.
- A branch switch MUST be its own pull request and MUST NOT change the Nix minor
  version in the same change.
- `action-setup-nix` and the image MUST adopt a new Nix minor version only when
  the stable branch of the image contains it, `action-setup-nix` first.
- A second nixpkgs input MAY be used only for a tool that is broken on the
  stable branch, MUST be documented in the README and MUST be removed once the
  branch is fixed.

### Dates

- `action-flake-lock` MUST derive the end of life of every root nixpkgs input
  from its branch in `flake.lock`: the end of 31 December 20YY UTC for
  `nixos-YY.05` and `nixos-YY.05-small`, the end of 30 June of the next year UTC
  for `nixos-YY.11` and `nixos-YY.11-small`. For any other branch it MUST report
  a notice that the end of life is not known.
- From 30 days before the end of life, the action MUST report a warning and MUST
  NOT fail.
- After the end of life, the action MUST report a violation in runs of the
  `schedule` event and only a warning in runs of any other event.
- `action-flake-lock` and `workflow-check` MUST have the input `nixpkgs-eol`,
  `true` by default, that turns this check off.
- A change of the support policy of NixOS MUST reach the check only through a
  release of `action-flake-lock`.

### Scheduled runs

- Every repository with a flake MUST run `workflow-check` on a weekly
  `schedule`, or `action-flake-lock` itself where it checks a lock that
  `workflow-check` does not, with `nixpkgs-eol` on.
- The file `nixpkgs-eol` and a separate weekly job for the end of life MUST NOT
  be required, and MUST be removed once every repository runs a release of
  `workflow-check` with this rule.

## Consequences

- (+) Updates within a branch are fixes; a broken unrelated package rarely
  blocks security updates.
- (+) Every repository and generated project uses the same package set.
- (+) The end of life follows the branch that is actually locked, with no file
  to keep in sync and no copy of the check in every repository.
- (+) Pull requests are never blocked by the date; the weekly run fails after
  it.
- (−) Two switches a year in every repository, each checked by its CI.
- (−) Tool versions are older than on `nixos-unstable`, and a new Nix minor
  version arrives only with a stable branch that has it.
- (−) A change of the support policy of NixOS needs a release of
  `action-flake-lock` and its update in every repository.
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
- **The file `nixpkgs-eol` in every repository, as in FY-0026.** The date is
  written by hand, can disagree with `flake.lock`, and every repository carries
  its own job to check it.
- **The end of life fetched from endoflife.date.** Follows policy changes
  without a release, but the check would need the network and an outside service
  would decide whether a scheduled run fails.
- **A violation after the date in every event.** Pull requests would fail on a
  date, also those that make the switch possible.

[action-setup-nix-adr-0001]:
  https://github.com/flakeyard/action-setup-nix/blob/main/docs/adr/records/0001-set-up-nix-on-ci-runners.md
[image-devcontainer-adr-0001]:
  https://github.com/flakeyard/image-devcontainer/blob/main/docs/adr/records/0001-dev-container-image.md
[action-flake-lock-adr-0002]:
  https://github.com/flakeyard/action-flake-lock/blob/main/docs/adr/records/0002-end-of-life-of-nixpkgs.md
[workflow-check-adr-0003]:
  https://github.com/flakeyard/workflow-check/blob/main/docs/adr/records/0003-shell-library-and-end-of-life.md

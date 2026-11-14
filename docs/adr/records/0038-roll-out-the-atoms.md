---
id: FY-0038
title: Roll out the atoms
status: accepted
date: 2026-11-12
supersedes: []
superseded_by: []
related:
  [
    FY-0012,
    FY-0016,
    FY-0017,
    FY-0019,
    FY-0022,
    FY-0024,
    FY-0033,
    FY-0034,
    FY-0035,
    FY-0037,
  ]
affects:
  [
    "atom-*:**",
    "*:flake.nix",
    "*:dev/flake.nix",
    "*:.github/workflows/test.yaml",
    "workflow-check:.github/workflows/check.yaml",
    "action-flake-lock:**",
  ]
review_by:
---

## Context

[FY-0022](0022-bootstrap.md) builds the atoms in step 8, starting with
`atom-workflows`, and asks how the last minor release before a major one emits
the warning of [FY-0016](0016-pin-form.md), where that is recorded, who sets it
and how release-please keeps it. [FY-0017](0017-atom-contract.md) lists ten base
atoms. [FY-0035](0035-extract-logic-that-has-two-users.md) lets the copies of
the Nix formatting and shell checks in `workflow-check`, `workflow-scorecard`,
`renovate-config`, `image-devcontainer` and `action-flake-lock` stay until
`atom-nix`, `atom-shell` and `atom-workspace` are released.

Facts that shape the rollout:

- `action-setup-nix`, `action-shell-lib`, `workflow-release`, `.github` and
  `decisions` have no flake, so their workflow linting, formatting and spelling
  run only locally ([workflow-check ADR-0001][workflow-check-adr-0001]).
- The flake module of git-hooks.nix adds `checks.pre-commit` over the whole
  source of the project unless it is switched off, and only `atom-workspace` or
  the project may switch it off (FY-0017).
- A new released repository gets the release bot's key only by a rotation of the
  key ([FY-0033](0033-custody-of-the-release-bot-key-and-audit-of-releases.md)).
- Renovate treats flake inputs as git references without versions (FY-0016): an
  update of an atom shows two commits, no version and no `breaking` label.
- A warning printed during evaluation appears only in the log of the job that
  evaluates; in a passing run nobody reads it.
- Releases `0.y.z` are previews without guarantees
  ([FY-0024](0024-versioning-and-support.md)), and `workflow-release` keeps the
  moving tag `v0` at the newest of them.
- FY-0017 requires the tests of every atom to run on a schedule against current
  upstream revisions. The same job in ten repositories is a copy that FY-0035
  forbids.

## Decision

### Repositories

The ten base atoms of FY-0017 are created together, in one batch, after this
record and [FY-0037](0037-atoms-ship-default-configs-that-projects-replace.md)
are accepted, so the release bot's key is rotated once for all of them. A new
atom repository gets its workflows with its first code, so that it has no
failing runs while it is empty. Each atom gets an accepted `ADR-0001` in its
repository before its code is written (FY-0022).

### Order

Step 8 of FY-0022 builds the atoms in this order:

1. `atom-workflows`;
2. `atom-workspace`;
3. `atom-nix` and `atom-shell`, after which every repository with a copy allowed
   by FY-0035 adopts them in its next pull request;
4. `atom-markdown`, `atom-text`, `atom-data-formats`, `atom-reuse`,
   `atom-secrets` and `atom-commits`.

Until `atom-workspace` is released, a repository that adopts `atom-workflows`
switches off `checks.pre-commit` itself, in one line. That line goes away when
the repository adopts `atom-workspace`.

### Development flakes

A repository whose root flake is consumed by others, an atom or
`action-flake-lock`, keeps its development flake in `dev/`, as FY-0012 lets
atoms do. Every other repository uses its root flake as its development flake.
`action-setup-nix`, `action-shell-lib`, `workflow-release`, `.github` and
`decisions` get a root `flake.nix` that only imports atoms and, where the
repository's tests need tools no atom provides, declares them in
`devShells.flakeyard-project` (FY-0017). Every repository lists the checks of
its atoms in the input `expect-checks` of `workflow-check`, so a removed import
fails CI.

`image-devcontainer` is the one exception. By the owner's decision of
2026-11-02, made with
[FY-0034](0034-check-every-repository-with-workflow-check.md), it builds its
flake checks in its own job `image`, before the smoke tests. It passes an empty
`flake` and no `expect-checks` to `workflow-check`, so `workflow-check` builds
no flake checks; the `nix flake check` of the job `image` builds the checks of
its atoms.

### Versions and updates

Atoms are released by `workflow-release` with the moving tag, starting at
`v0.1.0`. While they are `0.y.z`, only flakeyard's own repositories use them,
with `ref=v0`. An update that breaks a repository is adapted on the branch of
its Renovate pull request. `v1.0.0` of every atom that the templates use is
released before the templates (step 9 of FY-0022).

### Next major version

The warning of FY-0016 is recorded in the file `nix/next-major.nix` of the atom:
the next major version and the path of its migration guide in the repository.
The maintainer adds the file in a minor release, made by a pull request of type
`feat` right before the first breaking change is merged; the breaking pull
request removes it. release-please keeps nothing: the file is ordinary content.
While the file exists, the atom warns at evaluation with `lib.warn` of the
consumer's nixpkgs. A test of the atom checks the warning and that the version
in the file is above the current major version.

So that the warning is also seen in CI, `action-flake-lock` warns, with an
annotation, about every flakeyard input whose repository has released a major
version above the one named by its `ref`.

### Scheduled tests

`workflow-check` takes a map of inputs to override while it builds the checks.
The `test.yaml` of every atom passes the current branches of nixpkgs,
treefmt-nix and git-hooks.nix on its weekly schedule and nothing otherwise. The
lock file stays unchanged and is still checked.

### Binary cache

Atoms push nothing to the binary cache until step 7 of FY-0022 is decided.

## Constraints

- The ten base atom repositories MUST be created in one batch, after FY-0037 and
  this record are accepted.
- An atom repository MUST NOT contain workflows before its first code.
- Atoms MUST be built in the order of this record. FY-0022 step 8 is amended
  accordingly.
- A repository MAY switch off `checks.pre-commit` itself only until it adopts
  `atom-workspace`.
- A repository whose root flake is consumed by others MUST keep its development
  flake in `dev/`; every other repository MUST use its root flake.
- Every repository MUST list the checks of the atoms it imports in
  `expect-checks`, except `image-devcontainer`. `image-devcontainer` MUST pass
  an empty `flake` and no `expect-checks` to `workflow-check`, and the job
  `image` of its `test.yaml` MUST run `nix flake check` on its flake, which
  imports the atoms.
- Atoms MUST be released with the moving tag `vX`. Repositories outside
  flakeyard MUST NOT be pointed to `0.y.z` releases by flakeyard templates or
  documentation.
- Every atom that the templates use MUST reach `v1.0.0` before the templates are
  released.
- The last minor release of a major version MUST contain `nix/next-major.nix`,
  and the atom MUST warn at evaluation while the file exists.
- `action-flake-lock` MUST warn about flakeyard inputs that have a newer major
  version.
- The scheduled tests of atoms MUST run through `workflow-check` and MUST NOT be
  written into the atoms' workflows.

## Consequences

- (+) One rotation of the bot's key for all base atoms, and no empty
  repositories with failing runs.
- (+) Every flakeyard repository gets workflow linting in CI with the first
  atom, and the copies of FY-0035 go with the third step.
- (+) A newer major version is announced in every pull request of every
  consumer, not only in an evaluation log.
- (+) The scheduled run against current upstreams exists once.
- (−) Five repositories get a flake whose only job is to lint them.
- (−) Updates of atoms during `0.y.z` may break flakeyard's repositories without
  a label; CI and the release notes are the only signals.
- (±) `atom-workspace` comes before the atoms with the most users, so the
  development shell exists before the copies are removed.

## Alternatives considered

- **Create each atom repository when its turn comes.** One rotation of the key
  per batch.
- **`atom-nix` and `atom-shell` second.** The copies would go earlier, but every
  adopting repository would carry the whole-source `checks.pre-commit` and no
  development shell until `atom-workspace`.
- **A scheduled job in each atom.** The same logic ten times.
- **The evaluation warning alone.** Seen only by people who read evaluation
  logs.
- **`v1.0.0` from the start.** Commits to the public API before any project used
  it.

[workflow-check-adr-0001]:
  https://github.com/flakeyard/workflow-check/blob/main/docs/adr/records/0001-check-pull-requests-and-pushes.md

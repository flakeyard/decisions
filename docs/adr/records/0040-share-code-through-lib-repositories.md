---
id: FY-0040
title: Share code through lib- repositories
status: accepted
date: 2026-11-16
supersedes: []
superseded_by: []
related: [FY-0003, FY-0016, FY-0017, FY-0022, FY-0024, FY-0035, FY-0038]
affects:
  [
    "lib-*:**",
    "atom-*:flake.nix",
    "atom-*:flake.lock",
    "atom-*:nix/**",
    "atom-*:dev/**",
    "action-flake-lock:**",
    "workflow-check:.github/**",
    "renovate-config:**",
    ".github:org/**",
    ".github:VERSIONING.md",
  ]
review_by:
---

## Context

[FY-0035](0035-extract-logic-that-has-two-users.md) requires that logic with two
users lives in one place. [FY-0017](0017-atom-contract.md) forbade atoms to
declare a flakeyard repository as an input, so the Nix code that every atom
needs had nowhere to go. With six atoms built, the copies are:

- `nix/follows.nix`, the check of `follows`, identical in five atoms;
  `nix/warning.nix`, the warning of the next major version, identical in five;
  the guarded import of upstream modules in every `nix/default.nix`;
- the choice of a configuration file ([FY-0037][fy-0037]): a project's file at a
  standard path or the atom's default, as its own store path, in three atoms
  with three implementations that already differ: one takes the first of two
  project files silently, the others fail on two files and on forms they cannot
  pass;
- in the tests, `dev/nix/facts.nix`, identical in all six, and the fixture
  builder and the tests of `follows`, the warning and atom-workspace, which
  differ only in their parameters.

Ten atoms would multiply this. The owner decided on 2026-11-16 that atoms may
depend on flakeyard repositories to remove the copies.

A design review showed:

- If projects had to `follows` a shared input of the atoms, every atom would run
  with the revision the project chose rather than the one it was tested with. A
  newer major version of the library would break atoms with an unrelated error,
  a major release would have to be taken by every atom of a project at once, and
  a minimum version could not be tested, since Renovate keeps the tests on the
  newest revision.
- When an atom pins the library itself, `nix flake update <atom>` in a project
  also updates the atom's nested lock entry of the library to the revision the
  atom's lock names (verified with Nix 2.35.2). A project's lock then holds one
  entry of the library per atom, with one store path when the revisions are
  equal.
- `action-flake-lock` fails on any source locked twice, also in nested entries,
  unless the project lists it in `allow-duplicates`.
- `workflow-check` checks the lock of the flake it is given, which for an atom
  is `dev/flake.lock`; the root `flake.lock` of an atom is not checked.
- A flake output other than the standard ones, such as `version`, makes
  `nix flake check` warn about an unknown output.
- GitHub does not redirect `uses:` of an action whose repository was renamed;
  workflows that use it fail with `repository not found` ([GitHub
  documentation][github-rename]).

## Decision

### lib- repositories

A `lib-<area>` repository holds code that the flakeyard repositories of one area
share: `lib-atom` for atoms, `lib-actions` for actions and reusable workflows.
The prefix names the role, not the delivery mechanism. The code may be in any
language, and the README states how the repository is consumed and pinned. A
`lib-` repository is not consumed by projects directly.

### lib-atom

`lib-atom` is a flake with no inputs. Its output `lib` is a set of plain
functions that take what they need, such as `lib` of the consumer's nixpkgs and
the path of the project's source, as arguments:

- the check of `follows` for an upstream flake, with the message that names the
  missing line, and the import of the upstream module behind it;
- the warning of the next major version from `nix/next-major.nix` (FY-0038);
- the choice of a configuration file: the standard names at the project's root,
  an error when more than one exists, an error for forms the atom cannot pass
  explicitly, and the chosen file as a store path of its own (FY-0037);
- `version`, read from `.release-please-manifest.json`.

Logic that only one atom needs, such as the configurations that typos reads from
`pyproject.toml`, stays in that atom.

The helpers of the atoms' tests live in `internal/testing/` of `lib-atom`: the
facts of the dev flakes, the builder of fixture projects and the generators of
the tests of `follows`, of the warning and of the glue of atom-workspace. Atoms'
dev flakes import the directory from the source of `lib-atom`; it is not part of
the public API.

### Atoms pin lib-atom

Every atom declares `lib-atom` as an input of its public flake and pins it in
its own `flake.lock`. Projects do not declare it and write no `follows` for it;
every atom runs with the revision it was released with, and the major versions
of `lib-atom` are invisible to projects. An atom's dev flake reads the revision
of `lib-atom` from the atom's root `flake.lock` and fetches that revision, so
that an atom has one pin of `lib-atom`.

`lib-atom` is released before the atoms take a new revision. Renovate proposes
its updates in atoms with the commit type `fix`, since a release of the atom
delivers the new code to projects. `lib-atom` checks every change against the
released atoms in its own tests, and the weekly scheduled tests of atoms
override their `lib-atom` with its current branch (FY-0038).

### Locks

`action-flake-lock` accepts nested entries of `github.com/flakeyard/lib-*`
locked more than once and warns about newer major versions only for the
project's own inputs. `workflow-check` also checks the root `flake.lock` of a
repository whose checked flake is in a subdirectory, so that the lock that atoms
ship to projects is checked.

### lib-actions

`action-shell-lib` moves to a new repository, `lib-actions`, with a new history
starting at `v0.1.0`, since it has had no major release. Its consumers move to
it in the order of FY-0035, and `action-shell-lib` is deleted once no repository
references it.

## Constraints

- A `lib-` repository MUST hold only code that flakeyard repositories of its
  area share, and its README MUST state how it is consumed and pinned.
- `lib-atom` MUST have no inputs. Its functions MUST take everything they need
  as arguments.
- An atom MAY declare `lib-atom` as an input and MUST NOT declare any other
  flakeyard repository. It MUST pin `lib-atom` in its own `flake.lock` and MUST
  NOT require projects to declare or follow it.
- Nix code needed by two atoms MUST live in `lib-atom`; helpers of atoms' tests
  MUST live in its `internal/testing/`, which MUST NOT be part of its public
  API.
- An atom's dev flake MUST take the revision of `lib-atom` from the atom's root
  `flake.lock`.
- Renovate MUST propose updates of `lib-atom` in atoms with the commit type
  `fix`.
- `action-flake-lock` MUST accept nested entries of `flakeyard/lib-*` locked
  more than once. `workflow-check` MUST check the root `flake.lock` of a
  repository whose checked flake is in a subdirectory.
- `action-shell-lib` MUST be replaced by `lib-actions` and deleted when no
  repository references it.

## Consequences

- (+) One implementation of the code every atom needs, tested once.
- (+) Projects see no new input and no new `follows` line; every atom runs with
  the `lib-atom` it was tested with.
- (+) A change of `lib-atom` cannot break a project before the atom that takes
  it is released.
- (−) A fix of `lib-atom` reaches projects only through a release of every atom
  that takes it.
- (−) A project's lock holds one entry of `lib-atom` per atom.
- (−) A change of the configuration choice makes `atom-workflows` fail on two
  project files of one tool, where it took the first; allowed in `0.y.z`
  (FY-0024).
- (±) The prefix `lib-` names a role, unlike the other prefixes, which name a
  delivery mechanism.

## Alternatives considered

- **Projects follow `lib-atom`.** One lock entry, but every atom runs with a
  revision it was not tested with, major versions must be taken by all atoms at
  once, and a minimum version cannot be tested.
- **Copies with an exception in FY-0035.** No dependency, but three
  implementations of the configuration choice had already diverged.
- **A shared flake-parts module.** Rejected by FY-0017: one more `follows` and a
  dependency between atoms; plain functions need neither.
- **Renaming `action-shell-lib`.** GitHub does not redirect `uses:` of a renamed
  repository, so a rename breaks every pin at once.
- **Keeping `action-shell-lib` under its name.** No migration, but two naming
  schemes for libraries.

[fy-0037]: 0037-atoms-ship-default-configs-that-projects-replace.md
[github-rename]:
  https://docs.github.com/en/repositories/creating-and-managing-repositories/renaming-a-repository

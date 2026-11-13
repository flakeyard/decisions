---
id: FY-0039
title: Reach repository files in Nix from one root
status: accepted
date: 2026-11-12
supersedes: []
superseded_by: []
related: [FY-0012, FY-0017, FY-0035, FY-0038]
affects:
  [
    "*:flake.nix",
    "*:dev/flake.nix",
    "*:nix/**",
    "*:dev/nix/**",
    "atom-*:nix/default.nix",
  ]
review_by:
---

## Context

The Nix code of flakeyard's repositories reaches the repository's scripts,
tests, workflows and lock files with paths relative to the file that names them:
`../..` in `nix/checks/formatting.nix`, `../../../../scripts/check/licenses` in
`image-devcontainer`, `../../../tools.nix` in the development flake of
`action-flake-lock`. A path like that encodes where both files are. Moving a
module to another directory, or the repository's development flake to `dev/`
([FY-0038](0038-roll-out-the-atoms.md)), changes the meaning of every such path,
and a wrong one fails only when that file is evaluated.

A path relative to a file in the same directory, such as `./default.nix`, moves
with the file. Only paths that climb out of their directory depend on the layout
around them.

## Decision

The root of the repository is defined once per flake, in its entry file, and
passed to the other Nix files:

- In a repository's flake, `flake.nix` defines `root = ./.;`; a development
  flake in `dev/` defines `root = ../.;`. The entry file passes it to the
  flake-parts modules as the module argument `root`.
- Every other Nix file reaches files outside its own directory only as
  `root + "/<path from the root>"`, such as `root + "/scripts/lib/image.sh"`.
- Paths within a file's own directory or below it, such as `./default.nix` or
  `./nix`, stay relative.

Atoms keep the form of their public flake fixed by
[FY-0017](0017-atom-contract.md) and may not set `_module.args`. In an atom,
`nix/default.nix` defines `root = ./..;` once and passes it to the atom's other
files as a function argument, like the atom's inputs.

`atom-nix` checks the rule with a hook `flakeyard-nix-relative-paths`, which
fails on a path that climbs out of its directory anywhere but in these entry
files. Until `atom-nix` is released, reviews check it.

The repositories that already have such paths, `image-devcontainer`,
`workflow-check`, `workflow-scorecard`, `renovate-config` and
`action-flake-lock`, move to the root in one pull request each, without changing
any derivation except those whose source is the edited Nix files themselves,
such as a formatting check.

## Constraints

- A flake's entry file MUST define the root of the repository once and pass it
  to the flake's modules as `root`.
- A Nix file other than a flake's entry file or an atom's `nix/default.nix` MUST
  NOT contain a path that climbs out of its own directory (`../`); it MUST use
  `root + "/<path>"` instead.
- An atom MUST define its root only in `nix/default.nix`, as `./..`, and MUST
  pass it to its other files as a function argument.
- `atom-nix` MUST provide a hook that enforces this rule.
- Moving existing code to the root MUST NOT change its derivations, except those
  whose source is the edited Nix files themselves.

## Consequences

- (+) Moving a file or a directory changes only the paths that name it from the
  root, and a moved development flake changes one line.
- (+) Every path to a repository file reads the same in every Nix file, from the
  root.
- (−) Every module that reaches repository files takes the argument `root`.
- (±) The root is a path inside the flake's source, so a check that uses
  `root + "/<path>"` still depends only on the files it names.

## Alternatives considered

- **`self` of the flake.** `self` is a string of a store path, not a path, so it
  cannot be used with `lib.fileset`, and reading a file through it makes the
  derivation depend on the whole source.
- **Keeping relative paths, checked by tests.** Tests find a broken path only
  after the move; the root avoids it.
- **A root per directory.** Several anchors to update instead of one.

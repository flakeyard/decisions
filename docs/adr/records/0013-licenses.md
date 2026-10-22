---
id: FY-0013
title: License code, generated files and texts
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0002, FY-0005]
affects:
  [
    "*:LICENSE",
    "*:LICENSES/**",
    "*:REUSE.toml",
    "template-*:template/**",
    "image-*:**",
  ]
review_by:
---

## Context

flakeyard is used by other organizations, including in proprietary projects. Its
products reach them in three forms:

- code they depend on: atoms, actions, workflows, the image's build, scripts;
- files that templates write into their repositories, which become part of their
  projects and are edited there;
- texts that explain the product: decisions and documentation.

A permissive license with attribution is the norm for code that others depend
on. Generated files are different: a project would otherwise have to keep a
flakeyard copyright notice in its own `flake.nix` and workflows, or relicense
files it did not write. Texts are better served by a license made for prose.

The dev container image also redistributes binaries of third-party packages
under their own licenses, some of which require access to the corresponding
source.

## Decision

- **Code** is licensed under MIT: atoms, actions, workflows, the image's build
  files, templates outside `template/`, scripts and their documentation.
  `flakeyard/.github` is code too.
- **Files generated into consumer projects**, everything under `template/` of a
  template repository, are licensed under MIT-0. Consumers may use, change and
  relicense them without keeping a notice. Templates do not choose a license for
  the generated project.
- **Texts** of `flakeyard/decisions` are licensed under CC-BY-4.0.

Copyright notices name "The flakeyard contributors". Contributions are accepted
under the license of the repository they are made to; there is no contributor
license agreement.

Every repository follows the [REUSE](https://reuse.software/) specification:
license texts in `LICENSES/`, licensing of files declared in `REUSE.toml` rather
than in headers of every file, and a top-level `LICENSE` for tools that look
only there. `atom-reuse` checks it.

The image is distributed together with how to get the source of everything in
it: every release states the commit and `flake.lock` it was built from, from
which Nix fetches the source of every package, and the image's SBOM lists the
packages and their licenses.

## Constraints

- Code MUST be licensed under MIT, files under `template/` of template
  repositories under MIT-0, and texts of `decisions` under CC-BY-4.0.
- Templates MUST NOT add a license, a copyright notice or a license header of
  flakeyard to generated projects.
- Copyright notices MUST name "The flakeyard contributors".
- Every repository MUST pass `reuse lint`, declare licensing in `REUSE.toml` and
  keep a top-level `LICENSE` with the license of most of its files.
- Code copied from third parties MUST keep its license and be declared in
  `REUSE.toml`.
- Every release of the image MUST name the commit it was built from and ship an
  SBOM with the license of every package.
- Packages whose licenses forbid redistribution MUST NOT be in the image.

## Consequences

- (+) Consumers can use every product in proprietary projects with one notice
  per dependency and none for generated files.
- (+) Licensing of every file is machine-checkable.
- (+) The image's third-party components can be traced to their sources and
  licenses.
- (−) Code that moves from a template's `template/` into an atom changes from
  MIT-0 to MIT, which is allowed only in that direction.
- (±) Without a contributor license agreement, relicensing later needs the
  consent of the contributors.

## Alternatives considered

- **MIT for generated files too.** Every generated project would have to carry a
  flakeyard notice in files it owns.
- **Apache-2.0 for code.** Adds a patent grant, but also notice and change
  marking duties that are heavy for small modules used in many projects.
- **A copyleft license.** Keeps changes open, but rules out many proprietary
  consumers.
- **MIT for texts.** Written for software and awkward for prose; CC-BY-4.0 is
  made for it.
- **License headers in every file.** Noise in small files; `REUSE.toml` declares
  the same in one place.

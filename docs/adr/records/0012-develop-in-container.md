---
id: FY-0012
title: Develop every repository in the dev container
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0004, FY-0005, FY-0009]
affects:
  ["*:.devcontainer/**", "*:.envrc", "*:README.md", ".github:CONTRIBUTING.md"]
review_by:
---

## Context

flakeyard publishes a dev container image so that people without Nix work as if
they had it ([FY-0005](0005-dev-container-image.md)), and atoms provide the
development shell of a project ([FY-0004](0004-atom-contract.md)). flakeyard's
own repositories need formatters, linters and Nix too. Until the image and the
atoms exist, they are developed with tools installed on the host, whose versions
differ from machine to machine and from CI.

Developing flakeyard the way its consumers develop their projects tests the
image, the atoms and the generated files every day, before consumers meet their
problems.

## Decision

Every flakeyard repository is developed in the flakeyard dev container image and
gets its tools from its own `flake.lock`:

- `.devcontainer/devcontainer.json` references
  `ghcr.io/flakeyard/image-devcontainer` by tag and digest. It may add
  `postCreateCommand: direnv allow` and named volumes for caches; everything
  else comes from the image's metadata.
- `.envrc` contains `use flake`, or `use flake ./dev` in repositories whose
  public flake is not their development flake, such as atoms.
- The development shell is built by `atom-workspace` from the atoms the
  repository uses.

A container engine, git and an editor are the only things a contributor
installs. Nix on the host is a supported alternative: `direnv` or `nix develop`
give the same shell without the container.

Local verification runs the same commands as CI
([FY-0009](0009-ci-on-runners.md)): `nix flake check` and the repository's apps.
The README of every repository says how to open it in the container and which
commands to run.

Repositories are moved to this setup as soon as the image and the atoms they
need are released. Until then, and for the first build of the image itself, host
tools or a disposable container with Nix are used. Later versions of the image
are developed in the previous release.

## Constraints

- Every flakeyard repository MUST contain `.devcontainer/devcontainer.json` that
  references the image by tag and digest, and `.envrc` that loads the
  repository's development flake, once the image and its atoms are released.
- `devcontainer.json` MUST NOT repeat settings carried by the image's metadata,
  use Dev Containers features, mount host paths other than the workspace, or
  contain credentials or personal editor settings.
- Tools MUST come from the repository's `flake.lock`. Instructions MUST NOT
  require installing tools on the host other than a container engine, git and an
  editor.
- The same development shell MUST work with Nix on the host.
- Local verification MUST use the same commands as CI.
- The README of every repository MUST describe opening it in the container and
  the verification commands.

## Consequences

- (+) Contributors need no Nix and no tools on the host, and get the versions CI
  uses.
- (+) flakeyard uses its own image, atoms and generated files every day and
  finds their problems first.
- (−) Developing in a container costs disk space and a first download of every
  repository's tools.
- (−) A broken image release blocks development until it is fixed or the
  previous digest is restored.
- (±) Contributors with Nix on the host can skip the container, so that path
  needs to keep working too.

## Alternatives considered

- **Nix on the host as a requirement.** Simple for Nix users, but excludes
  contributors who do not want Nix on their machines.
- **Tools installed on the host.** The current temporary state; versions drift
  between machines and CI.
- **A Dockerfile per repository with its tools.** Every repository maintains an
  image, and tools are pinned twice: in the image and in `flake.lock`.
- **Hosted environments only, such as Codespaces.** Ties development to one
  service and its quotas.

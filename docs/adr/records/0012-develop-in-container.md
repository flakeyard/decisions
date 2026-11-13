---
id: FY-0012
title: Develop every repository in the dev container
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0004, FY-0005, FY-0009, FY-0017]
affects:
  [
    "*:.devcontainer/**",
    "*:.envrc",
    "*:.gitignore",
    "*:README.md",
    ".github:CONTRIBUTING.md",
    "template-*:template/.devcontainer/**",
    "template-*:template/.gitignore*",
  ]
review_by:
---

## Context

flakeyard publishes a dev container image so that people without Nix work as if
they had it ([image-devcontainer ADR-0001][image-devcontainer-adr-0001]), and
atoms provide the development shell of a project
([FY-0004](0004-atom-contract.md)). flakeyard's own repositories need
formatters, linters and Nix too. Until the image and the atoms exist, they are
developed with tools installed on the host, whose versions differ from machine
to machine and from CI.

Developing flakeyard the way its consumers develop their projects tests the
image, the atoms and the generated files every day, before consumers meet their
problems.

## Decision

Every flakeyard repository is developed in the flakeyard dev container image and
gets its tools from its own `flake.lock`:

- `.devcontainer/devcontainer.json` references
  `ghcr.io/flakeyard/image-devcontainer` by tag and digest, and mounts the
  workspace under the home directory as the image's README requires, since the
  image's metadata cannot carry the workspace mount. It may add
  `postCreateCommand` with `direnv allow`, or with
  `direnv allow && direnv exec . true` so that the git hooks are installed when
  the container is created, and named volumes for caches; everything else comes
  from the image's metadata.
- `.envrc` contains `use flake`, or `use flake ./dev` in repositories whose
  public flake is not their development flake, such as atoms.
- The development shell is built by `atom-workspace` from the atoms the
  repository uses.

A container engine, git and an editor are the only things a contributor
installs. Nix on the host is a supported alternative: `direnv` or `nix develop`
give the same shell without the container.

Projects generated from flakeyard templates reference the image the same way:
templates emit `ghcr.io/flakeyard/image-devcontainer:vX@sha256:<digest>` and may
add the same `postCreateCommand` and named volumes for state under the home
directory that should survive recreation of the container, such as caches and
editor servers. The smoke tests of the image run only in the image's own
repository.

The shellHook of the development shell installs the git hooks
([FY-0017](0017-atom-contract.md)): it writes `.pre-commit-config.yaml` and
hooks that point to tools in the `/nix/store` of the container. Commits are
therefore made inside the container, where those paths exist. direnv runs the
shellHook only after `direnv allow`, once per worktree. The `.gitignore` of
every repository and of every generated project contains `.direnv/` and
`.pre-commit-config.yaml`, which the development shell writes.

The shellHook of a branch runs whenever its development shell is entered, and
the container holds forwarded git credentials and the SSH agent. Before an
untrusted branch is checked out for review, the worktree is blocked with
`direnv block`, so that the branch's code does not run on entry.

Local verification runs the same commands as CI
([FY-0009](0009-ci-on-runners.md)): `nix flake check` and the repository's apps.
The README of every repository says how to open it in the container and which
commands to run.

Every repository gets `.devcontainer/devcontainer.json` as soon as the image is
released, so that it is opened in the container from then on; Nix and the
image's tools are there even before the repository has a development shell.
`.envrc` and `postCreateCommand: direnv allow` come with the repository's
development shell, which `atom-workspace` builds; without one, direnv would fail
in the container. Until then, the commands of CI run in the container as they
are. Repositories created later get both files with their first code. Later
versions of the image are developed in the previous release; its first build
used host tools.

## Constraints

- Every flakeyard repository with code MUST contain
  `.devcontainer/devcontainer.json` that references the released image by tag
  and digest.
- Every flakeyard repository with a development shell MUST contain `.envrc` that
  loads it, and only then MAY its `devcontainer.json` run `direnv allow`. Its
  `postCreateCommand` MAY be `direnv allow && direnv exec . true`.
- The `.gitignore` of every flakeyard repository and of every project generated
  from a flakeyard template MUST contain `.direnv/` and
  `.pre-commit-config.yaml`.
- Commits MUST be made inside the container when the git hooks were installed
  there. Instructions MUST state that `direnv allow` is run once per worktree.
- A worktree MUST be blocked with `direnv block` before an untrusted branch is
  checked out in it for review.
- `devcontainer.json` MUST NOT repeat settings carried by the image's metadata,
  use Dev Containers features, mount host paths other than the workspace, or
  contain credentials or personal editor settings.
- Consumers MUST reference the image by digest; templates MUST emit
  `ghcr.io/flakeyard/image-devcontainer:vX@sha256:<digest>`.
- Smoke tests of the image MUST run only in the image repository's CI and MUST
  NOT be executed in consumer projects.
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

[image-devcontainer-adr-0001]:
  https://github.com/flakeyard/image-devcontainer/blob/main/docs/adr/records/0001-dev-container-image.md

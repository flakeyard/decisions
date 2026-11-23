---
id: FY-0042
title: Set up the dev container with one command and keep worktrees inside it
status: accepted
date: 2026-11-23
supersedes: []
superseded_by: []
related: [FY-0012, FY-0017, FY-0019, FY-0024, FY-0035]
affects:
  [
    "*:.devcontainer/devcontainer.json",
    "*:.gitignore",
    "*:README.md",
    "template-*:template/.devcontainer/**",
    "template-*:template/.gitignore*",
  ]
review_by:
---

## Context

[FY-0012](0012-develop-in-container.md) lets the `devcontainer.json` of a
repository with a development shell run `direnv allow`, or
`direnv allow && direnv exec . true` so that the git hooks are installed when
the container is created. The 17 repositories with a development shell have the
same `devcontainer.json`: the image by tag and digest, a bind mount of the
workspace only, and the second form of `postCreateCommand`. Their READMEs say to
run `direnv allow` once in every other worktree. The git hooks come from
[atom-workspace][atom-workspace] ([FY-0017](0017-atom-contract.md)): it sets
`core.hooksPath` to the `hooks` directory of the common git directory and writes
hooks that call tools by their `/nix/store` paths.

Three failures were observed:

- **A worktree outside the workspace.** The `.git` file of a worktree names its
  git directory by an absolute path into the main checkout. When the worktree is
  opened as the workspace of its own container, that path is outside the mount:
  git does not work in the container, and neither does anything that needs it.
- **Hooks shared by containers with different stores.** Every worktree of a
  repository shares `core.hooksPath`. Each container has its own Nix store, and
  entering the development shell in one container rewrites the shared hooks with
  paths of its own store. A commit in another container then calls tools that do
  not exist in its store.
- **Absolute paths of worktrees.** git records a worktree and its main checkout
  by absolute paths. The workspace is mounted at `/home/dev/NAME`, a different
  path from the one on the host, so a worktree created in the container is
  broken for git on the host, and one created on the host is broken in the
  container. git 2.48 added `worktree.useRelativePaths`, which records relative
  paths instead.
- **A setup that succeeds on a broken shell.** `direnv exec . true` exits with 0
  when the flake fails to evaluate or build: direnv reports the error and still
  runs the command. The container is created without a development shell and
  without hooks, and nothing says so until a commit runs no checks.

The same command line is repeated in every `devcontainer.json`, and any fix of
it would be repeated 17 times; by
[FY-0035](0035-extract-logic-that-has-two-users.md) it belongs in one place. The
image already holds the tools the setup needs, direnv and git.

The first creation of a container was measured with the image already pulled: 36
seconds in total, of which 34 seconds to get the development shell from
`cache.nixos.org` (328 MiB downloaded, nothing built), a store of 1.6 GB, and
0.13 seconds to load the shell again. The binary cache of the organization
([FY-0019](0019-binary-cache.md)) is not configured in the container and is not
needed for that. Several minutes seen earlier came from running the
`devcontainer` command line on the host through `nix run nixpkgs#devcontainer`,
which fetched nixpkgs each time, not from the container. The store stays in the
container, as [image-devcontainer ADR-0001][image-devcontainer-adr-0001] decided
when it rejected a volume for `/nix`.

## Decision

### Setup command

The image ships a command `devcontainer-setup` that sets up a project in its
container. Its contract and tests are recorded in [image-devcontainer
ADR-0002][image-devcontainer-adrs]. In summary, it takes no arguments, runs in
the workspace, prints one summary line and exits non-zero when:

- the workspace is a worktree whose `.git` points outside the mounted workspace,
  with a message to create worktrees inside the workspace;
- the development shell does not evaluate or build, after `direnv allow`;
- there is no pre-commit hook at `core.hooksPath`, or a tool that the hook calls
  is missing from the store.

Before those checks, it sets `git config --local worktree.useRelativePaths true`
in the repository, so that worktrees created afterwards work both in the
container and from git on the host.

The `postCreateCommand` of every repository with a development shell is exactly
`devcontainer-setup`. A failed setup fails the creation of the container, so a
broken shell is seen at once.

### Worktrees

Worktrees of a repository are created inside its workspace, under `.worktrees/`
(for example `git worktree add .worktrees/NAME`), and are used from the one
container of the workspace, whose store the shared hooks point to. `.worktrees/`
is listed in `.gitignore`. A worktree outside the workspace, or a container of
its own for a worktree, is not supported, and `devcontainer-setup` refuses a
workspace whose `.git` points outside the mount.

### README

The README of every repository says, in its development section, that:

- worktrees are created with `git worktree add .worktrees/NAME` and used from
  the same container;
- git on the host has to be 2.48 or newer to use those worktrees;
- `direnv allow` is run once in every worktree;
- `direnv block` is run in a worktree before a branch that is not trusted is
  checked out in it.

### Store and cache

The Nix store stays in the container, and the binary cache configuration does
not change.

### Templates

Projects generated from flakeyard templates follow the same rules: a template
that emits a `postCreateCommand` emits `devcontainer-setup`, and its
`.gitignore` lists `.worktrees/`.

### Rollout

1. `image-devcontainer` releases a version that ships `devcontainer-setup`.
2. Renovate updates the digest of the image in the 17 repositories.
3. One pull request per repository sets `postCreateCommand` to
   `devcontainer-setup`, adds `.worktrees/` to `.gitignore` and the worktree
   paragraph to the README, in one repository first and then in the rest.

Until a repository gets its pull request, FY-0012 applies to it unchanged.

### Amendments

This decision amends [FY-0012](0012-develop-in-container.md); the rest of it
stays in force. The only allowed `postCreateCommand` is `devcontainer-setup`,
instead of `direnv allow` or `direnv allow && direnv exec . true`; worktrees are
created inside the workspace and used from one container; `.gitignore` also
lists `.worktrees/`; worktrees record relative paths; and the README states how
worktrees are created.

## Constraints

- The `postCreateCommand` of a repository with a development shell MUST be
  exactly `devcontainer-setup`, the command of the image defined by
  image-devcontainer ADR-0002. Other forms, such as
  `direnv allow && direnv exec . true`, MUST NOT be used.
- A template that emits a `postCreateCommand` MUST emit `devcontainer-setup`.
- `devcontainer-setup` MUST exit non-zero when the workspace is a worktree whose
  `.git` points outside the mount, when the development shell does not evaluate
  or build, or when the pre-commit hook or a tool it calls is missing.
- `devcontainer-setup` MUST set `worktree.useRelativePaths` to `true` in the
  local git configuration of the repository.
- Worktrees MUST be created inside the workspace, under `.worktrees/`, and MUST
  be used from the container of that workspace. A worktree outside the workspace
  MUST NOT be documented or relied on as supported.
- The `.gitignore` of every repository and of every generated project with a
  `postCreateCommand` MUST contain `.worktrees/`.
- The README of every repository MUST say that worktrees are created with
  `git worktree add .worktrees/NAME` and used from the same container, that git
  on the host MUST be 2.48 or newer to use them, that `direnv allow` is run once
  per worktree, and that `direnv block` is run before an untrusted branch is
  checked out.
- The Nix store MUST stay in the container; this decision MUST NOT be read as a
  reason to add a volume for `/nix`.
- A repository MUST NOT set `postCreateCommand` to `devcontainer-setup` before
  its `devcontainer.json` references an image that ships the command.

## Consequences

- (+) A container whose shell or hooks are broken fails when it is created, with
  one line saying why, instead of later at a commit.
- (+) The setup is fixed in one place, the image, and reaches every repository
  with a digest update.
- (+) All worktrees of a repository share one store and one set of hooks that
  work.
- (−) `devcontainer-setup` is public API of the image: changing its behaviour is
  a release of the image under [FY-0024](0024-versioning-and-support.md).
- (+) Worktrees record relative paths, so a worktree created in the container
  works from git on the host and the reverse.
- (−) git on the host older than 2.48 cannot use those worktrees.
- (±) Worktrees created before the setup set `worktree.useRelativePaths` keep
  their absolute paths until `git worktree repair --relative-paths` rewrites
  them.
- (−) One container serves every worktree of a repository, so their builds share
  its disk and its store.
- (±) The development shell is downloaded once per container, about 36 seconds
  for these repositories, as before.

## Alternatives considered

- **Keeping `direnv allow && direnv exec . true`.** It succeeds on a broken
  shell and is repeated in every `devcontainer.json`.
- **A setup script in every repository or in an atom.** Repeats the logic or
  needs the development shell that the setup has to check first; the image is
  the one place that runs before the shell exists.
- **Worktrees outside the workspace with a container each.** Needs a mount of
  the main checkout's git directory, which FY-0012 forbids, and still has
  containers overwrite each other's hooks.
- **Hooks per worktree.** The hook path is set by atom-workspace for the common
  git directory; changing it does not remove the second store and its download.
- **A volume for `/nix` or a binary cache of the organization in the
  container.** The measurement shows no need for either, and image ADR-0001
  rejected the volume.

[atom-workspace]: https://github.com/flakeyard/atom-workspace
[image-devcontainer-adr-0001]:
  https://github.com/flakeyard/image-devcontainer/blob/main/docs/adr/records/0001-dev-container-image.md
[image-devcontainer-adrs]:
  https://github.com/flakeyard/image-devcontainer/tree/main/docs/adr/records

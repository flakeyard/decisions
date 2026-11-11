---
id: FY-0035
title: Extract logic that has two users
status: accepted
date: 2026-11-07
supersedes: []
superseded_by: []
related:
  [FY-0001, FY-0003, FY-0017, FY-0022, FY-0024, FY-0029, FY-0030, FY-0033]
affects: ["*:**"]
review_by:
---

## Context

flakeyard repositories repeat their logic. `report.sh`, `api.sh`, `contents.sh`
and the forms of repository names were copied from `workflow-check` into
`workflow-release`, `action-flake-lock`, `action-setup-nix`, `.github`,
`image-devcontainer` and `renovate-config`, together with test harnesses and
stubs of `gh` and `nix`. The Nix modules `nix/checks/formatting.nix`,
`nix/formatter.nix`, `nix/checks/shell.nix`, `nix/apps/make.nix`,
`nix/sources.nix` and `nix/checks/unit.nix` exist in several repositories. The
copies have already diverged, and a fix reaches only the copy it was made in.

[FY-0030](0030-actions-workflows-and-release-bot.md) keeps the logic of an
action in its own repository and the logic of a workflow in actions. It offers
no way to share shell code between repositories without Nix, and some of that
code has to run without Nix: `action-setup-nix` runs before Nix exists, and API
steps use the `bash`, `gh` and `jq` of the runner.

A probe of a composite action that exposes its own directory to its caller
established:

- A composite action without steps whose output is `${{ github.action_path }}`
  runs, but the output is empty on Linux, macOS and Windows runners and in a
  container job.
- One step with `shell: /usr/bin/bash --noprofile --norc -p -e {0}` that writes
  `github.action_path`, passed through an environment variable, to
  `GITHUB_OUTPUT` gives the directory of the action. It works when used
  directly, through a nested composite action, in a container job and with a
  hostile environment: a fake `bash` first on `PATH` through `GITHUB_PATH` and
  `BASH_ENV` set through `GITHUB_ENV` did not hijack it.
- `/usr/bin/bash` does not exist on macOS and Windows runners.

## Decision

### The rule

Logic that has at least two users is extracted to one reusable place. A user is
a place that runs the logic: two places in one repository, or places in two
repositories. Logic is scripts, `jq` programs, Nix modules, test helpers and
stubs.

Caller boilerplate is not logic: thin callers such as `test.yaml`,
`scorecard.yaml` and `release.yaml`, and `renovate.json`, `REUSE.toml` and
`LICENSE`. The templates generate it (step 9 of [FY-0022](0022-bootstrap.md)).

A test may restate a value as an independent oracle where reusing the code under
test would make the test check that code against itself. The test says so in a
comment.

### Library actions

Shell code shared at run time without Nix is delivered by a library action. It
is an `action-` repository ([FY-0003](0003-repository-naming.md)) whose one task
is to expose its directory, and its product is the files in that directory. The
first one is `action-shell-lib`, designed in [action-shell-lib
ADR-0001][action-shell-lib-adr-0001]. It runs one step on Linux and another on
every other runner:

```yaml
outputs:
  path:
    description: Directory of the library.
    value: ${{ steps.path.outputs.path }}
runs:
  using: composite
  steps:
    - id: path
      if: runner.os == 'Linux'
      shell: /usr/bin/bash --noprofile --norc -p -e {0}
      env:
        ACTION_DIR: ${{ github.action_path }}
      run: printf 'path=%s\n' "$ACTION_DIR" >>"$GITHUB_OUTPUT"
    - if: runner.os != 'Linux'
      shell: bash
      run: |
        echo "::error::Unsupported runner $RUNNER_OS: use a GitHub-hosted" \
          "Linux runner, such as ubuntu-24.04 or ubuntu-24.04-arm."
        exit 1
```

A consumer pins it like any action, gives the step an `id`, passes the output to
its own steps as the environment variable `SHELL_LIB` and sources files from
that directory:

```yaml
- id: shell-lib
  uses: flakeyard/action-shell-lib@<40-character sha> # v1.2.0
- shell: bash
  env:
    SHELL_LIB: ${{ steps.shell-lib.outputs.path }}
  run: "$GITHUB_ACTION_PATH/scripts/check.sh"
```

A library brings no tools. Its functions use what the calling step provides: the
tools of the consumer's flake, or the runner's `gh` and `jq` in API steps.

The path step needs `/usr/bin/bash`, so it runs only on Linux. On every other
runner the second step of the library, with the `bash` of the runner, writes the
error of the rejection and fails the job. The text of the rejection lives in the
library, and consumers do not reject runners other than Linux themselves.

### Tests of consumers

Every `uses:` of the library in one repository names the same commit. The tests
of a consumer may need the library without a runner that has resolved `uses:`.
They fetch its source at that commit and take the commit from the file that pins
it, the consumer's `action.yml` or a workflow, so there is one pin: with
`builtins.fetchGit { url; rev; shallow = true; }` in a development flake, or
with `git fetch` of that commit in tests that run on a runner without Nix. A
repository that uses the library only in its tests pins it in the job of its CI
workflow that runs them, and takes the commit from there. When those tests run
inside the flake checks of `workflow-check`, whose caller job has no steps of
its own, the repository pins the library in a job of its own in the same
workflow, which `check` needs, and the tests take the commit from that workflow.
This is not a second delivery mechanism.

Shared test helpers live in `internal/testing/` of the library: harnesses, stubs
of `gh` and `nix` and their fixtures. Consumers' tests may use them at the
pinned commit. They are not public API, as the `internal/` presets of
`renovate-config` are not ([FY-0024](0024-versioning-and-support.md)).

The one exception to the rule is the loader of the library for local test runs.
A loader cannot come from the library that it loads, so each consumer whose
tests run locally keeps its own minimal one: unless `SHELL_LIB` is already set,
it reads the commit from the file that pins it, fetches that commit once into a
cache, such as `~/.cache/flakeyard/action-shell-lib/<commit>`, and sets
`SHELL_LIB`. Everything else that the tests share comes from the library.

### Versions

The public API of a library action is the names of its functions, their
arguments and return codes, and the variables they read and write, as its README
documents them. `internal/testing/` is not part of it. Removing or changing any
of it is a major release. A major release of a library is not by itself a major
release of its consumers: a consumer is released by the effect of the update on
its own public API.

A release of a consumer pins the library by commit, so it still fixes all the
logic it runs ([FY-0030](0030-actions-workflows-and-release-bot.md)).

### Releases of a library change

A change of `action-shell-lib` is released in this order: `action-shell-lib`,
then `action-setup-nix` and `action-flake-lock`, then `workflow-release` and
`workflow-check` once each. `workflow-check` also pins `action-setup-nix` and
`action-flake-lock`, so it takes all three updates in one release.

Renovate proposes updates of `flakeyard/action-shell-lib` with the commit type
`fix` in consumers that ship the library to their users: actions, where it runs
in `action.yml`, and reusable workflows, where it runs in the workflow their
callers use. In repositories that use it only in their own CI or tests, such as
`workflow-scorecard` and `image-devcontainer`, an update is `ci` like any other
update of CI ([FY-0007](0007-branching-and-releases.md)). The rule for actions
and workflows lives in their presets of `renovate-config`
([FY-0029](0029-renovate-mend.md)); a workflow that uses the library only in its
tests overrides it in its `renovate.json`.

### Trust

The library runs in sensitive jobs: in the job of `workflow-release` that holds
the key of `flakeyard-bot`, and in `action-setup-nix`, which runs `sudo`. Its
releases pass the release gate of
[FY-0030](0030-actions-workflows-and-release-bot.md) and the audit of
[FY-0033](0033-custody-of-the-release-bot-key-and-audit-of-releases.md) like
every flakeyard release. Consumers that allow only listed actions add it to
their list.

### Bootstrap

`action-shell-lib` is released before `workflow-release` and `action-setup-nix`
use it. Its own first release runs on a `workflow-release` that does not use it
yet.

The Nix modules that are duplicated today stay until the atoms that own them
([FY-0017](0017-atom-contract.md)) are released: `atom-nix`, `atom-shell` and
`atom-workspace`. Each repository with such a copy adopts the atom in its next
pull request after the release. The atoms stay step 8 of the order of FY-0022,
which otherwise holds.

### Amendments

This decision amends other records; the rest of each stays in force.

- [FY-0001](0001-record-decisions.md), edited in place: a new record may amend
  an accepted one by stating the amendment in its own Decision and Constraints,
  as this one does for FY-0022.
- [FY-0003](0003-repository-naming.md), edited in place: a library action is an
  `action-` repository, and the tests of its consumers may fetch its source at
  the pinned commit without making that a second delivery mechanism.
- [FY-0024](0024-versioning-and-support.md), edited in place: the public API of
  library actions, and that a library major is not by itself a consumer major.
  `VERSIONING.md` of `.github` lists it.
- [FY-0030](0030-actions-workflows-and-release-bot.md), edited in place: the
  logic of an action may live in a library action it pins, and the tests of
  consumers may use `internal/testing/` of a library.
- [FY-0022](0022-bootstrap.md), stated here: the place of `action-shell-lib` in
  the bootstrap order and the exception for duplicated Nix modules, both under
  "Bootstrap" above. An edit in place would renumber the steps that other
  records cite by number, and the exception has its own end condition.

## Constraints

### Shared logic

- Logic with at least two users, in one repository or in several, MUST exist in
  one reusable place. Copies MUST NOT be added.
- Caller boilerplate that templates generate is exempt.
- A test MAY restate a value instead of reusing the code under test only where
  reuse would make the code test itself, and MUST say so in a comment.

### Delivery

- Shell code shared at run time without Nix MUST be delivered by a library
  action, an `action-` repository whose only task is to expose its directory.
- A library action MUST expose its directory through one step with
  `if: runner.os == 'Linux'` and
  `shell: /usr/bin/bash --noprofile --norc -p -e {0}` that writes
  `github.action_path`, passed through `env`, to `GITHUB_OUTPUT` as `path`.
- A library action MUST reject runners other than Linux itself, in a second step
  with `if: runner.os != 'Linux'` and `shell: bash` that writes an error and
  exits 1.
- A library action MUST NOT bring tools of its own.
- Consumers MUST pin a library action like any other action, every `uses:` of it
  in one repository at the same commit, and MUST pass its output to their steps
  as `SHELL_LIB`. Consumers MUST NOT reject runners other than Linux themselves.
- Consumers' tests MAY fetch the library's source only at the pinned commit, and
  MUST take that commit from the `action.yml` or workflow that pins it. A
  repository that uses the library only in tests MUST pin it in the job of its
  CI workflow that runs them; when they run inside the flake checks of
  `workflow-check`, it MUST pin it in a job of its own in the same workflow,
  which `check` needs, and the tests MUST take the commit from that workflow.
- Shell test helpers shared between repositories MUST live in
  `internal/testing/` of the library. The only exception is the loader of the
  library for local test runs: a consumer MAY keep one, which MUST do nothing
  but, unless `SHELL_LIB` is set, read the pinned commit, fetch that commit once
  into a cache and set `SHELL_LIB`.

### Versions and releases

- The public API of a library action MUST be documented in its README: function
  names, arguments, return codes and the variables they read and write.
  `internal/testing/` MUST NOT be part of it.
- A consumer MUST NOT be released as a major version only because it takes a
  major release of a library.
- A change of `action-shell-lib` MUST be released in the order
  `action-shell-lib`, then `action-setup-nix` and `action-flake-lock`, then
  `workflow-release` and `workflow-check`.
- Updates of `flakeyard/action-shell-lib` MUST be proposed with the commit type
  `fix` in actions and reusable workflows that run the library for their users,
  and with `ci` in repositories that use it only in their own CI or tests.

### Order and exceptions

- `action-shell-lib` MUST be released before `workflow-release` and
  `action-setup-nix` use it.
- The duplicated Nix modules MAY stay only until the atom that owns them is
  released; each repository with a copy MUST adopt the atom in its next pull
  request after that release.

## Consequences

- (+) A fix of shared logic is made once and reaches every user through
  Renovate.
- (+) A release of an action or a workflow still fixes everything it runs.
- (+) Tests of consumers run the same library code as their release.
- (−) A change of the library takes up to three releases in a fixed order before
  every consumer has it.
- (−) The library runs in the jobs that hold the release key and `sudo`, so a
  compromised release of it reaches them; the release gate and the audit are
  what stop it.
- (−) Consumers need Linux runners and `/usr/bin/bash` for any step that uses
  the library.
- (±) Until the atoms are released, the duplicated Nix modules stay where they
  are.

## Alternatives considered

- **Vendored copies generated from one source.** Still duplicated in every
  repository, and Renovate of Mend cannot run a generator to vendor a new
  version.
- **A library action without steps.** Simplest, but the probe showed that its
  output is empty.
- **A library action that runs a script given as an input (`with: script`).**
  Every variable and output of the script would have to pass through the inputs
  and outputs of the library.
- **An exception for `action-setup-nix`.** Breaks the rule for the action that
  needs it most, and the hardened step works before Nix exists.

[action-shell-lib-adr-0001]:
  https://github.com/flakeyard/action-shell-lib/blob/main/docs/adr/records/0001-shell-library.md

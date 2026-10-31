---
id: FY-0020
title: Deliver actions and reusable workflows
status: accepted
date: 2026-10-28
supersedes: [FY-0008]
superseded_by: []
related: [FY-0002, FY-0003, FY-0006, FY-0007, FY-0010, FY-0016]
affects: ["action-*:**", "workflow-*:**", "template-*:template/.github/**"]
review_by:
---

## Context

flakeyard ships its CI logic to consumers as GitHub Actions: composite actions
in `action-*` repositories and reusable workflows in `workflow-*` repositories,
one product per repository ([FY-0003](0003-repository-naming.md)). Consumers
reference them with `uses:` from their own workflows, and the code runs with the
permissions and secrets of the consumer's repository.

That makes every action and workflow part of the consumer's supply chain. A tag
can be moved after review; a step can interpolate untrusted text into a shell; a
workflow can ask for more permissions than it needs. A tool fetched at run time
from a moving source, such as `nixpkgs#<tool>`, changes behaviour without a
release.

Some steps need a token whose events start other workflows. Pull requests and
tags created with the built-in `GITHUB_TOKEN` do not trigger workflows, except
for `workflow_dispatch` and `repository_dispatch`. flakeyard has its own GitHub
App for releases ([FY-0007](0007-branching-and-releases.md)), but consumers are
other organizations and may not want to create one.

A probe of release automation on GitHub-hosted runners established:

- A check run started through `workflow_dispatch` does not count for the
  required checks of a pull request, even on its head commit. A commit status
  that the same run posts with `GITHUB_TOKEN` does.
- An environment secret reaches a job of a reusable workflow only when the
  caller passes a secret under the same name; an empty `environment` is accepted
  and means none.
- `actions/create-github-app-token` identifies an App by its client ID; the
  numeric App id is deprecated there.
- A step can fail after it has changed the repository, for example after
  creating a release, when a later API call fails transiently. A rerun then
  finds nothing left to do unless the workflow derives its state from the
  repository.
- GitHub offers `uses: $/<path>` for references within the same repository. It
  is resolved from the repository, not from the runner's file system, and counts
  as pinned.
- Most steps of release automation only call the GitHub API. `bash`, `gh` and
  `jq` are installed on GitHub-hosted runners and need no Nix.

This record replaces [FY-0008](0008-actions-and-workflows.md) with these
findings; everything else is unchanged.

## Decision

### Actions

An action is a composite action with one task. Its logic lives in scripts in its
repository, and the tools it runs come from the action's own `flake.nix` and
`flake.lock`, so a release of the action fixes the versions of its tools. Only
`action-setup-nix` runs before Nix exists and pins its installer by hash.

Steps that only call the GitHub API, such as moving a tag, reading a release or
posting a commit status, are an exception: they use `bash`, `gh` and `jq` as
installed on GitHub-hosted runners and may live in an action or in a workflow.
They run with `set -euo pipefail`, and those tools are part of the documented
runner requirements.

Actions are generic: project specifics, such as which flake outputs to build or
which systems to check, are inputs. Untrusted values, such as pull request
titles and branch names, reach scripts only through environment variables, never
through expression interpolation inside `run:`.

### Reusable workflows

A reusable workflow is a thin composition of actions: it chooses runners,
permissions, the order of steps and which inputs reach which action. It contains
no logic of its own beyond that and the API steps described above. The product
workflow of `workflow-<name>` is `.github/workflows/<name>.yaml`; other files in
that directory are the repository's own CI.

Every workflow declares no permissions at the top level and the minimum each job
needs. Secrets are declared and passed explicitly; workflows do not rely on
`secrets: inherit`. A job that needs an environment secret takes the environment
name as an input, and callers pass the secret under the name the workflow
declares.

Workflows that change the repository, such as releasing or moving tags, are
idempotent: a rerun completes what a failed run left undone, and repeating a
completed run changes nothing.

### References

Consumers and flakeyard itself reference actions and workflows by full commit
SHA with the version in a comment:

```yaml
uses: flakeyard/action-setup-nix@<40-character sha> # v1.2.0
```

Renovate updates the SHA and the comment together. Within flakeyard, workflows
reference actions of other repositories the same way. References within one
repository use `$/<path>`, not `./<path>`.

### Bot of the consumer

A bot is optional for consumers. Workflows take a GitHub App as optional inputs
(its client ID and its private key) and fall back to `GITHUB_TOKEN` without
them. A client ID without a key is an error, not a fallback. Personal access
tokens are not supported. Without an App:

- publishing after a release (images, binary cache, release assets) still works,
  because it runs in the workflow run of the release commit, not in a workflow
  triggered by the new tag;
- checks do not start on pull requests opened by the release workflow, so the
  release workflow starts the check workflow on that branch through
  `workflow_dispatch` after every update of the pull request. Every check
  workflow accepts that trigger and, when started by it, reports its result as a
  commit status on the commit it checked: `pending` when it starts, then
  `success` or `failure`. Required checks name the status with GitHub Actions as
  its source;
- moving the tag `vX` is a separate input, off by default, because projects that
  nobody pins do not need it.

Consumers without an App must allow GitHub Actions to create pull requests in
their repository settings. Inside flakeyard the App is required.

### Tests

Actions and workflows are tested in the CI of their own repositories. Their
tests and fixtures are not part of what consumers reference.

## Constraints

### Action content

- An action MUST be a composite action with one task.
- An action MUST take its tools from its own `flake.lock`. It MUST NOT run tools
  from `nixpkgs#`, from the consumer's flake, or downloaded without a pinned
  hash. `action-setup-nix` MUST pin its installer by hash.
- Steps that only call the GitHub API MAY use `bash`, `gh` and `jq` of
  GitHub-hosted runners, MUST run with `set -euo pipefail` and MUST be listed in
  the runner requirements of their action or workflow.
- An action MUST NOT assume project-specific names; they MUST be inputs.
- Values from events, inputs and other untrusted sources MUST reach scripts only
  through environment variables. `${{ }}` expressions MUST NOT appear inside
  `run:`.

### Workflow content

- A reusable workflow MUST contain only the composition of actions and API
  steps: runners, permissions, steps, conditions and inputs. Other logic MUST
  live in actions.
- The product workflow of `workflow-<name>` MUST be
  `.github/workflows/<name>.yaml`.
- Workflows MUST set `permissions: {}` at the top level and the minimum per job.
  Required permissions MUST be documented as public API.
- Workflows MUST declare every secret they use and MUST NOT require
  `secrets: inherit`. A job that uses an environment secret MUST take the
  environment name as an input.
- Workflows that change the repository MUST be idempotent and MUST derive what
  is left to do from the state of the repository, not only from the outputs of
  earlier steps.
- Every check workflow MUST accept `workflow_dispatch` and, when started by it,
  MUST report `pending`, then `success` or `failure`, as a commit status on the
  checked commit.

### Pinning

- References to actions and workflows of other repositories MUST use a full
  commit SHA followed by a version comment. Tags and branches MUST NOT be used
  as references.
- References within one repository MUST use `$/<path>`. Workspace-relative
  references (`./`) MUST NOT be used.

### Tokens

- Workflows MUST work with `GITHUB_TOKEN` alone. A GitHub App MUST be an
  optional input, identified by its client ID, and its absence MUST only disable
  documented features. A client ID without a private key MUST fail the run.
- Workflows MUST NOT accept personal access tokens.
- Workflows MUST NOT require secrets or Apps of flakeyard.
- Publishing that follows a release MUST run in a workflow run of the release
  commit (the run that created the release, or a rerun of it), never in a
  workflow triggered by the new tag: attestations name the commit of the run.
- Moving the tag `vX` MUST be an input that is off by default.

### Test isolation

- Tests and fixtures of actions and workflows MUST run only in their own
  repositories and MUST NOT be referenced by consumers.

## Consequences

- (+) A consumer's CI changes only when the consumer merges an update of a
  pinned SHA, and the tools inside an action change only with its release.
- (+) Each action and workflow has its own version, changelog and permissions.
- (+) Consumers use flakeyard workflows without creating a GitHub App or sharing
  any secret with flakeyard.
- (+) Untrusted input cannot inject shell code through expressions.
- (−) An action brings its own nixpkgs, so a project downloads tools that may
  already be in its own lock.
- (−) Without an App, release pull requests rely on `workflow_dispatch` and
  commit statuses for their checks, consumers must allow Actions to create pull
  requests, and consumers who need `vX` must enable it.
- (−) Anyone who can run a workflow with `statuses: write` can post a passing
  status; this is the same exposure as any check run for people with write
  access.
- (±) API steps depend on the `gh` and `jq` of the runner image instead of a
  pinned version.
- (−) Many small repositories and SHA updates for every release.
- (±) Workflows stay small and change rarely; most changes are releases of
  actions.

## Alternatives considered

- **JavaScript or container actions.** JavaScript actions need a build and a
  committed bundle; container actions run only on Linux and add an image per
  action. Composite actions with Nix-provided tools need neither.
- **Tools from the consumer's nixpkgs.** Fewer downloads, but the behaviour of
  an action would depend on the project's lock instead of the action's release.
- **References by tag.** Readable, but a tag can be moved to unreviewed code.
- **One repository with all workflows.** Rejected in
  [FY-0003](0003-repository-naming.md): one version for unrelated workflows.
- **A required GitHub App for consumers.** Simpler workflows, but every consumer
  organization would have to create and maintain an App.
- **Personal access tokens as a fallback.** Long-lived and tied to a person;
  `GITHUB_TOKEN` with documented limits is safer.
- **API steps in actions with tools from Nix.** Pinned tools, but every workflow
  would need Nix installed before it could call the API.
- **Check runs for dispatched checks.** They do not count for required checks of
  pull requests.

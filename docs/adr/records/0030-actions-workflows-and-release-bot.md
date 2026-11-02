---
id: FY-0030
title: Deliver actions and workflows with a gated release bot
status: accepted
date: 2026-11-02
supersedes: [FY-0020, FY-0021]
superseded_by: []
related:
  [
    FY-0002,
    FY-0003,
    FY-0006,
    FY-0007,
    FY-0009,
    FY-0011,
    FY-0016,
    FY-0022,
    FY-0024,
    FY-0028,
    FY-0029,
  ]
affects:
  [
    ".github:org/**",
    "*:.github/workflows/**",
    "action-*:**",
    "workflow-*:**",
    "template-*:template/.github/**",
  ]
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

Releases of flakeyard create tags `vX.Y.Z`, move tags `vX` and open release pull
requests ([FY-0007](0007-branching-and-releases.md)). flakeyard's own
repositories are also kept up to date from a template by automated pull
requests, some of which change workflow files. Pull requests and tags created
with the built-in `GITHUB_TOKEN` do not start other workflows, except for
`workflow_dispatch` and `repository_dispatch`, and `GITHUB_TOKEN` cannot change
files under `.github/workflows/`. Rulesets protect tags `v*` from everyone
except one bypass actor, which therefore holds the most powerful credential of
the organization: with it, a release can be published from any commit. A
personal access token would tie that credential to a person, live until it is
revoked and carry every permission of the person's account. Consumers are other
organizations and may not want to create a GitHub App.

A probe of release automation on GitHub-hosted runners established:

- A check run started through `workflow_dispatch` does not count for the
  required checks of a pull request, even on its head commit. A commit status
  that the same run posts with `GITHUB_TOKEN` does.
- An environment secret reaches a job of a reusable workflow only when the
  caller passes a secret under the same name; an empty `environment` is accepted
  and means none.
- `actions/create-github-app-token` identifies an App by its client ID, while
  rulesets name their bypass actors by the numeric App id.
- The deployment branch policy of an environment matches branch names, so
  whoever can create a branch named `release/v9` can deploy to it.
- A step can fail after it has changed the repository, for example after
  creating a release, when a later API call fails transiently. A rerun then
  finds nothing left to do unless the workflow derives its state from the
  repository.
- GitHub offers `uses: $/<path>` for references within the same repository. It
  is resolved from the repository, not from the runner's file system, and counts
  as pinned.
- Most steps of release automation only call the GitHub API. `bash`, `gh` and
  `jq` are installed on GitHub-hosted runners and need no Nix.

[FY-0028](0028-supply-chain.md) then required the approval of an organization
owner for the environment `release`, which holds the key of the release bot,
because the Renovate app hosted by Mend ([FY-0029](0029-renovate-mend.md)) can
write to every repository. Applying that showed:

- Every job that uses an environment with a required reviewer waits for it, also
  with `deployment: false`. The release workflow used the key in two jobs on
  every push, one for releases and one for the release pull request, so every
  merge would wait for two approvals, also when nothing is released.
- GitHub allows Actions to create pull requests only together with approving
  them: the setting "Allow GitHub Actions to create and approve pull requests"
  (`can_approve_pull_request_reviews`) is one flag, at the organization and at
  each repository. A repository can enable it only while the organization allows
  it, and a new repository takes the organization's value when it is created;
  later changes at the organization do not reach it. With the flag on, any job
  with `pull-requests: write` can approve a pull request as
  `github-actions[bot]`, and a workflow can ask for that permission in its own
  file whatever the default is.
- A job can wait for approval for 30 days, and a workflow run can be rerun for
  30 days after it was created.
- After a rejection the merged release pull request stays labelled
  `autorelease: pending`, and release-please opens no new release pull request
  while such a pull request exists.
- Marking a release as latest is a change of release metadata, which
  `GITHUB_TOKEN` with `contents: write` can make; it needs neither the bypass of
  the tag ruleset nor the key.

A live probe of the gated release workflow then showed:

- Each release asks for one approval. Rerunning the failed jobs of a rejected
  run asks again, and the approvals of a run list only its latest attempt.
- Time spent waiting for approval does not count toward `timeout-minutes`, and a
  waiting job holds its concurrency group.
- After a release, `GITHUB_TOKEN` opens the next release pull request.
- A merge commit whose message contains `[skip ci]` gets no push run.
- release-please finds the latest release only by the version in the manifest.
  While that version has no tag, it lists the whole history again and bumps the
  version by all of it.

This record replaces [FY-0020](0020-actions-and-workflows.md) and
[FY-0021](0021-release-bot.md) with these findings. What changes is who opens
release pull requests, which jobs use the environment `release` and when, the
key check and the flag; everything else is unchanged.

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
them. A client ID without a key is an error, not a fallback: every job that
needs the key fails before it uses it, and the first job of the run fails when
the key is visible to it. Personal access tokens are not supported.

Release pull requests are opened with the App or with `GITHUB_TOKEN`, chosen by
an input of the release workflow. With an App the App is the default, so that
release pull requests get ordinary check runs; a consumer whose environment has
a required reviewer chooses `GITHUB_TOKEN`, so that only releases wait. When
release pull requests are opened with `GITHUB_TOKEN`:

- checks do not start on them, so the release workflow starts the check workflow
  on that branch through `workflow_dispatch` after every update of the pull
  request. Every check workflow accepts that trigger and, when started by it,
  reports its result as a commit status on the commit it checked: `pending` when
  it starts, then `success` or `failure`. Required checks name the status with
  GitHub Actions as its source;
- the repository must allow GitHub Actions to create pull requests.

Without an App, publishing after a release (images, binary cache, release
assets) still works, because it runs in the workflow run of the release commit,
not in a workflow triggered by the new tag. Moving the tag `vX` is a separate
input, off by default, because projects that nobody pins do not need it.

### Tests

Actions and workflows are tested in the CI of their own repositories. Their
tests and fixtures are not part of what consumers reference.

### flakeyard-bot

flakeyard uses one GitHub App, `flakeyard-bot`, owned by the organization,
installed only there, on all repositories, without a webhook. It has these
repository permissions and no organization permissions:

- contents, read and write: release commits, tags `vX.Y.Z` and moving `vX`;
- pull requests, read and write: comments of release-please on merged release
  pull requests and template updates;
- issues, read and write: labels that release-please sets on pull requests;
- workflows, read and write: template updates that change workflow files;
- metadata, read.

Workflows obtain a token of the App for each job with
`actions/create-github-app-token`, scoped to the current repository and to the
permissions the job needs. Such tokens expire after one hour. The App's commits
are created through the GitHub API, so GitHub signs them.

The App is the only bypass actor of the ruleset that protects tags `v*`. Its
numeric id is the organization variable `BOT_APP_ID`, used by rulesets, and its
client ID is the organization variable `BOT_CLIENT_ID`, used by workflows. Its
private key is the secret `APP_PRIVATE_KEY` of the GitHub environment `release`
of every repository that releases, and that environment can be deployed only
from `main` and `release/v*` and only after an organization owner approves the
job ([FY-0028](0028-supply-chain.md)). Maintenance branches are `release/vN`
with `N >= 1`: releases `0.y.z` are made only on `main`, and the release
workflow skips pushes to `release/v0`. Only organization owners can create
branches `release/v*`, so the branch policy cannot be met by a new branch.
Organization settings as code create the environment, its branch policy, its
reviewer and the rulesets; the key is set when a repository is set up and when
the key is rotated. The key file is kept outside of any repository.

The private key is rotated once a year and at once when it may have been
exposed; the old key is deleted as soon as the new one is in place.

The App is internal to flakeyard. Consumers bring their own App or none.

### Release gate

In flakeyard, release pull requests are opened and updated with `GITHUB_TOKEN`,
and their checks are dispatched. The key is used only by the job of the release
workflow that creates releases and moves tags `v*`, and only that job of the
release workflow uses the environment `release`. Marking the newest release as
latest runs with `GITHUB_TOKEN` in a job without the environment.

The release workflow first runs a job without the key and without the
environment that decides whether the job with the key is due: when the push is
the merge of a release pull request still labelled `autorelease: pending`; when
such a pull request was merged earlier and the newest push run of its merge
commit completed without a rejection, so this run takes over; or when the tag
`vX` has to be created or moved forward to an existing release. A release whose
job was rejected is not due again on later pushes: the owner either reruns the
rejected run, which asks for approval again, or abandons the version: relabels
the pull request from `autorelease: pending` to `autorelease: abandoned` and
merges a revert of its merge commit, which puts the manifest back at the last
released version. The next release pull request usually has the same version
number, since no tag was created. That decision is not a security boundary: a
wrong decision costs an approval request or delays a release, and the job with
the key derives what it does from the repository when it runs. The decision is
reported in the job summary and as notices of the run. Without an environment,
the job with the key runs on every push, as there is nothing to save.

A run can be approved and rerun only for 30 days. When the merge commit of a
pending release pull request has no push run, or its run is older than 29 days,
the release workflow fails with the instruction to abandon that version, instead
of releasing a commit whose own run can never publish it. When a release exists
but the run of its commit can no longer publish it, the owner cuts the next
patch release.

### Actions may create pull requests

The organization and every released repository allow GitHub Actions to create
and approve pull requests; every other repository does not. Organization
settings as code set the flag at both levels
([FY-0011](0011-settings-as-code.md)), so that a repository created later does
not keep the inherited value.

The flag lets any workflow of a released repository approve a pull request as
`github-actions[bot]`, so anyone who can push, including a third-party app with
write access, could add an approval. While rulesets require no approvals
([FY-0022](0022-bootstrap.md)), that gives nothing that write access does not
already give.

What would change this decision: a ruleset of a released repository requires
approvals, or a second maintainer reviews pull requests. Then the flag is turned
off there, and release pull requests are opened by an App again, through a
second App for pull requests only or through one gated job for both passes.

### Approval

The owner approves a release by checking the repository, not the output of the
run, because a modified caller workflow can write any summary or notice: the run
is a push run of the release workflow at a commit that merges a release pull
request the owner merged, the commits since the previous release are signed
squash merges of reviewed pull requests, `.github/` changed only as reviewed,
and the release workflow is referenced at the commit of one of its release tags.
`org/README.md` of `.github` lists the steps.

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
  earlier steps or jobs.
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
  documented features.
- A client ID without a private key MUST fail every job that needs the key
  before the key is used, and MUST fail the first job of the run when the key is
  visible to that job.
- Workflows MUST NOT accept personal access tokens.
- Workflows MUST NOT require secrets or Apps of flakeyard.
- A release workflow MUST let the caller open release pull requests with
  `GITHUB_TOKEN` also when an App is given, and MUST then start `check-workflow`
  through `workflow_dispatch` after every update of the release pull request.
- Publishing that follows a release MUST run in a workflow run of the release
  commit (the run that created the release, or a rerun of it), never in a
  workflow triggered by the new tag: attestations name the commit of the run.
- Moving the tag `vX` MUST be an input that is off by default.

### Gated release jobs

- A release workflow MUST start a job that uses an environment given by the
  caller only when a job without the key and without the environment has decided
  that it is due: a merged release pull request is pending and this push merged
  it, or the newest push run of its merge commit completed without a rejection;
  or the tag `vX` must be created or moved forward. Without an environment, the
  job MAY run on every push.
- A merged release pull request whose release was rejected MUST NOT make later
  pushes due.
- The job with the key MUST derive what it does from the repository when it
  runs, not from the decision. It MUST NOT check out or run code of the
  repository.
- A release workflow MUST NOT take over a pending release whose merge commit has
  no push run or whose run is older than 29 days, or tell anyone to publish by
  hand; it MUST fail with the instruction to abandon that version. A release
  that exists but whose run can no longer publish it MUST NOT be published from
  another run; the next patch release replaces it.
- Marking the latest release MUST use `GITHUB_TOKEN` and MUST NOT use the
  environment `release`.

### Release bot

- flakeyard automation that creates releases or moves tags MUST use
  `flakeyard-bot`. Automation that opens pull requests MUST use `flakeyard-bot`,
  except release pull requests, which MUST be opened and updated with
  `GITHUB_TOKEN`. Personal access tokens MUST NOT be used.
- `flakeyard-bot` MUST have only the permissions listed in this decision and no
  organization permissions. Adding a permission MUST be recorded in a new `FY`
  ADR.
- Tokens of the App MUST be created per job, scoped to the current repository
  and to the permissions of that job.
- `flakeyard-bot` MUST be the only bypass actor of the ruleset for tags `v*`.
- The private key MUST be stored only as the secret `APP_PRIVATE_KEY` of the
  environment `release`, and `release` MUST be deployable only from `main` and
  `release/v*`. It MUST NOT be an organization or repository secret.
- Rulesets MUST allow only organization owners to create branches `release/v*`.
- Release workflows MUST release only on `main` and `release/vN` with `N >= 1`,
  and MUST skip pushes to `release/v0`.
- Workflows MUST identify the App by `BOT_CLIENT_ID`; rulesets MUST name it by
  `BOT_APP_ID`.
- Only jobs that create releases, move tags `v*` or open pull requests that
  change workflow files MAY use the environment `release`. Jobs that open or
  update release pull requests MUST NOT use it.
- The private key MUST be rotated at least once a year and immediately when it
  may have been exposed.
- Workflows shipped to consumers MUST NOT reference `flakeyard-bot`, its id or
  its key.

### Settings

- "Allow GitHub Actions to create and approve pull requests" MUST be enabled at
  the organization and in every released repository, and MUST be disabled in
  every other repository, both set by organization settings as code.

### Test isolation

- Tests and fixtures of actions and workflows MUST run only in their own
  repositories and MUST NOT be referenced by consumers.

## Consequences

- (+) A consumer's CI changes only when the consumer merges an update of a
  pinned SHA, and the tools inside an action change only with its release.
- (+) Each action and workflow has its own version, changelog and permissions.
- (+) Consumers use flakeyard workflows without creating a GitHub App or sharing
  any secret with flakeyard, and a consumer with an App keeps check runs on its
  release pull requests unless it gates its environment.
- (+) Untrusted input cannot inject shell code through expressions.
- (+) No credential belongs to a person, and every token lives at most an hour
  with the permissions of one job in one repository.
- (+) Only approved jobs on protected branches can obtain the key; other
  workflows and pull request branches cannot.
- (+) Releases start workflows, template updates change workflow files, and
  commits of the App are signed.
- (+) In flakeyard only releases and repairs of `vX` wait for the owner;
  ordinary merges do not.
- (−) The approval is the only defence against a compromised app that can merge
  and release. It is only as good as a check of the repository that does not
  rely on the run's own output, and the owner has to make it for every release.
- (−) A rejected release blocks new release pull requests until the owner reruns
  or abandons it, and abandoning it takes a revert commit.
- (−) A release left waiting for 29 days, or merged without a push run, can no
  longer be published; its version is abandoned.
- (−) A misconfigured key of a gated environment shows up only in a run that
  releases, after the approval.
- (−) Release pull requests of flakeyard rely on `workflow_dispatch` and commit
  statuses for their checks, and are authored by `github-actions[bot]`.
- (−) GitHub Actions may approve pull requests in released repositories; the
  decision has to be revisited before rulesets require approvals.
- (−) Template updates that change workflows still use `release` and wait for
  the owner.
- (−) An action brings its own nixpkgs, so a project downloads tools that may
  already be in its own lock.
- (−) Without an App, consumers who need `vX` must enable it, and consumers that
  open release pull requests with `GITHUB_TOKEN` must allow Actions to create
  pull requests.
- (−) Anyone who can run a workflow with `statuses: write` can post a passing
  status; this is the same exposure as any check run for people with write
  access.
- (−) The key has to be set in every new repository and in every repository at
  each rotation.
- (−) Whoever administers the App can publish releases; App ownership is part of
  the organization's trust.
- (±) API steps depend on the `gh` and `jq` of the runner image instead of a
  pinned version.
- (−) Many small repositories and SHA updates for every release.
- (±) Workflows stay small and change rarely; most changes are releases of
  actions.
- (±) Changes to the App's permissions are rare but need an ADR.

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
- **`GITHUB_TOKEN` only, no App.** No credential to protect, but tags do not
  start workflows and template updates cannot change workflow files.
- **A machine user account.** A second account to secure with its own password
  and two-factor authentication, whose tokens are still personal access tokens.
- **The private key as an organization secret.** One place to set and rotate,
  but every workflow on every branch of every repository could read it.
- **Release pull requests opened with the App in the gated environment.** Check
  runs on release pull requests, but every merge waits for two approvals, one
  for the release pass and one for the pull request.
- **One gated job for both passes.** One approval, but on every push, also when
  nothing is released.
- **The key in a second, ungated environment for release pull requests.** Any
  merged workflow could read it.
- **A second App for pull requests only, its key in an ungated environment.**
  Keeps check runs, and a stolen key gives no more than a merging app already
  has, but adds a second App, a second key to rotate and a second entry in the
  installation check. Kept for the day the flag has to go.
- **Takeover of a rejected release by the next push.** Every push would ask for
  approval again until the owner approved.

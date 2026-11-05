---
id: FY-0031
title: Check pull requests and pushes with workflow-check
status: superseded
date: 2026-11-02
supersedes: []
superseded_by: [FY-0034]
related:
  [
    FY-0003,
    FY-0007,
    FY-0009,
    FY-0012,
    FY-0016,
    FY-0017,
    FY-0022,
    FY-0024,
    FY-0028,
    FY-0029,
    FY-0030,
  ]
affects:
  [
    "workflow-check:**",
    "workflow-scorecard:**",
    "*:.github/workflows/test.yaml",
    "*:.github/workflows/scorecard.yaml",
    ".github:org/repository/rulesets/**",
  ]
review_by:
---

> **Superseded by [FY-0034](0034-check-every-repository-with-workflow-check.md)
> for the organization and by [workflow-check ADR-0001][workflow-check-adr-0001]
> for the design of the workflow.**

## Context

Every flakeyard repository with CI has a check workflow, `test.yaml`, that ends
with a job `required` ([FY-0009](0009-ci-on-runners.md)). Each repository
carries its own copy of that job, of a job `pending` and of the scripts that
post the commit status of dispatched runs
([FY-0030](0030-actions-workflows-and-release-bot.md)). Pull request titles
([FY-0007](0007-branching-and-releases.md)), the form of `renovate.json`
([FY-0029](0029-renovate-mend.md)) and the immutable releases behind preset and
template tags ([FY-0028](0028-supply-chain.md)) are checked only by the
reviewer, and rulesets require no status checks ([FY-0022](0022-bootstrap.md)).
`workflow-check` is the product that ends these exceptions.

Some of these checks cannot be flake checks: they need the event, such as the
title of a pull request, or the GitHub API, such as whether a release is
immutable. Lint tools are owned by the atoms and reach CI as checks of the
repository's flake ([FY-0017](0017-atom-contract.md)), with the same lock as
local runs ([FY-0012](0012-develop-in-container.md)).

A called workflow cannot depend on jobs of its caller, and the caller cannot add
jobs to it. Check runs of jobs inside a called workflow are named
`<caller job> / <called job>`.

A probe in a temporary repository established:

- The job `required` of a workflow called by the job `check` has the check run
  `check / required`. A ruleset entry with that context and `integration_id`
  15368 (the GitHub Actions app) accepts both the check run of a `pull_request`
  run and a commit status with that context posted with `GITHUB_TOKEN`.
- Check runs of `workflow_dispatch` runs are not associated with the pull
  request and do not satisfy the ruleset, even on its head commit. A dispatched
  run counts only through the commit status it posts.
- A failed check run and a success status of the same name on one commit are two
  entries, and both must pass: the pull request stays blocked.
- `job.check_run_id` is set in jobs of a called workflow; the job with that id
  has the name of the check run, also on reruns and after a cancellation.
- `toJSON(needs)` passed as an input gives every job id with its result. With
  `if: ${{ !cancelled() }}` on the caller job, a run cancelled by a newer push
  skips the call and leaves no `check / required` on the old commit; a result
  `cancelled` reaches the input only when the caller job runs after a
  cancellation.
- "Re-run failed jobs" reruns the called workflow, because it depends on the
  failed job. The jobs of attempt `n` include every job of the run, reused ones
  with their earlier result.
- A job with `always()` still runs after its run was cancelled by `concurrency`,
  sees the cancelled job and posts `failure`. The newer run of the same
  concurrency group waits until the cancelled run has finished, so its status is
  posted later.

A second probe ran `ossf/scorecard-action` with `publish_results: true` in a
reusable workflow, in the same repository and in another one:

- Results were published for the calling repository and its SARIF reached the
  caller's code scanning in both cases.
- The signing identity is the called workflow file (`job_workflow_ref`). The
  Scorecard server fetches that file at the ref the caller wrote and verifies
  only it: no top-level `env` or `defaults`, no top-level write permissions,
  `id-token: write` only on the Scorecard job, one Ubuntu runner, no container
  or services, no job `env` or `defaults`, and only steps that use allowlisted
  actions. The caller's file is not checked. The rules are those of
  `api/app/server/verify_workflow.go` in `ossf/scorecard-infra`.
- The analysed branch must be the repository's default branch.
- A tag ref of the called workflow is accepted, but Scorecard counts it as an
  unpinned action and the caller loses its Pinned-Dependencies points.

## Decision

### Product

`flakeyard/workflow-check` delivers `.github/workflows/check.yaml`. Every
repository calls it from `test.yaml` as the last job, with the id `check`. It:

- builds the checks of the repository's flake natively on every system the flake
  declares, after checking that the given runners cover exactly those systems;
- checks the pull request title, `renovate.json` and the immutable releases
  behind preset and template tags;
- ends with the job `required`, whose check run is `check / required`, over its
  own jobs and the caller's;
- posts the commit status of dispatched runs.

Lint tools stay in the atoms: workflow-check builds the checks they define and
bundles no tools of its own. Checks that need the event or the API are steps of
the workflow that use the API and the runner's `bash`, `gh` and `jq`. They read
repository files through the contents API at the checked commit and do not check
out code; only the job that builds the flake checks out the repository.

### Caller contract

The caller passes `toJSON(needs)` as the required input `needs` and lists every
other job of its workflow in the `needs` of `check`. `required` treats a missing
or malformed input as a failure, namespaces the caller's jobs apart from its own
and also reads the jobs of the current run attempt from the API. The jobs of the
check workflow are `required` itself and the jobs that `required` needs, named
`check / <id>`. Every other job of the attempt is the caller's, also one whose
own name looks like `check / lint`. A caller's job that has not completed is
missing from `needs`. A caller's job passes only when it succeeded, or when it
was skipped and `caller-may-skip` names it, listed in `needs` or not. Jobs of
the attempt are matched with `caller-may-skip` by their names without matrix
values, so a job that may be skipped sets no `name:`. The caller job runs with
`if: ${{ !cancelled() || github.event_name == 'workflow_dispatch' }}`, so that a
superseded pull request run leaves no result and a dispatched run reports one on
every outcome.

### Required context

The only required status check is `check / required` with GitHub Actions as its
source. The workflow takes the context of its commit status from the name of its
own job, found through `job.check_run_id`, so the status and the check run
always have the same name. Branch rulesets require it with
`strict_required_status_checks_policy` on and `do_not_enforce_on_create` on.

### Dispatched runs

When the caller's event is `workflow_dispatch`, the job `pending` posts
`pending` on the checked commit and `required` posts `success` or `failure` with
`always()`, also after a cancellation. Runs of other events post no status. The
first status appears only when `check` starts, after the caller's own jobs; a
run that never reaches `check` posts nothing and the pull request stays blocked.

### Public API

Inputs: `needs` (required), `caller-may-skip`, `flake` (`""` disables the flake
jobs), `systems` (an array of `{system, runner}`), `expect-checks`,
`extra-substituters`, `extra-trusted-public-keys`, `timeout-minutes`,
`pr-title`, `pr-title-types`, `renovate-config`, `renovate-presets`,
`release-tags` and `allow-unpinned`. The booleans `pr-title`, `renovate-config`
and `release-tags` switch single checks and are `true` by default; a
switched-off check succeeds with a notice. No secrets and no outputs.
Permissions: `actions: read`, `contents: read`, `pull-requests: read`,
`statuses: write`. Events: `pull_request` (`opened`, `synchronize`, `reopened`,
`edited`), `push`, `schedule` and `workflow_dispatch`. Runners: GitHub-hosted
Ubuntu x86_64 and arm64. The context `check / required`, the caller contract and
the default title rule are part of the public API.

The check of `flake.lock` by `action-flake-lock` ([FY-0016](0016-pin-form.md))
and the push to the binary cache are added in later minor versions.

### OpenSSF Scorecard

Scorecard is not part of `check.yaml`: it runs on other events and needs
`id-token: write` and `security-events: write`, which every pull request run
would otherwise get, and the server rejects a workflow file with other jobs or
steps. It comes from a separate product, `flakeyard/workflow-scorecard`, whose
reusable workflow holds only the Scorecard job and meets the server's rules.

Every repository calls it from its own workflow file, `scorecard.yaml`, pinned
by full commit SHA, on `push` to the default branch, `schedule` and
`workflow_dispatch`, with `contents: read`, `actions: read`,
`security-events: write` and `id-token: write` on the calling job. Each
repository adds it in the pull request that adopts workflow-check.

`workflow-scorecard` itself does not call its product: its `scorecard.yaml` is
the reusable workflow and also runs on `push`, `schedule` and
`workflow_dispatch`, so the repository still has exactly one Scorecard workflow.

The four permissions are those of a public repository. Scorecard asks for more
read permissions in private repositories; flakeyard has none, and a private
repository generated from a template keeps the four until a later version of
this decision covers it.

## Constraints

### Caller

- Every repository with CI MUST call `check.yaml` from `test.yaml` as a job with
  the id `check` and without `name:`, and that job MUST be the last one: no job
  MAY depend on it.
- The job `check` MUST list every other job of the workflow in `needs`, MUST
  pass `toJSON(needs)` as the input `needs` and MUST run with
  `if: ${{ !cancelled() || github.event_name == 'workflow_dispatch' }}`.
- `test.yaml` MUST NOT cancel a run in progress for the `pull_request` action
  `edited`.

### Required check

- The only required status check of branch rulesets MUST be `check / required`
  with `integration_id` 15368, with `strict_required_status_checks_policy` and
  `do_not_enforce_on_create` set to `true`.
- `required` MUST fail when the input `needs` is missing or malformed, when a
  job in it did not succeed and is not allowed to be skipped, and when a job of
  the run attempt outside the check workflow has not completed, or neither
  succeeded nor was skipped with its name in `caller-may-skip`.
- `required` MUST take as the jobs of the check workflow only `required` and the
  jobs it needs, not every job named like `check / *`.
- `caller-may-skip` MUST be matched with the names of the jobs of the attempt
  without matrix values. A caller's job that may be skipped MUST NOT set
  `name:`.
- The context of the commit status MUST be derived from the job's own name
  through `job.check_run_id`, not from an input.

### Commit status

- When the caller's event is `workflow_dispatch`, `check.yaml` MUST post
  `pending` on the checked commit when `check` starts and MUST post `success` or
  `failure` with `always()`, also when the run was cancelled.
- Runs of other events MUST NOT post commit statuses.

### Checks

- Checks that need the event or the GitHub API MUST be API steps. Repository
  files MUST be read through the contents API at the checked commit, and jobs
  other than the one that builds the flake MUST NOT check out code.
- Lint tools MUST come from the atoms through the repository's flake, not from
  `check.yaml`.
- Every check that applies to a flakeyard repository MUST be switchable by its
  own boolean input, `true` by default. A switched-off or non-applicable check
  MUST succeed with a notice and MUST NOT be skipped.
- The workflow MUST fail when the systems the flake declares differ from the
  systems of the runners it is given.
- Owners, repositories and tags read from files MUST be validated before they
  are used in an API path, and untrusted text MUST be escaped before it is
  printed as a workflow command or in the job summary.

### Workflow

- `check.yaml` MUST accept only the events listed in this decision and MUST fail
  on any other.
- `check.yaml` MUST require no secrets, no environment and no GitHub App, and
  MUST ask for no permissions beyond those listed in this decision.

### Scorecard

- The reusable workflow of `workflow-scorecard` MUST contain only the Scorecard
  job and MUST meet the rules the Scorecard server verifies.
- Every repository MUST run Scorecard from exactly one workflow,
  `scorecard.yaml`, which MUST call `workflow-scorecard` pinned by full commit
  SHA and MUST run only on the default branch, on `push`, `schedule` and
  `workflow_dispatch`. In `workflow-scorecard`, `scorecard.yaml` is the reusable
  workflow itself and runs on these events instead of calling it.
- The calling job MUST have exactly the permissions `contents: read`,
  `actions: read`, `security-events: write` and `id-token: write`, and the
  workflow MUST have no top-level write permissions.

## Consequences

- (+) The required-check logic and the status protocol live in one released
  product instead of a copy in every repository.
- (+) Titles, `renovate.json` and immutable preset and template tags are checked
  in CI, and rulesets can require a check.
- (+) A job forgotten in `needs` fails the check instead of passing silently.
- (+) A status posted by another workflow cannot unblock a pull request whose
  check run failed.
- (+) The Scorecard job and its pins live in one product; a repository keeps
  only a short caller.
- (−) The checks of workflow-check start only after the caller's own jobs, so
  repositories with long own jobs wait longer.
- (−) A dispatched run shows `pending` only when `check` starts, and none at all
  when the run never reaches it.
- (−) The required context depends on the caller's job id; a caller that names
  it differently is blocked by the ruleset.
- (−) An edit of a pull request description reruns the whole check workflow.
- (−) Repositories without a flake still run their lints only locally until the
  atoms are released.
- (±) The verdict depends on the jobs API and on the runner's `gh` and `jq`.

## Alternatives considered

- **`check` in parallel, `required` polling the jobs API.** Needs no `needs`,
  posts `pending` at once and runs in parallel, but it polls, holds a runner
  while it waits and has to exclude itself and filter by attempt. Kept as the
  fallback if the sequence costs too much.
- **The caller keeps `pending` and `required` around a shared action.** Keeps
  the literal check `required`, but every repository keeps the copied jobs and
  the status protocol.
- **Waiting for the atoms.** Nothing to build now, but titles, presets and
  rulesets would stay unchecked.
- **A plain context `required`.** Only a job of the caller can produce it.
- **Lint tools bundled in workflow-check.** A second owner of each tool and
  different versions in CI and local runs.
- **Scorecard inside `check.yaml`.** Every pull request run would get the
  permissions Scorecard needs, and the server would reject the file.
- **A Scorecard job copied into every repository.** Works, but every repository
  carries the job and its pins and updates them on its own.

[workflow-check-adr-0001]:
  https://github.com/flakeyard/workflow-check/blob/main/docs/adr/records/0001-check-pull-requests-and-pushes.md

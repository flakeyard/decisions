---
id: FY-0034
title: Check every repository with workflow-check and workflow-scorecard
status: accepted
date: 2026-11-05
supersedes: [FY-0031]
superseded_by: []
related: [FY-0001, FY-0009, FY-0022, FY-0028, FY-0030]
affects:
  [
    "*:.github/workflows/test.yaml",
    "*:.github/workflows/scorecard.yaml",
    "template-*:template/.github/workflows/**",
    "workflow-check:.github/workflows/check.yaml",
    "workflow-scorecard:**",
    ".github:org/repository/rulesets/**",
  ]
review_by:
---

## Context

Every flakeyard repository with CI checks pull requests and pushes with the
reusable workflow of `workflow-check`, which ends with the job `required`
([FY-0009](0009-ci-on-runners.md)). The design of that workflow, including the
probes behind its required context and its caller contract, is recorded in
[workflow-check ADR-0001][workflow-check-adr-0001]. The rules that bind the
repositories that call it, and their branch rulesets, are recorded here.

Every repository also runs OpenSSF Scorecard ([FY-0028](0028-supply-chain.md)).
A probe ran `ossf/scorecard-action` with `publish_results: true` in a reusable
workflow, in the same repository and in another one:

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

This record moves the organization part of
[FY-0031](0031-check-pull-requests-and-pushes-with-workflow-check.md) unchanged
in substance; the product part moves to [workflow-check
ADR-0001][workflow-check-adr-0001], following
[FY-0001](0001-record-decisions.md).

## Decision

### Check workflow

Every flakeyard repository with CI, and every project generated from a flakeyard
template, calls `.github/workflows/check.yaml` of `flakeyard/workflow-check`
from `test.yaml` as the last job, with the id `check`. Its check run
`check / required` is the only required status check of branch rulesets, with
GitHub Actions as its source. When the caller's event is `workflow_dispatch`,
the workflow posts the commit status of the run, as every check workflow does
([FY-0030](0030-actions-workflows-and-release-bot.md)).

The caller passes `toJSON(needs)` as the input `needs`, lists every other job of
its workflow in the `needs` of `check`, and runs `check` with
`if: ${{ !cancelled() || github.event_name == 'workflow_dispatch' }}`. A
caller's job that may be skipped sets no `name:`, because `caller-may-skip` is
matched with job names without matrix values.

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

### Bootstrap

`workflow-check` and `workflow-scorecard` are step 5 of the bootstrap order of
[FY-0022](0022-bootstrap.md). A repository adopts both in its first pull request
after their release; until then it runs their checks as FY-0022 describes.

## Constraints

### Caller

- Every repository with CI, and every project generated from a flakeyard
  template, MUST call `check.yaml` from `test.yaml` as a job with the id `check`
  and without `name:`, and that job MUST be the last one: no job MAY depend on
  it.
- The job `check` MUST list every other job of the workflow in `needs`, MUST
  pass `toJSON(needs)` as the input `needs` and MUST run with
  `if: ${{ !cancelled() || github.event_name == 'workflow_dispatch' }}`.
- `test.yaml` MUST NOT cancel a run in progress for the `pull_request` action
  `edited`.

### Required check

- The only required status check of branch rulesets MUST be `check / required`
  with `integration_id` 15368, with `strict_required_status_checks_policy` and
  `do_not_enforce_on_create` set to `true`.
- A caller's job that may be skipped MUST NOT set `name:`.

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

- (+) The Scorecard job and its pins live in one product; a repository keeps
  only a short caller.
- (±) The design of the check workflow, its consequences and the alternatives to
  it are recorded in [workflow-check ADR-0001][workflow-check-adr-0001].

## Alternatives considered

- **A Scorecard job copied into every repository.** Works, but every repository
  carries the job and its pins and updates them on its own.

[workflow-check-adr-0001]:
  https://github.com/flakeyard/workflow-check/blob/main/docs/adr/records/0001-check-pull-requests-and-pushes.md

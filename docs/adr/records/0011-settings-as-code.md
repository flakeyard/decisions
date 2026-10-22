---
id: FY-0011
title: Keep organization settings as code
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0002, FY-0003, FY-0007, FY-0009, FY-0010]
affects: [".github:org/**"]
review_by:
---

## Context

flakeyard consists of many small repositories
([FY-0002](0002-one-repository-per-concern.md)), and several decisions are
enforced by settings rather than by code: squash merges and signed commits on
`main` ([FY-0007](0007-branching-and-releases.md)), tags only from the bot and
the environment that holds its key ([FY-0010](0010-release-bot.md)), secret
scanning, immutable releases, the actions that workflows may use. Consumers
trust these settings without being able to see them.

The organization is on GitHub Free. It has no organization-level rulesets and no
custom repository roles, so every repository needs its own rulesets. Settings
changed by hand in the web interface leave no history and drift apart between
repositories.

## Decision

Settings of the organization and of its repositories are kept as code in
`flakeyard/.github` under `org/`, as JSON files and two idempotent scripts that
use the GitHub API through `gh`:

- `settings.sh` applies organization settings: base permissions, who may create
  repositories, and Actions policies (allowed actions, required SHA pinning,
  read-only default `GITHUB_TOKEN` that cannot approve pull requests, approval
  of workflows from external contributors, retention, no self-hosted runners).
- `apply.sh` applies repository settings to every repository, or to the
  repositories it is given: merge settings, labels, topics, rulesets, the
  environment `release`, secret scanning with push protection, Dependabot
  alerts, private vulnerability reporting, immutable releases and CodeQL.

Repository-specific differences are derived only from repository name prefixes
([FY-0003](0003-repository-naming.md)), such as topics per prefix or CodeQL for
atoms and actions.

An organization owner runs the scripts from their own machine after the change
is merged, and runs `apply.sh` for every new repository before its first pull
request. The scripts do not run in CI: that would need an organization
administration token stored in the organization.

Settings without an API, or that depend on external services, are listed in
`org/README.md` under "Not covered", each with what has to be done by hand.

## Constraints

- Every organization and repository setting that has an API MUST be defined in
  `.github/org/` and applied by `settings.sh` or `apply.sh`.
- A setting MUST be changed by a pull request to `.github` first and applied
  after the merge. Changes made only in the web interface MUST be treated as
  drift and are overwritten by the next run.
- Settings without an API MUST be listed in `org/README.md` with the manual
  steps.
- The scripts MUST be idempotent and MUST be safe to run for all repositories at
  once.
- Differences between repositories MUST be derived only from name prefixes,
  except for `.github` and `decisions`, which MAY be named explicitly.
- `apply.sh` MUST be run for a new repository before it receives its first pull
  request.
- Third-party actions MUST be allowed by `settings.sh` before workflows use
  them.
- The scripts MUST NOT run with a token stored in the organization.

## Consequences

- (+) Settings have a history, a review and the same values in every repository.
- (+) Consumers and contributors can read the rules that protect the
  repositories they depend on.
- (+) A new repository gets every protection with one command.
- (−) Drift is found only when a script runs again; nothing detects a change
  made in the web interface.
- (−) Applying settings depends on an owner running the scripts.
- (±) Moving to a plan with organization rulesets would move part of `apply.sh`
  to `settings.sh`.

## Alternatives considered

- **Settings by hand in the web interface.** No history, and values drift apart
  across many repositories.
- **Terraform with the GitHub provider.** Declarative with drift detection, but
  needs a state backend, credentials for it and a provider that lags behind new
  settings such as immutable releases.
- **A settings GitHub App** (`safe-settings` or similar). Applies settings
  continuously, but needs hosting or a third party with administration rights
  over every repository.
- **Running the scripts in CI.** Automatic, but an administration token of the
  organization would become a secret that workflows can read.

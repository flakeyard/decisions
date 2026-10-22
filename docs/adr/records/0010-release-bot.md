---
id: FY-0010
title: Release bot as a GitHub App
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0007, FY-0008, FY-0009]
affects: [".github:org/**", "*:.github/workflows/**", "workflow-*:**"]
review_by:
---

## Context

Releases of flakeyard create tags `vX.Y.Z`, move tags `vX` and open release pull
requests ([FY-0007](0007-branching-and-releases.md)). flakeyard's own
repositories are also kept up to date from a template by automated pull
requests, some of which change workflow files. These steps need a token that the
built-in `GITHUB_TOKEN` cannot replace: pull requests and tags created with it
do not start other workflows, and it cannot change files under
`.github/workflows/`.

Rulesets protect tags `v*` from everyone except one bypass actor. That actor
holds the most powerful credential of the organization: with it, a release can
be published from any commit.

A personal access token would tie that credential to a person, live until it is
revoked and carry every permission of the person's account.

## Decision

flakeyard uses one GitHub App, `flakeyard-bot`, owned by the organization,
installed only there, on all repositories, without a webhook. It has these
repository permissions and no organization permissions:

- contents, read and write: release commits, tags `vX.Y.Z` and moving `vX`;
- pull requests, read and write: release pull requests and template updates;
- issues, read and write: labels that release-please sets on pull requests;
- workflows, read and write: template updates that change workflow files;
- metadata, read.

Workflows obtain a token of the App for each job with
`actions/create-github-app-token`, scoped to the current repository and to the
permissions the job needs. Such tokens expire after one hour. The App's commits
are created through the GitHub API, so GitHub signs them.

The App is the only bypass actor of the ruleset that protects tags `v*`. Its id
is the organization variable `BOT_APP_ID`. Its private key is a secret of the
GitHub environment `release` of every repository that releases, and that
environment can be deployed only from `main` and `release/v*`. Organization
settings as code create the environment and its branch policy; the key is set
when a repository is set up and when the key is rotated.

The private key is rotated once a year and at once when it may have been
exposed; the old key is deleted as soon as the new one is in place.

The App is internal to flakeyard. Consumers bring their own App or none
([FY-0008](0008-actions-and-workflows.md)).

## Constraints

- flakeyard automation that creates releases, moves tags or opens pull requests
  MUST use `flakeyard-bot`. Personal access tokens MUST NOT be used.
- `flakeyard-bot` MUST have only the permissions listed in this decision and no
  organization permissions. Adding a permission MUST be recorded in a new `FY`
  ADR.
- Tokens of the App MUST be created per job, scoped to the current repository
  and to the permissions of that job.
- `flakeyard-bot` MUST be the only bypass actor of the ruleset for tags `v*`.
- The private key MUST be stored only as a secret of the environment `release`,
  and `release` MUST be deployable only from `main` and `release/v*`. It MUST
  NOT be an organization or repository secret.
- Only jobs that release, move tags or open automated pull requests MAY use the
  environment `release`.
- The private key MUST be rotated at least once a year and immediately when it
  may have been exposed.
- Workflows shipped to consumers MUST NOT reference `flakeyard-bot`, its id or
  its key.

## Consequences

- (+) No credential belongs to a person, and every token lives at most an hour
  with the permissions of one job in one repository.
- (+) Only bot jobs on protected branches can obtain the key; other workflows
  and pull request branches cannot.
- (+) Releases start workflows, change workflow files and are signed.
- (−) The key has to be set in every new repository and in every repository at
  each rotation.
- (−) Whoever administers the App can publish releases; App ownership is part of
  the organization's trust.
- (±) Changes to the App's permissions are rare but need an ADR.

## Alternatives considered

- **`GITHUB_TOKEN` only.** No credential to protect, but release pull requests
  get no checks, tags do not start workflows and template updates cannot change
  workflow files.
- **A personal access token of a maintainer.** Long-lived, tied to a person and
  scoped to everything the person can do.
- **A machine user account.** A second account to secure with its own password
  and two-factor authentication, whose tokens are still personal access tokens.
- **The private key as an organization secret.** One place to set and rotate,
  but every workflow on every branch of every repository could read it.

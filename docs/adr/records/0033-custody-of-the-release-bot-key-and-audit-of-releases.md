---
id: FY-0033
title: Custody of the release bot key and audit of releases
status: accepted
date: 2026-11-03
supersedes: []
superseded_by: []
related:
  [FY-0007, FY-0011, FY-0024, FY-0028, FY-0029, FY-0030, FY-0031, FY-0032]
affects:
  [
    ".github:org/README.md",
    ".github:org/audit/**",
    ".github:org/organization/installations.json",
    ".github:.github/workflows/audit.yaml",
    ".github:SECURITY.md",
  ]
review_by:
---

## Context

`flakeyard-bot` is the only bypass actor of the ruleset that protects tags `v*`
([FY-0030](0030-actions-workflows-and-release-bot.md)). Its private key is the
secret `APP_PRIVATE_KEY` of the environment `release`, which only an approved
job on `main` or `release/v*` can read. The approval protects the secret, not
the key itself: whoever holds a copy of the key can create an installation token
from any machine, without GitHub Actions and without the environment, and with
it create a tag `vX.Y.Z` and an immutable release from any commit of any
repository. Consumers' Renovate then proposes that release as an ordinary
update.

FY-0030 kept the key file outside of any repository, on the owner's machine, and
rotated the key once a year. It said nothing about how a leak would be noticed
or what to do then. A design review found that nothing in the organization would
notice a release made outside the release process.

GitHub Apps and secrets behave in ways that shape the answer:

- An App can have several private keys at once. A key deleted in the App
  settings creates no new tokens; tokens created before expire within an hour.
- A new private key is generated only in the App settings, and the browser
  downloads it as a file.
- A secret cannot be read back, neither in the web interface nor through the
  API.
- A release made with the key is authored by the App, like a genuine one. Its
  tag, however, points to a commit that is not the merge commit of a release
  pull request, unless the attacker also merged one into a protected branch.
- Rulesets require no approval, so the key can open, label and merge such a pull
  request. It merges as the App: `merged_by` of the pull request is
  `flakeyard-bot[bot]`, not the owner. The list of pull requests does not carry
  `merged_by`; each pull request has to be read on its own.
- Every flakeyard repository is public, so `GITHUB_TOKEN` of any repository can
  read the releases, tags and pull requests of all of them.

FY-0030 also gave the App `workflows: write`. A leaked key with that permission
could change the workflows of every repository, but the App cannot release
without it: GitHub refuses a ref made by an App, such as the tag of a release,
whose commits carry workflow files unless the App may write workflows, and the
commits of every tag do. A release of image-devcontainer failed this way while
the permission was removed.

## Decision

### Custody of the key

No copy of the key is kept outside GitHub. Since a secret cannot be read back,
storing the key always means generating a new one, for a new released
repository, for the yearly rotation and after a leak alike:

1. generate a new private key in the App settings;
2. store it in every released repository with
   `BOT_PRIVATE_KEY_FILE=<file> org/apply.sh`, without a list of repositories,
   so that no repository keeps an older key;
3. delete the file;
4. delete every older key in the App settings.

`org/README.md` in `.github` describes this procedure.

### Leaked key

When the key may have been exposed, the owner:

1. deletes the key in the App settings;
2. stores a new key as above;
3. runs the release audit and examines every mismatch it reports;
4. reviews every tag `v*` and every release created since the key may have been
   exposed, also those the audit passes;
5. deletes forged releases and their tags, moves forged `vX` back, and never
   reuses the version of a forged release;
6. tells consumers through a security advisory in every affected repository.

This runbook is in `org/README.md`, and `SECURITY.md` links it.

Step 5 is the only exception to the rule of
[FY-0024](0024-versioning-and-support.md) that tags `vX.Y.Z` and their releases
are never changed or deleted. It applies only to a release or tag shown to be
forged: one that the release process did not create. Only the App may change
tags `v*`, so the owner turns off the ruleset `release-tags` of that repository
for the change; the next run of `apply.sh` turns it back on.

### Release audit

A workflow `audit.yaml` in `.github` runs every week and on `workflow_dispatch`.
For every release `vX.Y.Z` of every non-archived repository of the organization,
it checks that:

- the release was made by `flakeyard-bot`;
- the commit of its tag is the merge commit of a merged pull request from the
  branch `release-please--branches--<base>` of the same repository into
  `<base>`, which is `main` or `release/vN` with `N >= 1`, labelled
  `autorelease: tagged` ([FY-0007](0007-branching-and-releases.md));
- that pull request was merged by a reviewer of the environment `release`
  (`org/repository/environments/release-reviewers.json` in `.github`).

It does not check the moving tags `vX`; the runbook reviews them by hand. It
reports every mismatch and then fails; a failed run is the alert. It reads with
a read-only `GITHUB_TOKEN`. Its logic is a script under `org/audit/` with unit
tests against a stub of `gh`; the workflow only runs it.

The audit checks the organization, not a commit. It is therefore not a check
workflow in the sense of FY-0030 and
[FY-0031](0031-check-pull-requests-and-pushes-with-workflow-check.md): the
commit status protocol of dispatched runs does not apply to it, it posts no
commit status, and no ruleset requires it.

### Permissions of the App

`flakeyard-bot` keeps the permission `workflows`, which releases need, and
`org/organization/installations.json` lists it
([FY-0029](0029-renovate-mend.md)). Its tokens ask for it only in the jobs that
create releases or move tags, and the key custody, the gate and the audit above
bound what a leaked key can do with it.

## Constraints

- The private key of `flakeyard-bot` MUST NOT be kept outside GitHub. The file
  of a new key MUST be deleted once the key is stored, and older keys MUST be
  deleted in the App settings once the new key is stored in every released
  repository.
- Storing the key, for a new repository or a rotation, MUST store a newly
  generated key in every released repository.
- `org/README.md` MUST describe the custody procedure and the runbook for a
  leaked key, and `SECURITY.md` MUST link the runbook.
- A tag `v*` or a release MAY be deleted or moved only when it is shown to be
  forged, by an owner, with the ruleset `release-tags` of that repository turned
  off only for that change. The version of a forged release MUST NOT be reused.
- The release audit MUST run at least weekly and on `workflow_dispatch`, MUST
  cover every release `vX.Y.Z` of every non-archived repository, and MUST fail
  on any release that was not made by `flakeyard-bot`, whose tag is not the
  merge commit of a release pull request as described in this decision, or whose
  release pull request was not merged by a reviewer of the environment
  `release`.
- The release audit MUST use only a read-only `GITHUB_TOKEN`. Its logic MUST
  live in a script with unit tests, not in the workflow.
- The release audit MUST NOT post commit statuses and MUST NOT be a required
  check.

## Consequences

- (+) No copy of the key waits on a machine to leak; a leak needs an approved
  job that runs, or the few minutes of a rotation.
- (+) A release made with a stolen key from an arbitrary commit is found within
  a week, and the owner has a procedure for it.
- (−) A leaked key can change workflow files; the audit and the runbook are what
  find and undo such changes.
- (−) Every new released repository rotates the key of every released
  repository.
- (−) The audit finds a forged release only after the fact; consumers may have
  merged it by then.
- (+) A release pull request merged with a stolen key is found too, since the
  App, not the owner, merged it.
- (−) The audit does not detect a release pull request merged with the owner's
  own stolen session, nor forged tags `vX`, images or deleted releases; step 4
  of the runbook covers them by hand.

## Alternatives considered

- **Keeping the key file on the owner's machine.** Needed for nothing, since a
  new key can always be generated, and the machine becomes part of the trust of
  every release.
- **Checking only the author of releases.** A stolen key produces the same
  author; only the link to a merged release pull request tells them apart.
- **The audit as a job of `workflow-check`.** It would run per commit in every
  repository, while the audit is about the organization.
- **Removing `workflows: write`.** Tried, and releases failed: GitHub refuses
  the tag of a release made by an App without it.

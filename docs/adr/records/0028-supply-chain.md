---
id: FY-0028
title: Secure the supply chain
status: accepted
date: 2026-11-02
supersedes: [FY-0018]
superseded_by: []
related:
  [
    FY-0003,
    FY-0005,
    FY-0007,
    FY-0009,
    FY-0011,
    FY-0013,
    FY-0016,
    FY-0021,
    FY-0022,
    FY-0024,
    FY-0029,
    FY-0030,
    FY-0031,
  ]
affects:
  [
    ".github:org/**",
    "*:.github/workflows/**",
    "*:renovate.json",
    "renovate-config:**",
    "image-*:**",
    "template-*:template/**",
  ]
review_by:
---

## Context

Consumers run flakeyard code in their CI with their permissions and secrets,
evaluate it in their flakes and develop inside its image. A compromise of
flakeyard, of a dependency of flakeyard or of a release in transit becomes a
compromise of every consumer that updates.

Several decisions already close parts of this: pull requests and signatures on
`main` ([FY-0007](0007-branching-and-releases.md)), SHA references and no
untrusted interpolation in actions
([FY-0030](0030-actions-workflows-and-release-bot.md)), pushes to the binary
cache only from protected branches ([FY-0009](0009-ci-on-runners.md)), the
release bot and its key ([FY-0030](0030-actions-workflows-and-release-bot.md)),
settings applied from code ([FY-0011](0011-settings-as-code.md)). What is
missing is how consumers verify what they receive, how dependencies of flakeyard
are updated and how the posture is measured.

Renovate knows release dates only for some datasources, such as GitHub Actions
and container images. Flake inputs follow branches or moving tags
([FY-0016](0016-pin-form.md)) and have no release dates: a minimum release age
either holds their updates back forever or is ignored with a warning.
Vulnerability databases do not cover flake inputs either.

Designing the Renovate preset showed what [FY-0018](0018-supply-chain.md) did
not account for:

- Renovate presets and Copier templates are resolved by tag. Renovate's
  `renovate-config` manager updates only references pinned to a version tag,
  `github>flakeyard/renovate-config#vX.Y.Z`; a commit SHA in that place is never
  updated. Copier records the tag it rendered in `.copier-answers.yml` and
  selects versions by tags ([FY-0007](0007-branching-and-releases.md)).
- GitHub immutable releases lock the tag of a release to its commit: the tag
  cannot be moved or deleted while the release exists, and its name cannot be
  reused. Organization settings enable immutable releases in every repository,
  and only `flakeyard-bot` can create tags `v*`. The moving tag `vX` has no
  release ([FY-0007](0007-branching-and-releases.md)) and is not locked.
- For GitHub tags, Renovate takes the release date from the later of the commit
  or tagger date and the publication date of the GitHub release. Commit and
  tagger dates are set by whoever creates the tag, so a tag without a release
  can claim any age. Updates without a release date, such as images on the
  GitHub Container Registry, are proposed at once, because the minimum release
  age must not hold them back forever.
- Dependabot creates no alerts for actions pinned to a commit SHA, and Renovate
  has no vulnerability data for GitHub tags, flake inputs or container images. A
  fix of a published vulnerability in an action therefore waits for the minimum
  release age like any other update.
- The Renovate app hosted by Mend ([FY-0029](0029-renovate-mend.md)) is a
  third-party GitHub App with write access to code, workflows, pull requests,
  checks and commit statuses of every repository. Rulesets on `main` require
  pull requests but no approvals, and no status checks yet
  ([FY-0022](0022-bootstrap.md)). Whoever controls the app could open and merge
  a pull request, then merge the release pull request, and `flakeyard-bot` would
  publish the result to every consumer.

[FY-0021](0021-release-bot.md) did not put required reviewers on the environment
`release`, so that releases would not wait for a person. A third party with
write access to every repository changes that trade-off.

This record replaces [FY-0018](0018-supply-chain.md) with these findings;
everything else is unchanged.

## Decision

### Sources

- Commits on `main` and `release/v*` are signed, and release tags are created
  only by the bot.
- Every dependency is pinned: flake inputs by `flake.lock`, actions by full
  commit SHA (required by organization settings), the image's packages by its
  `flake.lock`, Renovate presets and Copier templates by the tag `vX.Y.Z` of an
  immutable GitHub release. Third-party actions must be in the allowlist of
  organization settings.
- Renovate proposes every update as a pull request that passes CI before it is
  merged. Updates of third-party dependencies with release dates are proposed
  only after a minimum release age of 3 days, so that a compromised release has
  time to be noticed and withdrawn. Flake inputs, updates of flakeyard's own
  repositories and updates that fix a published vulnerability are proposed at
  once.

### Tags of immutable releases

A tag `vX.Y.Z` of an immutable GitHub release of flakeyard is an immutable
reference. It is used only where the consumer can resolve nothing but a tag:
Renovate presets and Copier templates. Everything that can be pinned by commit
SHA or digest still is. The moving tag `vX` is never used for presets or
templates, because it has no release and moves. A repository's CI checks that
every such tag it references belongs to an immutable release.

The release, not the tag, is what makes the reference immutable: deleting the
release would free the tag. Releases are never deleted
([FY-0024](0024-versioning-and-support.md)).

### Limits of the minimum release age

The minimum release age delays an update only by the date its datasource
reports. A tag pushed without a GitHub release, with an old commit or tagger
date, passes the age at once; updates without a date get no delay at all. The
delay therefore protects against a compromised release that is noticed and
withdrawn, not against an attacker who controls the upstream repository. Review
of the diff and CI remain the controls for the rest.

### Vulnerability fixes

Fixes of published vulnerabilities skip the delay and the schedule only when
Renovate receives an alert for them. Actions pinned by SHA, flake inputs and
images receive none. Flake inputs and images without release dates are proposed
at once anyway; a vulnerable action is updated by a maintainer, who starts the
pending update from the Dependency Dashboard. Documentation does not promise
more.

### Releases

- Releases and their tags `vX.Y.Z` are immutable.
- Every published artifact, the image and any release asset, gets a build
  provenance attestation from `actions/attest-build-provenance`, and the image
  an SBOM attestation as well. Consumers verify them with
  `gh attestation verify` against the `flakeyard` owner.
- Git-only releases are consumed by commit SHA (atoms, actions, workflows) or by
  the tag of their immutable release (templates, the Renovate preset); the SHA
  or the release is what consumers verify.

### Third-party apps

A third-party GitHub App with write access is installed only when an accepted
`FY` ADR names it and its permissions, and organization settings check its
installation. Such an app must not be able to publish a release on its own:

- the environment `release` requires the approval of an organization owner
  before a job can use it, so no run obtains the key of `flakeyard-bot` without
  a person;
- required status checks, when rulesets get them, name GitHub Actions as their
  source, so a commit status set by another app cannot satisfy them.

### Detection

- Every repository has secret scanning with push protection, Dependabot alerts
  and private vulnerability reporting; atoms and actions have CodeQL.
- Workflows are linted by actionlint and zizmor through `atom-workflows`.
- Every repository runs OpenSSF Scorecard on a schedule and publishes its
  results, through the reusable workflow of `workflow-scorecard`
  ([FY-0031](0031-check-pull-requests-and-pushes-with-workflow-check.md));
  findings are fixed or recorded as accepted with a reason, such as Code-Review
  while one maintainer merges their own pull requests.

## Constraints

- Every dependency MUST be pinned to an immutable reference: a locked flake
  input, a full commit SHA, a digest or, for Renovate presets and Copier
  templates only, the tag `vX.Y.Z` of an immutable GitHub release.
- Renovate presets and Copier templates MUST NOT be referenced by a moving tag
  `vX` or by a branch.
- CI of a flakeyard repository MUST check that every preset or template tag it
  references belongs to an immutable GitHub release.
- Third-party actions MUST be on the allowlist of organization settings before
  they are used.
- Dependency updates MUST arrive as pull requests that pass CI. Updates of
  third-party dependencies whose datasource has release dates MUST wait at least
  3 days after their release. Flake inputs, updates of flakeyard's own
  repositories and updates that fix a published vulnerability MAY be proposed at
  once. The minimum release age MUST NOT be configured so that updates without
  release dates are held back indefinitely.
- Documentation MUST NOT claim that the minimum release age protects against a
  compromised upstream repository, or that fixes of vulnerabilities in actions,
  flake inputs or images are proposed at once.
- A third-party GitHub App with write access MUST NOT be installed unless an
  accepted `FY` ADR names it and its permissions, and organization settings MUST
  check its installation.
- The environment `release` MUST require the approval of an organization owner.
- Required status checks MUST name GitHub Actions as their source.
- Every published artifact MUST have a build provenance attestation; the image
  MUST also have an SBOM attestation.
- Releases MUST be built by CI from the tagged commit; artifacts MUST NOT be
  uploaded from a developer machine.
- Every repository MUST enable the detection features listed in this decision
  through organization settings and MUST run OpenSSF Scorecard on a schedule.
- `VERSIONING.md` or the README of each published artifact MUST show consumers
  how to verify it.

## Consequences

- (+) Consumers can verify where every artifact was built and from which commit.
- (+) Presets and templates are pinned in the form their tools can update, and
  the pin cannot move while the release exists.
- (+) A compromised upstream release of an action or image has to stay unnoticed
  for 3 days before an update to it is even proposed.
- (+) A compromised third-party app cannot publish a release without an owner.
- (+) Scorecard makes the posture visible to consumers and shows regressions.
- (−) Ordinary fixes of third-party actions and images arrive 3 days late, and
  fixes of vulnerable actions too unless a maintainer starts them.
- (−) Flake inputs, including nixpkgs, get no delay; a compromised upstream
  commit is stopped only by CI and review.
- (−) Every run that uses the environment `release` waits for an owner.
- (−) Every repository carries a Scorecard workflow and its findings to triage.
- (±) Git-only releases have no attestation of their own; their trust rests on
  signed commits, protected tags, immutable releases and SHA references.

## Alternatives considered

- **Signing images with cosign keys.** A long-lived key to protect; keyless
  attestations bound to the workflow identity give the same assurance.
- **No minimum release age.** Updates arrive sooner, including compromised
  releases that would have been withdrawn within hours.
- **A minimum release age for flake inputs too.** Without release dates,
  Renovate either never proposes their updates or ignores the age.
- **Signed tags instead of attestations.** Tags are created by the bot through
  the API and protected by rulesets; a signature would add a key without telling
  consumers how an artifact was built.
- **Scorecard only in a few repositories.** Every repository is consumed
  separately, so each needs its own score.
- **Presets pinned by commit SHA.** Renovate accepts a SHA in a preset
  reference, but its `renovate-config` manager never updates one. It would need
  a `renovate.json5` with a version comment and a custom manager in the preset
  itself, and Copier would still record a tag.
- **Moving tags `vX` for presets and templates.** Fewer update pull requests,
  but the reference moves without review and has no release to verify.
- **One approval on every pull request instead of a reviewer on `release`.**
  Stops an app from merging alone, but with one maintainer it needs a bypass on
  every pull request; the environment stops the publication itself.

---
id: FY-0018
title: Secure the supply chain
status: accepted
date: 2026-10-22
supersedes: [FY-0014]
superseded_by: []
related:
  [FY-0005, FY-0007, FY-0008, FY-0009, FY-0010, FY-0011, FY-0013, FY-0016]
affects:
  [
    ".github:org/**",
    "*:.github/workflows/**",
    "*:renovate.json",
    "renovate-config:**",
    "image-*:**",
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
untrusted interpolation in actions ([FY-0008](0008-actions-and-workflows.md)),
pushes to the binary cache only from protected branches
([FY-0009](0009-ci-on-runners.md)), the release bot and its key
([FY-0010](0010-release-bot.md)), settings applied from code
([FY-0011](0011-settings-as-code.md)). What is missing is how consumers verify
what they receive, how dependencies of flakeyard are updated and how the posture
is measured.

Renovate knows release dates only for some datasources, such as GitHub Actions
and container images. Flake inputs follow branches or moving tags
([FY-0016](0016-pin-form.md)) and have no release dates: a minimum release age
either holds their updates back forever or is ignored with a warning.
Vulnerability databases do not cover flake inputs either.

This record replaces [FY-0014](0014-supply-chain.md) and limits the minimum
release age to dependencies with release dates; everything else is unchanged.

## Decision

### Sources

- Commits on `main` and `release/v*` are signed, and release tags are created
  only by the bot.
- Every dependency is pinned: flake inputs by `flake.lock`, actions by full
  commit SHA (required by organization settings), the image's packages by its
  `flake.lock`. Third-party actions must be in the allowlist of organization
  settings.
- Renovate proposes every update as a pull request that passes CI before it is
  merged. Updates of third-party dependencies with release dates are proposed
  only after a minimum release age of 3 days, so that a compromised release has
  time to be noticed and withdrawn. Flake inputs, updates of flakeyard's own
  repositories and updates that fix a published vulnerability are proposed at
  once.

### Releases

- Releases and their tags `vX.Y.Z` are immutable.
- Every published artifact, the image and any release asset, gets a build
  provenance attestation from `actions/attest-build-provenance`, and the image
  an SBOM attestation as well. Consumers verify them with
  `gh attestation verify` against the `flakeyard` owner.
- Git-only releases (atoms, actions, workflows, templates, the Renovate preset)
  are consumed by commit SHA; the SHA is what consumers verify.

### Detection

- Every repository has secret scanning with push protection, Dependabot alerts
  and private vulnerability reporting; atoms and actions have CodeQL.
- Workflows are linted by actionlint and zizmor through `atom-actions`.
- Every repository runs OpenSSF Scorecard on a schedule and publishes its
  results; findings are fixed or recorded as accepted with a reason, such as
  Code-Review while one maintainer merges their own pull requests.

## Constraints

- Every dependency MUST be pinned to an immutable reference: a locked flake
  input, a full commit SHA or a digest.
- Third-party actions MUST be on the allowlist of organization settings before
  they are used.
- Dependency updates MUST arrive as pull requests that pass CI. Updates of
  third-party dependencies whose datasource has release dates MUST wait at least
  3 days after their release. Flake inputs, updates of flakeyard's own
  repositories and updates that fix a published vulnerability MAY be proposed at
  once. The minimum release age MUST NOT be configured so that updates without
  release dates are held back indefinitely.
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
- (+) A compromised upstream release of an action or image has to stay unnoticed
  for 3 days before an update to it is even proposed.
- (+) Scorecard makes the posture visible to consumers and shows regressions.
- (−) Ordinary fixes of third-party actions and images arrive 3 days late.
- (−) Flake inputs, including nixpkgs, get no delay; a compromised upstream
  commit is stopped only by CI and review.
- (−) Every repository carries a Scorecard workflow and its findings to triage.
- (±) Git-only releases have no attestation of their own; their trust rests on
  signed commits, protected tags and SHA references.

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

---
id: FY-0006
title: Versioning and support policy
status: superseded
date: 2026-10-22
supersedes: []
superseded_by: [FY-0024]
related: [FY-0002, FY-0004, FY-0005]
affects:
  [
    ".github:VERSIONING.md",
    ".github:SECURITY.md",
    "atom-*:**",
    "action-*:**",
    "workflow-*:**",
    "image-*:**",
    "template-*:**",
    "renovate-config:**",
  ]
review_by:
---

> **Superseded by [FY-0024](0024-versioning-and-support.md).**

## Context

Every repository that consumers pin is released on its own
([FY-0002](0002-one-repository-per-concern.md)). Consumers are other
organizations: they update through automated pull requests and need to know from
the version alone whether an update can be merged without reading the code, and
how long they may stay on an old major version.

The products differ in what consumers depend on. An atom is a set of options,
tools and checks ([FY-0004](0004-atom-contract.md)); an action or a reusable
workflow is a set of inputs, outputs and permissions; the image is a user, an
entrypoint and settings ([FY-0005](0005-dev-container-image.md)); a template is
a set of questions and the files it generates; the Renovate preset is a set of
update rules. A version scheme is useful only if each kind states what its
compatibility covers.

Atoms and the image run tools whose versions flakeyard does not control: atoms
take every tool from the consumer's nixpkgs. A new warning or a new finding
after a nixpkgs update cannot be prevented by an atom release.

flakeyard is maintained by a small team, so it can support only a few release
lines at once.

## Decision

### Versions

Every repository that consumers pin is versioned with
[semantic versioning](https://semver.org/) and released from `main`. Versions of
different repositories are independent. Releases start at `v1.0.0`; earlier
`0.y.z` releases are previews without compatibility guarantees or support.
Repositories that consumers do not pin, `.github` and `decisions`, have no
releases.

Each release is tagged `vX.Y.Z`; these tags and their releases are immutable.
The tag `vX` points to the latest release of major version `X`.

### Public API

A breaking change of the public API requires a new major version. The public API
of each kind of repository is:

- **Atoms**: as listed in [FY-0004](0004-atom-contract.md).
- **Actions and reusable workflows**: inputs, outputs, secrets, required
  permissions and the events and runners they support.
- **Dev container image**: as listed in [FY-0005](0005-dev-container-image.md).
- **Templates**: questions and their defaults, the layout of generated projects,
  the major versions of other flakeyard repositories they reference, and the
  minimum Copier version.
- **Renovate preset**: preset names, and rules that change which updates are
  proposed, grouped or merged automatically.

Versions of tools taken from nixpkgs are not part of the public API, except for
the Nix implementation of the image.

### Changes

- **Patch**: fixes that keep the public API.
- **Minor**: backwards-compatible additions and deprecations. A deprecated
  feature keeps working and warns where the consumer sees it: an evaluation
  warning for atoms, a workflow annotation for actions and workflows, a message
  on standard error for the image, a message after `copier update` for
  templates. A new check, hook or rule that could fail a passing project ships
  disabled and is enabled in the next major version.
- **Major**: removals and incompatible changes. A feature is removed only after
  it was deprecated in a minor release of the previous major version. Every
  major release comes with a migration guide in its release notes.

### Support

Fixes are released as a new patch of the latest minor release of the current
major version; older minor releases are not patched. The previous major version
receives security fixes for 6 months after the first release of the next major
version. Older versions receive nothing.

Policy documents in `flakeyard/.github`, `VERSIONING.md` and `SECURITY.md`,
state these rules for consumers and link here.

## Constraints

- Every repository that consumers pin MUST use semantic versioning with tags
  `vX.Y.Z` and a moving tag `vX`. `.github` and `decisions` MUST NOT be
  released.
- Tags `vX.Y.Z` and their releases MUST NOT be changed or deleted after
  publication.
- The first release with compatibility guarantees MUST be `v1.0.0`. Releases
  `0.y.z` MUST be marked as pre-releases.
- Every kind of repository MUST have its public API listed in an accepted `FY`
  ADR and in `VERSIONING.md`. A change to that list MUST be made in the ADR
  first.
- A breaking change of the public API MUST be released as a new major version,
  and the major release MUST include a migration guide.
- A feature MUST be deprecated in a minor release, with a warning in the channel
  of its kind, before a major release removes it.
- A check, hook or rule that could fail a project that passed before MUST ship
  disabled in a minor release.
- Fixes MUST be released only on the latest minor of the current major and, for
  security fixes, of the previous major for 6 months after the next major is
  first released.

## Consequences

- (+) The version tells consumers whether an update is safe to merge
  automatically.
- (+) Each kind of repository has an explicit list of what its compatibility
  covers, and reviewers check changes against it.
- (+) Consumers have at least 6 months to migrate from a major version and a
  warning before anything is removed.
- (−) Strict rules make minor changes slower: new checks wait for a major
  release before they are enabled.
- (−) A nixpkgs update in a project can add warnings or findings regardless of
  atom versions.
- (±) Maintaining the previous major means up to two release lines per
  repository for 6 months.

## Alternatives considered

- **Calendar versioning.** Says when a release was made, not whether it is
  compatible, so automated updates could not be merged on the version alone.
- **One version for the whole product.** Couples unrelated concerns again;
  rejected in [FY-0002](0002-one-repository-per-concern.md).
- **Tool versions as public API.** Atoms would have to pin nixpkgs, which
  contradicts taking tools from the consumer's nixpkgs.
- **Long-term support for older majors or patches for older minors.** More
  release lines than a small team can test and maintain.
- **Support only for the latest release.** Consumers would have to migrate on
  the day of every major release.

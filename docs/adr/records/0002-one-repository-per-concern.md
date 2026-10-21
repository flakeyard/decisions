---
id: FY-0002
title: Split the product into one repository per concern
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0001]
affects:
  ["atom-*:**", "action-*:**", "workflow-*:**", "image-*:**", "template-*:**"]
review_by:
---

## Context

The predecessors of flakeyard were two Copier templates, `templates-base` and
`templates-lang-cpp`. Each template carried everything a project needed: Nix
modules, composite actions, CI workflows, linter configurations, profiling and
coverage scripts. All of it was copied into every generated project.

This had three effects:

- A fix in shared logic reached a project only through `copier update`, with a
  three-way merge of files the project might have touched.
- Unrelated concerns were released together: a change to profiling scripts
  produced a new version of the whole template.
- Consumers could not take one part, such as the linters, without the rest.

A design review proposed the opposite extreme: a single repository with one
flake exporting every module, shared CI and the container image, released with
one version. It is simpler to maintain for one person, but it couples every
concern again and makes every consumer update everything at once.

## Decision

The product is split into small repositories, each responsible for one concern
and released on its own. A concern is something a consumer can adopt, update or
leave out independently: a group of linters, a toolchain, a reusable workflow,
the dev container image.

Consumers pin each repository separately. Updates arrive per repository, so a
change to one concern produces a change only for the projects that use it.

Templates do not copy shared logic into projects. They compose the repositories:
the generated project references them and keeps only the files that must
physically exist in it.

How the repositories compose, how they are named and how they are pinned are
recorded in separate decisions.

## Constraints

- A repository MUST have one concern. A change that touches two concerns MUST be
  split between their repositories.
- Every repository that consumers pin MUST be versioned and released
  independently of the others.
- A repository MUST NOT require another repository of the same kind to be
  present unless that dependency is part of its documented interface.
- Shared logic MUST NOT be copied into generated projects. Templates MUST
  reference the repository that provides it.
- Combinations of repositories MUST be tested where they are composed, in the CI
  of the templates.

## Consequences

- (+) Consumers adopt and update concerns independently, and an update carries
  only the change of one concern.
- (+) A breaking change in one concern does not force a major release of the
  others.
- (+) Generated projects contain fewer files and no copies of shared logic.
- (−) Many repositories to maintain: each needs CI, releases and settings. This
  is mitigated by organization settings as code and a template for flakeyard's
  own repositories.
- (−) A change that spans several concerns needs coordinated pull requests.
- (±) Integration is tested in template CI rather than inside each repository,
  so a combination can break after an independent release and is caught by the
  next scheduled template run.

## Alternatives considered

- **Keep self-contained templates** (status quo). Simple to start, but shared
  logic is copied into every project and updates depend on Copier merges.
- **One repository with one flake** exporting all modules, CI and the image,
  with a single version. One lock entry and one update stream for consumers, but
  every concern is coupled again: a change anywhere is an update everywhere, and
  consumers cannot pick parts.
- **One repository with several sub-flakes** (`?dir=` inputs). Combines the
  costs of both: separate lock entries and update streams per sub-flake, but one
  repository, one history and one release process for all of them.

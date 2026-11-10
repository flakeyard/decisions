---
id: FY-0001
title: Record architecture decisions of the organization
status: accepted
date: 2026-10-21
supersedes: []
superseded_by: []
related: [FY-0024, FY-0035]
affects: ["*:docs/adr/**"]
review_by:
---

## Context

flakeyard publishes building blocks that other organizations consume: Nix flake
atoms, GitHub Actions, reusable workflows, a dev container image and Copier
templates. They live in many small repositories, and most significant decisions
span several of them: how atoms compose, how consumers pin versions, what the
templates generate, how releases are trusted.

A decision recorded in one repository is invisible from the others, and
consumers need one place that explains why the products work together the way
they do. Decisions about a single product still belong next to its code.

## Decision

Organization-wide decisions are recorded as ADRs in the `flakeyard/decisions`
repository, under `docs/adr/`, with the ids `FY-NNNN`.

A decision is organization-wide when it binds more than one repository or the
organization itself: organization settings, trust, versioning, naming,
conventions, and the contracts of families of repositories, such as `atom-*` or
`image-*`. The design of a single product, including its public API, is recorded
in that product's repository, under `docs/adr/`, with the ids `ADR-NNNN` and the
format of the records here. The versioning rules of
[FY-0024](0024-versioning-and-support.md) apply to that API.

An ADR is proposed by a pull request that adds it with the status `proposed`. It
is accepted by changing the status to `accepted` in the same or a later pull
request. Rejected ADRs are merged too, with the status `rejected`, so that the
question is not reopened without new arguments. A decision is changed only by a
new ADR that supersedes the old one, or that amends it: the new ADR states in
its own Decision and Constraints what it adds to the old one or which exception
it makes, links it in `related`, and the rest of the old one stays in force
([FY-0035](0035-extract-logic-that-has-two-users.md)).

Policy documents for consumers, such as `VERSIONING.md` in `flakeyard/.github`,
state the rules briefly and link to the ADR that explains them.

## Constraints

- Decisions that bind more than one repository or the organization itself MUST
  be recorded here as `FY-NNNN`.
- The design of a single product, including its public API, MUST be recorded in
  that product's repository as `ADR-NNNN`.
- Repository ADRs MUST NOT contradict an accepted `FY` ADR. A repository that
  needs to deviate MUST first get the `FY` ADR superseded.
- Constraints of accepted `FY` ADRs MUST be followed by every repository of the
  organization.
- References to `FY` ADRs from other repositories MUST use the full id
  (`FY-NNNN`) and a link to the record.
- ADRs MUST be changed through pull requests. An accepted ADR MUST NOT be edited
  in substance; it is superseded or amended by a new ADR instead. An amending
  ADR MUST state each amendment in its Decision and Constraints and MUST list
  the amended ADR in `related`. Typos, links and `affects` may be corrected.
  Until every flakeyard product has released `v1.0.0`, a very small change of
  substance MAY be made in place; larger changes MUST supersede the ADR.
- `affects` MUST name the governed code: path globs for this repository and
  `repository-glob:path-glob` for other repositories of the organization.
- Records and policy documents MUST be written in English.

## Consequences

- (+) One place explains to consumers and contributors how the products work
  together.
- (+) Cross-repository rules are binding and can be checked against code.
- (−) A change that spans repositories needs a pull request here first.
- (±) The design of each product, including its public API, is explained in its
  own repository.
- (±) Policy documents duplicate the essence of some ADRs and must be kept in
  sync with them.

## Alternatives considered

- **ADRs in `flakeyard/.github`.** Mixes decisions with community health files
  and organization settings; the repository has a different purpose and review
  flow.
- **ADRs only in the repositories they concern.** Cross-repository decisions
  have no natural home, and consumers would have to search many repositories.
- **The same `ADR-NNNN` ids everywhere.** `ADR-0003` would be ambiguous between
  this repository and any other one.

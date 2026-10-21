---
id: FY-0003
title: Name repositories by delivery mechanism
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0002]
affects: [".github:org/**", "renovate-config:**", "template-*:template/**"]
review_by:
---

## Context

[FY-0002](0002-one-repository-per-concern.md) splits the product into many small
repositories. Consumers and automation need to tell from a name how a repository
is used: organization settings, topics and CodeQL setup are applied by name
globs, Renovate rules group updates by name, and `affects` globs of these
records select repositories by name.

Grouping by purpose ("product" versus "service") does not work: the dev
container image, the Renovate preset and reusable workflows look like
infrastructure, but generated projects reference them directly, so they are part
of the product. What differs between repositories is how a consumer receives
them, and that also decides how they are pinned and updated.

## Decision

The prefix of a repository name names its delivery mechanism:

- `atom-`: a flake-parts module, consumed as a flake input, pinned by commit in
  `flake.nix`.
- `action-`: a composite action, consumed by `uses:` in a workflow step, pinned
  by commit SHA.
- `workflow-`: one reusable workflow, consumed by `uses:` in a workflow job,
  pinned by commit SHA.
- `image-`: an OCI image, consumed by `image:` in `devcontainer.json`, pinned by
  digest.
- `template-`: a Copier template, consumed by `copier copy` and `copier update`,
  pinned by release tag.
- `renovate-config`: Renovate presets, consumed by `extends` in `renovate.json`,
  pinned by release tag.
- No prefix: `.github` and `decisions`, which projects do not consume.

The part after the prefix names the concern, not a tool: `atom-shell`, not
`atom-shellcheck`. `renovate-config` keeps the name Renovate expects for an
organization preset repository, so presets resolve as
`github>flakeyard/renovate-config`.

Every reusable workflow gets its own `workflow-` repository, so that its inputs,
secrets and permissions are versioned independently of other workflows.

## Constraints

- A repository consumed by projects MUST carry the prefix of the mechanism it is
  consumed through, from the list above.
- A repository MUST deliver its product through exactly one mechanism. Other
  outputs MAY exist only for its own development and tests.
- A `workflow-` repository MUST contain exactly one reusable workflow.
- The name after the prefix MUST be lowercase, hyphen-separated and MUST name a
  concern rather than a tool.
- Repositories without a prefix MUST NOT be referenced by generated projects.
- Automation (organization settings, topics, Renovate rules, `affects` globs)
  MAY rely on prefixes and MUST NOT rely on any other part of a name.

## Consequences

- (+) The name tells consumers and automation how a repository is used, pinned
  and updated.
- (+) Settings and rules are applied per kind of repository with one glob.
- (−) Renaming a repository that changes its mechanism is a breaking change for
  every consumer; the old repository has to stay or redirect.
- (±) More `workflow-` repositories than a single shared one, in exchange for
  independent versions.

## Alternatives considered

- **Prefixes by purpose** (product versus service). The image, the Renovate
  preset and the workflows are consumed directly, so the split was wrong for
  exactly the repositories it was meant to describe.
- **No prefixes.** Automation would need a hand-maintained list of repositories,
  and consumers could not tell a module from an action.
- **One repository for all reusable workflows.** One version for all of them: a
  breaking change in one workflow would raise the major version for callers of
  the others, and Renovate would update every reference at once.

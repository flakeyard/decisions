---
id: FY-0027
title: Publish the dev container image
status: accepted
date: 2026-10-29
supersedes: [FY-0025]
superseded_by: []
related:
  [FY-0005, FY-0009, FY-0013, FY-0018, FY-0020, FY-0021, FY-0024, FY-0026]
affects: ["image-devcontainer:**"]
review_by:
---

## Context

The dev container image is published to `ghcr.io/flakeyard/image-devcontainer`
as one manifest list for `linux/amd64` and `linux/arm64`, with immutable tags
`vX.Y.Z`, a moving tag `vX` and provenance attestations; consumers reference it
by digest ([FY-0005](0005-dev-container-image.md)). It needs an SBOM with the
license of every package ([FY-0013](0013-licenses.md),
[FY-0018](0018-supply-chain.md)), and its publication runs in the workflow run
of the release ([FY-0020](0020-actions-and-workflows.md)).

Probes of publishing to the GitHub Container Registry (GHCR) established:

- GHCR does not enforce immutable tags: anyone who can push to the package can
  point a tag at another manifest. It deletes versions, not tags; a platform
  manifest referenced by the manifest list appears as an untagged version.
- GHCR has no referrers API. `actions/attest` stores attestations under tags
  `sha256-<digest of the subject>`, one per attested digest, and adds to them on
  every run.
- A package pushed with `GITHUB_TOKEN` from a public repository is public and
  linked to it when the image names the repository in
  `org.opencontainers.image.source`.
- Platform manifests can be pushed by digest without a tag. Two tools that
  compress layers differently produce different digests from the same archive.
- A manifest list written as canonical JSON is byte-for-byte reproducible and
  keeps its digest when pushed by digest. The tool that creates and tags a list
  in one call leaves no point to check the tag before it moves.
- An unknown tag is reported as `MANIFEST_UNKNOWN`; other failures, such as
  missing access, are reported differently.
- The notes and the title of an immutable release can still be edited
  ([FY-0024](0024-versioning-and-support.md) allows adding references to
  published artifacts).
- `gh attestation verify` succeeds only for an attestation of the given workflow
  and digest.
- sbomnix finds the license of a package only through the nixpkgs metadata of a
  flake; its matching by name and version is a heuristic. Evaluating nixpkgs
  with `allowUnfree = false` fails on any unfree license.

The image is built by `dockerTools.streamLayeredImage`, which yields a script;
the image archive exists only when the script runs. A label or a timestamp that
differs between commits gives every commit its own digest.

## Decision

### What is published

Each architecture is built natively on its runner, tested, and pushed by digest
without a tag. The same tool computes a platform digest and pushes it. A
manifest list of both is then written as canonical JSON, pushed by its digest
and tagged. The digest of the image is the digest of the manifest list; it is
what consumers pin. The digests of the platform manifests are internal but
attested.

The configuration of the image contains nothing that changes from commit to
commit by itself: no commit hash, no version from the workflow and the creation
date `1970-01-01`. Version, commit, source, license and description are
annotations of the manifest list and part of the attestations. The same commit
therefore yields the same platform digests.

Every platform manifest gets a provenance attestation and an SBOM attestation,
and the manifest list gets a provenance attestation. The SBOM is generated with
`sbomnix` from the runtime closure of the image and lists the license of every
package. Every pull request checks on both architectures that:

- nixpkgs, evaluated with `allowUnfree = false`, builds the image;
- every component of the SBOM has a license and none of them forbids
  redistribution, by the license list of nixpkgs;
- the store paths of the image are exactly the closure the SBOM describes.

### How it is published

Publication runs after the release job in the same workflow run, in three steps:

1. For each architecture, a job without write permissions builds the image,
   checks its size and runs the full smoke tests on the release commit. It
   outputs the platform digest.
2. A job with `packages: write` and the attestation permissions builds the image
   again, fails if the digest differs from the tested one, pushes it and attests
   it.
3. A job writes the manifest list, sets `vX.Y.Z` and `vX`, attests the list,
   adds the reference `ghcr.io/flakeyard/image-devcontainer:vX.Y.Z@<digest>` to
   the notes of the release, and pulls the list again on both architectures to
   verify the digests, a short smoke test and the attestations.

The state of `vX.Y.Z` is read before the list is pushed: absent only when the
registry reports `MANIFEST_UNKNOWN`, otherwise the digest it points at. The tag
is set when absent or already at the list's digest; another digest, and any
other error, fail the run. `vX` points at the manifest list of the newest
release of major version `X` whose `vX.Y.Z` exists in the registry, and never
moves back. It can lag the git tag `vX` until a publication succeeds.

An attestation is created only when `gh attestation verify` fails for that
digest and the release workflow. Publication runs in one concurrency group per
repository that queues runs and never cancels them. Reruns complete what a
failed run left undone and change nothing after a complete run. A release whose
image could not be published within the 30 days in which GitHub allows reruns is
followed by a new patch release.

Release and publication jobs substitute only from `cache.nixos.org`: the image
consists of nixpkgs paths, and no other cache can influence a published image.

### Trust

Publication uses `GITHUB_TOKEN` with `packages: write`, without an environment
([FY-0021](0021-release-bot.md) keeps the environment `release` for the bot's
key). Anyone with write access to the repository can therefore push to the
package; the immutability of tags is a property of the publication workflow, not
of the registry. Consumers rely on the digest they pin and on verifying its
attestations against the release workflow of the repository, which the README
shows with `gh attestation verify`. The README explains which digest to pin and
where release notes name it.

Nothing deletes versions of the package. Untagged versions and the tags
`sha256-*` of attestations are never deleted.

### Reproducibility

The repository checks that the same commit yields the same image:

- pull requests rebuild with `--rebuild` every derivation of the image that is
  not in `cache.nixos.org`, and stream the image twice with the same result;
- the publication compares the tested and the pushed digest;
- a weekly job rebuilds the newest release on both architectures and compares
  the digests with the registry.

## Constraints

### Content and digests

- The configuration of the image MUST NOT contain values that differ between
  commits without a change of the image, such as the commit or the time of the
  build. Its creation date MUST be `1970-01-01`.
- The digest of the image MUST be the digest of the manifest list. The notes of
  every release MUST name it.
- Every platform manifest MUST have a provenance and an SBOM attestation, and
  the manifest list a provenance attestation.
- The SBOM MUST be generated with `sbomnix` from the runtime closure and list
  the license of every package.
- Every pull request MUST check on both architectures that nixpkgs with
  `allowUnfree = false` builds the image, that every SBOM component has a
  license that allows redistribution, and that the image contains exactly the
  closure of the SBOM.

### Publication

- Platform manifests MUST be pushed by digest without tags, by the tool that
  computed the digest; temporary tags MUST NOT be used.
- The image MUST pass the full smoke tests on the release commit before it is
  pushed, and the pushed digest MUST equal the tested one.
- Jobs that run the tests of the image MUST NOT have write permissions or
  `id-token: write`.
- The manifest list MUST be pushed by the digest of its canonical JSON before
  any tag points at it.
- `vX.Y.Z` MUST be treated as absent only on `MANIFEST_UNKNOWN`, and MUST be set
  only when absent or already at the same digest. `vX` MUST point at the newest
  `vX.Y.Z` of major version `X` that exists in the registry and MUST NOT move
  back.
- An attestation MUST NOT be created when one from the release workflow already
  verifies for the same digest.
- Publication MUST run in a concurrency group that queues runs and never cancels
  them, MUST be idempotent, and MUST verify the pulled manifest list and its
  attestations.
- Release and publication jobs MUST substitute only from `cache.nixos.org`.
- Publication MUST NOT use the environment `release`.

### Registry

- Versions of the package MUST NOT be deleted. Untagged versions and the tags
  `sha256-*` MUST NOT be deleted.
- The README MUST explain which digest to pin, where release notes name it, and
  how to verify the image with `gh attestation verify` against the release
  workflow.

### Rebuilds

- Pull requests MUST rebuild with `--rebuild` the derivations of the image that
  are not in `cache.nixos.org` and compare two streams of the image.
- A weekly job MUST rebuild the newest release and compare its digests with the
  registry.

## Consequences

- (+) What consumers pull is what CI tested, and a rebuild shows it.
- (+) Tags cannot move silently through the publication workflow, and every
  digest can be verified against its source.
- (+) Reruns neither move tags nor pile up attestations.
- (−) Anyone with write access can still push to the package outside the
  workflow; consumers who skip verification are not protected from it.
- (−) A release with a broken image cannot be repaired under its tag; it needs a
  new patch release.
- (−) Every release builds each architecture three times: tested, pushed and
  pulled back.
- (±) The package lists one `sha256-*` tag per attested digest.
- (±) Container tools show the creation date `1970-01-01`.

## Alternatives considered

- **Tags per architecture, such as `vX.Y.Z-amd64`.** Simple, but adds mutable
  tags that consumers could pin and that cleanup would have to understand.
- **Temporary tags for platform manifests.** GHCR cannot remove a tag without
  deleting the manifest the list refers to.
- **Creating and tagging the manifest list in one call.** Leaves no point to
  refuse moving an existing `vX.Y.Z`.
- **The commit and the build date in the image configuration.** Easier to read
  in container tools, but every commit would get new digests, and the tested
  pull request could never have the published digest.
- **A license allowlist of SPDX identifiers.** Fails on every new license until
  the list is edited, and inherits the heuristic of the SBOM.
- **Publication in the environment `release`.** Would restrict pushes to
  protected branches, but FY-0021 keeps that environment for the bot's key, and
  GHCR would still accept pushes from any other job.
- **A separate workflow triggered by the release tag.** Tags created with
  `GITHUB_TOKEN` start no workflows, and FY-0020 requires publication in the
  release run.

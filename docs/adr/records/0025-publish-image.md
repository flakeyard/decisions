---
id: FY-0025
title: Publish the dev container image
status: superseded
date: 2026-10-28
supersedes: []
superseded_by: [FY-0027]
related: [FY-0005, FY-0009, FY-0013, FY-0018, FY-0020, FY-0021]
affects: ["image-devcontainer:**"]
review_by:
---

> **Superseded by [FY-0027](0027-publish-image.md).**

## Context

The dev container image is published to `ghcr.io/flakeyard/image-devcontainer`
as one manifest list for `linux/amd64` and `linux/arm64`, with immutable tags
`vX.Y.Z`, a moving tag `vX` and provenance attestations; consumers reference it
by digest ([FY-0005](0005-dev-container-image.md)). It needs an SBOM with the
license of every package ([FY-0013](0013-licenses.md),
[FY-0018](0018-supply-chain.md)), and its publication runs in the workflow run
of the release ([FY-0020](0020-actions-and-workflows.md)).

The GitHub Container Registry (GHCR) shapes how this can be done:

- It does not enforce immutable tags: anyone who can push to the package can
  point a tag at another manifest.
- It deletes versions, not tags. A platform manifest referenced by the manifest
  list, and an attestation stored next to an image, appear as untagged versions;
  deleting untagged versions breaks tagged images.
- Pushing with `GITHUB_TOKEN` needs only `packages: write` in the job. The
  environment `release` that protects the release bot's key may be used only by
  jobs that release, move tags or open automated pull requests
  ([FY-0021](0021-release-bot.md)).
- The visibility of a new package and its link to the repository can be changed
  only in the web interface, after the first push.

The image is built by `dockerTools.streamLayeredImage`, which yields a script;
the image archive exists only when the script runs. A label or a timestamp that
differs between commits gives every commit its own digest.

## Decision

### What is published

Each architecture is built natively on its runner, tested, and pushed by digest
without a tag. A manifest list of both is then pushed and tagged. The digest of
the image is the digest of the manifest list; it is what consumers pin and what
release notes and the README name. The digests of the platform manifests are
internal but attested.

The configuration of the image contains nothing that changes from commit to
commit by itself: no commit hash, no version from the workflow and the creation
date `1970-01-01`. Version and commit are annotations of the manifest list and
part of the attestations. The same commit therefore yields the same digests, and
a commit that does not change the image does not change its platform digests.

Every platform manifest gets a provenance attestation and an SBOM attestation,
and the manifest list gets a provenance attestation. The SBOM is generated with
`sbomnix` from the runtime closure of the image and lists the license of every
package. A check that runs in every pull request fails when a package in the
closure has a license that forbids redistribution.

### How it is published

Publication runs after the release job in the same workflow run, in three steps:

1. For each architecture, a job without write permissions builds the image,
   checks its size and runs the full smoke tests on the release commit. It
   outputs the platform digest.
2. A job with `packages: write` and the attestation permissions builds the image
   again, fails if the digest differs from the tested one, pushes it and attests
   it.
3. A job creates the manifest list, sets `vX.Y.Z` and `vX`, attests the list,
   and pulls it again on both architectures to verify the digests, a short smoke
   test and the attestations.

`vX.Y.Z` is set only when it does not exist or already points at the same
digest; another digest fails the run. `vX` is derived at the time it is moved
from the newest published release of major version `X`, as the release workflow
derives the git tag, and points at that release's manifest list. Publication
runs in one concurrency group per repository and never cancels a run. Reruns
complete what a failed run left undone and change nothing after a complete run.
A release whose image could not be published within the 30 days in which GitHub
allows reruns is followed by a new patch release.

Release and publication jobs substitute only from `cache.nixos.org`: the image
consists of nixpkgs paths, and no other cache can influence a published image.

### Trust

Publication uses `GITHUB_TOKEN` with `packages: write`, without an environment.
Anyone with write access to the repository can therefore push to the package;
the immutability of tags is a property of the publication workflow, not of the
registry. Consumers rely on the digest they pin and on verifying its
attestations against the release workflow of the repository, which the README
shows with `gh attestation verify`.

Nothing deletes versions of the package automatically, and untagged versions are
never deleted by hand. The owner sets the package to public and links it to the
repository once, after the first publication.

### Reproducibility

The repository checks that the same commit yields the same image:

- pull requests rebuild with `--rebuild` the derivations the repository defines
  itself, and stream the image twice with the same result;
- the publication compares the tested and the pushed digest;
- a weekly job rebuilds the newest release on both architectures and compares
  the digests with the registry.

## Constraints

### Content and digests

- The configuration of the image MUST NOT contain values that differ between
  commits without a change of the image, such as the commit or the time of the
  build. Its creation date MUST be `1970-01-01`.
- The digest of the image MUST be the digest of the manifest list. Release notes
  and the README MUST name it.
- Every platform manifest MUST have a provenance and an SBOM attestation, and
  the manifest list a provenance attestation.
- The SBOM MUST be generated with `sbomnix` from the runtime closure and list
  the license of every package. A check in every pull request MUST fail on a
  package whose license forbids redistribution.

### Publication

- Platform manifests MUST be pushed by digest without tags; temporary tags MUST
  NOT be used.
- The image MUST pass the full smoke tests on the release commit before it is
  pushed, and the pushed digest MUST equal the tested one.
- Jobs that run the tests of the image MUST NOT have write permissions or
  `id-token: write`.
- `vX.Y.Z` MUST be set only when absent or already at the same digest. `vX` MUST
  be derived from the newest published release of major version `X` at the time
  it is moved.
- Publication MUST run in a concurrency group that does not cancel runs, MUST be
  idempotent, and MUST verify the pulled manifest list and its attestations.
- Release and publication jobs MUST substitute only from `cache.nixos.org`.
- Publication MUST NOT use the environment `release`.

### Registry

- Versions of the package MUST NOT be deleted automatically, and untagged
  versions MUST NOT be deleted.
- The README MUST show how to verify the image with `gh attestation verify`
  against the release workflow.

### Rebuilds

- Pull requests MUST rebuild the repository's own derivations with `--rebuild`
  and compare two streams of the image.
- A weekly job MUST rebuild the newest release and compare its digests with the
  registry.

## Consequences

- (+) What consumers pull is what CI tested, and a rebuild shows it.
- (+) Tags cannot move silently through the publication workflow, and every
  digest can be verified against its source.
- (−) Anyone with write access can still push to the package outside the
  workflow; consumers who skip verification are not protected from it.
- (−) A release with a broken image cannot be repaired under its tag; it needs a
  new patch release.
- (−) Every release builds each architecture three times: tested, pushed and
  pulled back.
- (±) The creation date `1970-01-01` is shown by container tools and registries.

## Alternatives considered

- **Tags per architecture, such as `vX.Y.Z-amd64`.** Simple, but adds mutable
  tags that consumers could pin and that cleanup would have to understand.
- **Temporary tags for platform manifests.** GHCR cannot remove a tag without
  deleting the manifest the list refers to.
- **The commit and the build date in the image configuration.** Easier to read
  in container tools, but every commit would get new digests, and the tested
  pull request could never have the published digest.
- **Publication in the environment `release`.** Would restrict pushes to
  protected branches, but [FY-0021](0021-release-bot.md) keeps that environment
  for the bot's key, and GHCR would still accept pushes from any other job.
- **A separate workflow triggered by the release tag.** Tags created with
  `GITHUB_TOKEN` start no workflows, and FY-0020 requires publication in the
  release run.

---
id: FY-0032
title: Publish images behind an approval
status: accepted
date: 2026-11-03
supersedes: [FY-0027]
superseded_by: []
related:
  [
    FY-0005,
    FY-0009,
    FY-0011,
    FY-0013,
    FY-0024,
    FY-0036,
    FY-0028,
    FY-0029,
    FY-0030,
  ]
affects: ["image-*:**", ".github:org/repository/environments/**"]
review_by:
---

## Context

The dev container image is published to `ghcr.io/flakeyard/image-devcontainer`
as one manifest list for `linux/amd64` and `linux/arm64`, with immutable tags
`vX.Y.Z`, a moving tag `vX` and provenance attestations; consumers reference it
by digest ([image-devcontainer ADR-0001][image-devcontainer-adr-0001]). It needs
an SBOM with the license of every package ([FY-0013](0013-licenses.md),
[FY-0028](0028-supply-chain.md)), and its publication runs in the workflow run
of the release commit ([FY-0030](0030-actions-workflows-and-release-bot.md)).

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
  and digest. From gh 2.101 on, `--source-digest` also requires the commit the
  attestation names, which is the commit of the workflow run that created it.
- sbomnix finds the license of a package only through the nixpkgs metadata of a
  flake; its matching by name and version is a heuristic. Evaluating nixpkgs
  with `allowUnfree = false` fails on any unfree license.

The image is built by `dockerTools.streamLayeredImage`, which yields a script;
the image archive exists only when the script runs. A label or a timestamp that
differs between commits gives every commit its own digest.

[FY-0027](0027-publish-image.md) published the image with `GITHUB_TOKEN` and
`packages: write`, without an environment, and told consumers to verify
attestations against the release workflow. A design review then showed that this
check proves only that some run of `release.yaml` in the repository built the
image. Whoever can merge to `main`, such as the Renovate app of Mend with write
access to every repository ([FY-0029](0029-renovate-mend.md)), can merge a
changed workflow whose push run pushes, attests and tags an image, and that
image verifies like a release. For actions and workflows the same attack stops
at the tag `v*`, which only the gated key of `flakeyard-bot` can create
([FY-0030](0030-actions-workflows-and-release-bot.md)). The image was the only
product that reached consumers without the owner's approval.

This record replaces FY-0027. What changes is the environment of the publishing
jobs, the order of the steps of the manifest list, and how consumers verify the
image; everything else is unchanged.

## Decision

### What is published

Each architecture is built natively on its runner, tested, and pushed by digest
without a tag. The same tool computes a platform digest and pushes it. A
manifest list of both is then written as canonical JSON, pushed by its digest,
attested and tagged. The digest of the image is the digest of the manifest list;
it is what consumers pin. The digests of the platform manifests are internal but
attested.

The configuration of the image contains nothing that changes from commit to
commit by itself: no commit hash, no version from the workflow and the creation
date `1970-01-01`. Version, commit, source, license and description are
annotations of the manifest list. The same commit therefore yields the same
platform digests. A platform digest that a release leaves unchanged keeps the
attestations of the release that first published it, so platform attestations
are verified against the digest and the release workflow, not against a commit.
The manifest list names the commit and changes with every release, so its
attestation is bound to the commit of the release.

Every platform manifest gets a provenance attestation and an SBOM attestation,
and the manifest list gets a provenance attestation. The SBOM is generated with
`sbomnix` from the runtime closure of the image and lists the license of every
package. Every pull request checks on both architectures that:

- nixpkgs, evaluated with `allowUnfree = false`, builds the image;
- every component of the SBOM has a license and none of them forbids
  redistribution, by the license list of nixpkgs;
- the store paths of the image are exactly the closure the SBOM describes.

### The environment `publish`

Every `image-*` repository has a GitHub environment `publish`. It has the same
required reviewers as `release`, the list in
`org/repository/environments/release-reviewers.json` of `.github`, admins cannot
bypass it, and it can be deployed only from `main` and `release/v*`. It holds no
secrets: publication keeps using `GITHUB_TOKEN` with `packages: write`,
`id-token: write` and `attestations: write`. Its data is
`org/repository/environments/publish.json` in `.github`, applied by `apply.sh`
like the other environments ([FY-0011](0011-settings-as-code.md)).

Every job that writes to the registry runs in `publish`: the jobs that push the
platform manifests and attest them, and the job that publishes the manifest
list. The environment `release` stays as
[FY-0030](0030-actions-workflows-and-release-bot.md) defines it, for the job
that holds the key of `flakeyard-bot`; publication does not use it, because its
jobs check out and run the code of the repository.

The owner approves `publish` in the same way as `release`: in a push run of the
release workflow, or a rerun of it, at a commit that merges a release pull
request whose release the owner approved.

### How it is published

Publication runs after the release job in the same workflow run, in three steps:

1. For each architecture, a job without write permissions and without an
   environment builds the image, checks its size and runs the full smoke tests
   on the release commit. It outputs the platform digest.
2. For each architecture, a job in `publish` with `packages: write` and the
   attestation permissions builds the image again, fails if the digest differs
   from the tested one, pushes it and attests it.
3. A job in `publish` pushes the manifest list by its digest, attests it, sets
   `vX.Y.Z`, moves `vX` and adds the reference
   `ghcr.io/flakeyard/image-devcontainer:vX.Y.Z@<digest>` to the notes of the
   release, in this order. Jobs without write permissions and without an
   environment then pull the list again on both architectures to verify the
   digests, a short smoke test and the attestations.

No tag points at a manifest list before it has its provenance attestation.

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

The environment `publish` is a procedural gate. The reviewed release workflow
cannot write to the registry without the owner's approval of that run. It does
not stop a changed workflow: whoever can merge a workflow that drops
`environment:` can still push to the package with `GITHUB_TOKEN`, and the
immutability of GHCR tags remains a property of the publication workflow, not of
the registry.

What binds an image to a release is its verification. Consumers take the digest
from the release notes, not from a GHCR tag, and verify the provenance of the
manifest list against the release workflow and the commit of the git tag
`vX.Y.Z`:

```sh
gh attestation verify "oci://ghcr.io/flakeyard/<repo>@sha256:<digest>" \
  --repo flakeyard/<repo> \
  --signer-workflow flakeyard/<repo>/.github/workflows/release.yaml \
  --source-digest <commit of the git tag vX.Y.Z>
```

Only `flakeyard-bot`, after the owner's approval, can create that git tag, and
only a run at that commit can create an attestation that names it. A changed
workflow merged later runs at another commit, so its attestations fail this
check. The README and `scripts/publish/verify.sh` use this check; the platform
manifests are verified against the release workflow only. GHCR tags are a
convenience.

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

### Environment

- Every `image-*` repository MUST have the environment `publish`, defined in
  `org/repository/environments/publish.json` and applied by `apply.sh`, with the
  required reviewers of `release`, without admin bypass and deployable only from
  `main` and `release/v*`.
- `publish` MUST NOT hold secrets.
- Every job that pushes to the registry, attests, sets or moves a tag, or edits
  the notes of a release MUST run in `publish`. Jobs that build and test the
  image before the push, or verify it after, MUST NOT use an environment.
- Publication MUST NOT use the environment `release`.

### Publication

- Platform manifests MUST be pushed by digest without tags, by the tool that
  computed the digest; temporary tags MUST NOT be used.
- The image MUST pass the full smoke tests on the release commit before it is
  pushed, and the pushed digest MUST equal the tested one.
- Jobs that run the tests of the image MUST NOT have write permissions or
  `id-token: write`.
- The manifest list MUST be pushed by the digest of its canonical JSON and
  attested before any tag points at it. The steps MUST run in the order: push
  the list, attest it, set `vX.Y.Z`, move `vX`, name the image in the release
  notes.
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

### Verification

- The README and `scripts/publish/verify.sh` MUST verify the manifest list with
  `gh attestation verify` against the release workflow and with
  `--source-digest` set to the commit of the git tag `vX.Y.Z`.
- The README MUST explain which digest to pin, that the release notes name it,
  and that GHCR tags are a convenience, not the source of truth.

### Registry

- Versions of the package MUST NOT be deleted. Untagged versions and the tags
  `sha256-*` MUST NOT be deleted.

### Rebuilds

- Pull requests MUST rebuild with `--rebuild` the derivations of the image that
  are not in `cache.nixos.org` and compare two streams of the image.
- A weekly job MUST rebuild the newest release and compare its digests with the
  registry.

## Consequences

- (+) What consumers pull is what CI tested, and a rebuild shows it.
- (+) The release workflow publishes nothing without the owner's approval of the
  run.
- (+) A consumer who verifies the manifest list with `--source-digest` accepts
  only an image built at the commit of a gated git tag. A workflow merged
  without the owner cannot produce one.
- (+) Tags cannot move silently through the publication workflow, and no tag
  points at a list without provenance.
- (+) Reruns neither move tags nor pile up attestations.
- (−) The environment is only a procedural gate. A changed workflow that drops
  it can still push to the package and move GHCR tags; consumers who follow a
  GHCR tag, such as Renovate updates of `vX@sha256:`, or who skip the check of
  the commit, are not protected from it.
- (−) A release of the image asks for three approvals: the release job, the
  platform pushes and the manifest list.
- (−) If release-please creates the tag in the run of another push, such as when
  two merges race, the attestation of the list names that push's commit, and
  verification fails. It never passes falsely; the release is replaced by a new
  patch release.
- (−) Platform attestations are not bound to a commit; only the manifest list
  is.
- (−) A release with a broken image cannot be repaired under its tag; it needs a
  new patch release.
- (−) Every release builds each architecture three times: tested, pushed and
  pulled back.
- (±) The package lists one `sha256-*` tag per attested digest.
- (±) Container tools show the creation date `1970-01-01`.

## Alternatives considered

- **Publication without an environment, as in FY-0027.** One approval less, but
  the reviewed workflow publishes on any merge, and nothing marks a publication
  as approved.
- **Publication in the environment `release`.** One environment, but the key of
  `flakeyard-bot` would be visible to jobs that check out and run the code of
  the repository, which FY-0030 forbids.
- **The commit from `--format json` compared by a script.** Works with older gh
  versions, but repeats a check that `--source-digest` makes.
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
- **A separate workflow triggered by the release tag.** Tags created with
  `GITHUB_TOKEN` start no workflows, and FY-0030 requires publication in the run
  of the release commit.

[image-devcontainer-adr-0001]:
  https://github.com/flakeyard/image-devcontainer/blob/main/docs/adr/records/0001-dev-container-image.md

---
id: FY-0029
title: Run Renovate as the Mend-hosted app
status: accepted
date: 2026-11-02
supersedes: []
superseded_by: []
related:
  [FY-0003, FY-0011, FY-0012, FY-0016, FY-0022, FY-0024, FY-0028, FY-0030]
affects: [".github:org/**", "renovate-config:**", "*:renovate.json"]
review_by:
---

## Context

Dependency updates of every flakeyard repository and of every generated project
arrive as Renovate pull requests that follow the rules of the presets in
`renovate-config` ([FY-0003](0003-repository-naming.md),
[FY-0028](0028-supply-chain.md)). Something has to run Renovate. Renovate can
run as the app that Mend hosts for public repositories, or as self-hosted
Renovate in GitHub Actions with a token of its own.

Mend's documentation and the design of the presets established:

- The app is installed on the organization. Installed on all repositories, it
  runs them in Silent mode, which only looks up updates and opens nothing. A
  repository or the whole organization is switched to Interactive mode in the
  Mend Developer Portal, which has no API. The organization settings of the
  portal for Renovate also decide whether it opens pull requests ("Automated
  PRs"), whether it runs in a repository without a Renovate configuration
  ("Require config file") and whether it opens onboarding pull requests ("Create
  onboarding PRs"), for every repository of the organization.
- In Interactive mode, a repository without a Renovate configuration gets an
  onboarding pull request with Mend's default configuration, unless the
  configuration says otherwise.
- The app applies `org-inherited-config.json` from the default branch of
  `renovate-config` to every installed repository, before the repository's own
  `renovate.json`. That file is not pinned and not released.
- Mend chooses the Renovate version and updates it by hand, about once a week,
  holding back new major versions. Renovate's defaults change between versions.
  The presets are tested with the Renovate of their `flake.lock`
  ([FY-0012](0012-develop-in-container.md)), which can differ from Mend's.
- The app has write access to code, workflows, pull requests, issues, checks and
  commit statuses of every repository it is installed on, and read access to
  administration, packages, vulnerability alerts and members.
- Active repositories run every 4 hours; onboarded repositories without merged
  Renovate pull requests run once a day.
- For a flake input, Mend writes `flake.lock` with a Nix of its own build
  environment, whose version the configuration can constrain. Whether those lock
  files are accepted unchanged by flakeyard's Nix is checked by a probe before
  the first repository adopts the presets.

## Decision

flakeyard runs Renovate only as the app hosted by Mend, installed on the
organization for all repositories. flakeyard runs no Renovate that opens pull
requests itself; CI of `renovate-config` runs Renovate only to look up updates
in its tests.

### Mode and onboarding

The organization settings of Renovate in the Mend Developer Portal are the
primary mechanism. The owner set them by hand:

- Silent mode: off, so the organization is in Interactive mode;
- Automated PRs: on;
- Require config file: on, so a repository without `renovate.json` gets nothing;
- Create onboarding PRs: off, so no repository gets Mend's default
  configuration.

`renovate-config` holds the same settings in code: its
`org-inherited-config.json` contains only `onboarding: false` and
`requireConfig: "required"`. The file is reviewed like any change and keeps the
behaviour if the portal settings are reset; both say the same, so it does not
matter which one Mend applies first. It contains no update rules: what is
updated, when and how is decided only by the presets a repository pins in its
own `renovate.json`. CI of `renovate-config` allows only the onboarding keys it
lists in that file.

Every repository gets `renovate.json` through an ordinary pull request that
extends the presets by the tag of a release ([FY-0028](0028-supply-chain.md)).
Renovate opens pull requests in it only after that.

If Mend ever ignored both, the organization would go back to Silent mode and
each repository would be switched to Interactive by hand after its
`renovate.json` is merged; `org/README.md` would list that step for every new
repository.

### Installation check

`org/organization/installations.json` in `.github` lists every GitHub App that
must be installed on the organization: `renovate` and `flakeyard-bot`, each on
all repositories, with the exact permissions it may have. The permissions of
`renovate` are those the app asks for:

- read and write: contents, workflows, pull requests, issues, checks and commit
  statuses;
- read: administration, metadata, packages and vulnerability alerts;
- organization, read: members.

A permission the app asks for later is accepted only after this list and
`installations.json` name it ([FY-0028](0028-supply-chain.md)). `settings.sh`
compares `installations.json` with the installations of the organization in a
separate step after every other setting is applied, and fails when an App is
missing, suspended, not installed on all repositories or has other permissions.
A permission change accepted in the web interface therefore shows up as drift.
Installing the app stays under "Not covered" in `org/README.md`
([FY-0011](0011-settings-as-code.md)), marked as checked by `settings.sh`.

### Release environment

The app can write to every repository, so the environment `release` of every
released repository requires the approval of the organization owner
([FY-0028](0028-supply-chain.md)). `apply.sh` sets the reviewer. Only the job
that creates a release or moves tags uses the environment and waits for that
approval; runs that only update the release pull request do not
([FY-0030](0030-actions-workflows-and-release-bot.md)).

### Renovate version

The presets cannot pin the Renovate that runs them. They set explicitly every
rule that an `FY` ADR requires and extend no built-in Renovate preset, such as
`config:recommended`. Their public API is the rules they set explicitly
([FY-0024](0024-versioning-and-support.md)); a change of Renovate's defaults in
a new Mend version is not a change of the presets. The README of
`renovate-config` names the Renovate version its tests use and the version Mend
last ran, from the job log. The update pull request that `renovate-config`
receives for its own release is the first place a new release meets Mend's
version.

### Lock files

Pull requests that change `flake.lock` pass the CI of their repository, which
evaluates flakes with `--no-update-lock-file` ([FY-0016](0016-pin-form.md)),
before they are merged. A lock file written by Mend's Nix that flakeyard's Nix
rejects fails that CI instead of reaching `main`. No repository with flake
inputs adopts the presets before a probe shows that Mend's lock files pass it.
Whether the presets constrain the Nix that Mend uses is decided with that probe.

## Constraints

- Dependency update pull requests MUST come from the Mend-hosted Renovate app.
  flakeyard MUST NOT run Renovate that opens pull requests or pushes branches.
- The app MUST be installed on all repositories of the organization.
- `org/organization/installations.json` MUST list every App installed on the
  organization with its repository selection and exact permissions, and
  `settings.sh` MUST fail when the installations differ. That check MUST run
  after every other setting is applied.
- `org-inherited-config.json`, when it exists, MUST contain only the onboarding
  keys that CI of `renovate-config` allows. It MUST NOT contain `extends`,
  `packageRules`, schedules or any other rule that changes which updates are
  proposed, how or when.
- Renovate MUST NOT open pull requests in a repository before its own
  `renovate.json`, extending a flakeyard preset, is merged, and MUST NOT onboard
  a repository with Mend's default configuration.
- The organization settings of Renovate in the Mend Developer Portal MUST be
  Silent mode off, Automated PRs on, Require config file on and Create
  onboarding PRs off, and MUST be listed in `org/README.md` under "Not covered".
  `org-inherited-config.json` MUST state the same onboarding settings.
- The environment `release` of every released repository MUST have the
  organization owner as its required reviewer, set by `apply.sh`.
- Presets MUST set every rule required by an `FY` ADR explicitly and MUST NOT
  extend built-in Renovate presets.

## Consequences

- (+) No Renovate to host, upgrade or pay runner minutes for, and no flakeyard
  token with access to every repository.
- (+) A new repository needs no manual step in the Mend Developer Portal: the
  organization settings and the inherited configuration apply to it.
- (+) A change of the app's installation or permissions fails `settings.sh`
  instead of passing unnoticed.
- (−) A third party holds write access to every repository; the reviewer on
  `release` keeps it from publishing, not from opening or merging pull requests.
- (−) Every release waits for the owner.
- (−) The portal settings have no API; `settings.sh` cannot check them, and a
  change there shows up only in Renovate's behaviour.
- (−) The Renovate version, the schedule of runs and the build environment of
  lock files are Mend's; a new version can change defaults without a release of
  the presets.
- (−) `post-upgrade` commands of the presets are limited to what Mend allows, so
  updates that need a script, such as the Nix of `action-setup-nix`, stay
  outside Renovate.
- (±) `org-inherited-config.json` is read from the default branch, unpinned; it
  is safe only while it holds no update rules.

## Alternatives considered

- **Self-hosted Renovate in GitHub Actions.** Its version would come from
  `flake.lock` and it could run any post-upgrade command, but it needs a token
  with write access to code, workflows and pull requests of every repository.
  `flakeyard-bot` would need permissions
  [FY-0030](0030-actions-workflows-and-release-bot.md) does not grant, or a
  second App would need a key to protect, and flakeyard would maintain the runs
  and pay their minutes.
- **The organization in Silent mode, each repository switched to Interactive by
  hand.** No inherited configuration, but a manual step without an API for every
  new repository; kept as the fallback if Mend ignores the organization
  settings.
- **Update rules in `org-inherited-config.json`.** One place for every
  repository, but the rules would apply from the default branch of
  `renovate-config` without a release and without a pin
  ([FY-0028](0028-supply-chain.md)).
- **The app on selected repositories only.** Each new repository would need to
  be added by hand, and the installation check could not require `all`.

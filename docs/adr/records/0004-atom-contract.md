---
id: FY-0004
title: Atom layout and composition contract
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0002, FY-0003]
affects: ["atom-*:**", "action-flake-lock:**", "template-*:template/flake.nix*"]
review_by:
---

## Context

Atoms ([FY-0003](0003-repository-naming.md)) are flake-parts modules that
projects import as flake inputs. Each atom adds formatters, git hooks, checks,
apps and developer tools for one concern. Several atoms build on the same
upstream modules, `treefmt-nix` and `git-hooks.nix`, and contribute to things a
project has only once: the formatter behind `nix fmt`, the set of git hooks, the
development shell.

Experiments with real upstream revisions showed:

- Two atoms that import the same upstream module from different revisions make
  evaluation fail:
  `The option perSystem.<system>.treefmt ... is already declared`. Modules are
  deduplicated only when they are the same store path.
- With `follows` on the consumer side, every atom uses the consumer's copy of
  the upstream, modules deduplicate and the lock file stays small.
- Tools always come from the consumer's nixpkgs, whatever an atom pinned for its
  own tests.
- Upstream modules set their defaults with `mkDefault`; an atom using the same
  priority conflicts with them.
- Options that accept a single value, such as the package of the treefmt hook,
  fail when two atoms set them, even to the same value.

An earlier split by kind of tool (formatters, linters, security, maintenance)
cut domains across atoms (`shfmt` in one, `shellcheck` in another) and bundled
unrelated concerns (secret scanning with SBOM generation).

## Decision

### Layout

Atoms are split by domain. An atom owns everything for its domain: formatters,
linters as git hooks, a check per domain and developer tools. The base atoms
are:

- `atom-nix`: Nix files. nixfmt; statix and deadnix.
- `atom-shell`: shell scripts. shfmt; shellcheck.
- `atom-actions`: GitHub Actions workflows. actionlint and zizmor.
- `atom-markdown`: documentation. prettier for Markdown only; markdownlint;
  lychee as an app.
- `atom-data-formats`: YAML, JSON and TOML formatters.
- `atom-text`: all files. typos; editorconfig-checker limited to charset, line
  endings, final newline and trailing whitespace.
- `atom-reuse`: licensing. reuse lint.
- `atom-secrets`: leaked secrets. gitleaks as a hook, and an app that scans the
  history of a commit range.
- `atom-commits`: commit messages. commitlint as a `commit-msg` hook, and an app
  for a commit range.
- `atom-workspace`: the glue a project has once. It builds `devShells.default`,
  installs the git hooks, enables the treefmt hook and sets global defaults such
  as `treefmt.projectRootFile` and excludes for generated files. It has no
  inputs, and other atoms do not know about it.

Updating projects from templates and generating SBOMs belong to actions, not
atoms: both only make sense together with opening a pull request or uploading a
release.

Atoms for programming languages follow the same contract and are decided
separately.

### Dependencies

An atom declares the upstream flakes whose modules it imports as its own inputs,
under canonical names (`treefmt-nix`, `git-hooks`). The consumer makes every
atom use its own copy with `follows`, written by whoever adds the atom:

```nix
atom-shell = {
  url = "github:flakeyard/atom-shell/<commit>";
  inputs.treefmt-nix.follows = "treefmt-nix";
  inputs.git-hooks.follows = "git-hooks";
};
```

Dependencies stay one-way: the project depends on atoms, atoms depend on
upstream flakes. A missing `follows` is reported in two places:

- Every atom compares each of its inputs with the consumer's input of the same
  name before importing the upstream module, and fails with the exact line to
  add. The atom only compares store paths; it imports nothing from the
  consumer's inputs:

  ```text
  atom-shell: input 'treefmt-nix' is locked twice; add to input atom-shell:
    inputs.treefmt-nix.follows = "treefmt-nix";
  ```

- `action-flake-lock` checks `flake.lock` in CI before any evaluation and
  reports duplicates the atoms cannot see, such as a second nixpkgs.

### Structure of an atom

```text
atom-<name>/
├── flake.nix        inputs and flakeModules.default = import ./nix inputs
├── nix/
│   ├── default.nix  root module: key, _file, guarded upstream imports
│   └── *.nix        the atom's modules
└── dev/
    ├── flake.nix    tests: flake-parts, systems, fixtures
    └── flake.lock
```

### Values and priorities

Atoms set upstream options with `lib.mkOverride 900`: above the `mkDefault`
defaults of upstream modules, below plain definitions of the consumer, so a
project overrides an atom without `mkForce`. Atoms take every tool package from
the consumer's `pkgs` explicitly instead of relying on the package defaults of
upstream modules.

### Checks

Every atom provides one check for its domain, `checks.<domain>`, that runs only
its hooks over only its files. Changes outside the domain keep the check cached,
and CI shows which domain failed. The aggregated `checks.pre-commit` of
`git-hooks.nix` is disabled by `atom-workspace`. Formatting is checked by
`checks.treefmt`.

### Development shell

An atom with tools for people exposes them as `devShells.flakeyard-<atom>`.
`atom-workspace` builds `devShells.default` from every `flakeyard-*` fragment. A
fragment may bring a compiler through its `stdenv` and environment variables
through `passthru.flakeyard.env`; the workspace accepts at most one compiler and
fails on conflicting variables.

### Consumers

Atoms require flake-parts and are tested with the Nix implementation shipped in
flakeyard's dev container image and CI. Other implementations may work but are
not tested.

## Constraints

### Inputs

- An atom's public flake MUST declare as inputs exactly the upstream flakes
  whose modules it imports, under their canonical names `treefmt-nix` and
  `git-hooks`. New canonical names are added only by an `FY` decision.
- An atom MUST NOT declare `nixpkgs`, `flake-parts` or a flakeyard repository as
  an input of its public flake.
- Before importing an upstream module, an atom MUST fail evaluation with the
  exact `follows` line to add when the consumer has an input of the same name
  with a different store path. It MUST NOT import anything from the consumer's
  inputs.
- Consumers MUST deduplicate every input of every atom with `follows`. Every
  atom README MUST contain an "Inputs" section with the exact block.
- Template-generated CI MUST run `action-flake-lock` before any Nix evaluation.
  The check MUST compare normalized sources (fetcher type, case-insensitive
  host, owner and repository, `dir`), MUST skip sources it cannot identify, MUST
  honour a list of intentional duplicates and MUST print a `follows` line only
  when exactly one root input matches.

### Structure

- The public flake MUST contain only `inputs` and
  `outputs = inputs: { flakeModules.default = import ./nix inputs; };`.
- `nix/default.nix` MUST set `key = toString ./default.nix;` and
  `_file = ./default.nix;` and MUST NOT declare options. The atom's inputs MUST
  reach its other files only as function arguments, never through
  `_module.args`.
- Atoms MUST NOT set `_module.args` (including `pkgs`), overlays or `systems`.
- Atom tests MUST live in `dev/flake.nix` with their own lock, MUST import the
  atom the way consumers do, MUST evaluate checks on every system the atom
  claims and MUST run on a schedule against current upstream revisions. The
  README MUST state the oldest supported upstream revisions.

### Ownership

- Every tool, git hook id and treefmt formatter MUST have exactly one owning
  atom. Hook ids and formatter names that upstream does not define MUST start
  with `flakeyard-`. Names an atom adds to `apps`, `checks` and `devShells` MUST
  start with its domain or with `flakeyard-<atom>`.
- Atoms MUST set only options of their own tools, with `lib.mkOverride 900`, and
  MUST take tool packages from the consumer's `pkgs` explicitly.
- Only `atom-workspace` or the consumer MAY set `devShells.default`, the treefmt
  hook, `treefmt.projectRootFile`, `treefmt.settings.global.*`,
  `pre-commit.check.*`, `pre-commit.settings.excludes` and
  `pre-commit.settings.default_stages`.
- Developer-only tools MUST be exposed only through
  `devShells.flakeyard-<atom>`. CI MUST NOT use `nix develop`.

### Checks and platforms

- Every atom with hooks MUST provide `checks.<domain>` over only its own files.
- Checks MUST run without network access. Work that needs the network or git
  history MUST be an app.
- Formatters MUST be available on every supported system. Hooks and checks MAY
  be enabled per platform.
- Atoms MUST NOT write into the working tree. Every file an atom reads MUST be
  listed in its README.

### Public API

- The public API of an atom, where every incompatible change is a major release,
  is: its inputs and their names; `flakeModules`; its enabled tools, hook ids
  and formatter names; values it sets whose change rewrites files or fails a
  previously passing project; the files it reads; app names, arguments and
  environment variables; output names; supported systems; the oldest supported
  upstream revisions.
- Tool versions MUST NOT be treated as an atom's API: they come from the
  consumer's nixpkgs.

## Consequences

- (+) A project takes exactly the domains it has files for, and each atom can be
  released, tested and replaced on its own.
- (+) One copy of every upstream module, one nixpkgs and a small lock file.
- (+) A missing `follows` is reported with the line that fixes it, locally and
  in CI.
- (+) Projects override any value an atom sets with a plain definition.
- (+) Per-domain checks are cached separately and name the failing domain.
- (−) Every atom costs one to two `follows` lines in every project.
- (−) Atoms depend on the option names of upstream modules without pinning them;
  upstream renames are caught only by scheduled tests.
- (−) A broken tool in the consumer's nixpkgs fails the atom's check; the fix is
  a nixpkgs update or rollback in the project.
- (±) `atom-workspace` is required for a working development shell, and it is
  the one place where glue between atoms lives.

## Alternatives considered

- **Atoms split by kind of tool** (formatters, linters, security, maintenance).
  Domains were cut across atoms and unrelated concerns bundled.
- **Atoms import upstream modules from the consumer's inputs.** No `follows`
  needed, but atoms would depend on the input names of the project: a two-way
  dependency.
- **Atoms as data plus one adapter module.** Atoms isolated from upstream option
  names, but a schema of our own to design and maintain, and atoms limited to
  what the schema can express.
- **`follows` generated by templates.** Flake inputs cannot be computed, so this
  needs a table of every atom's inputs in each template and splits `flake.nix`
  between template and project.
- **A shared options-only atom for the development shell.** One more `follows`
  per atom and a dependency between atoms; plain `devShells` fragments and
  `atom-workspace` achieve the same.
- **One aggregated `checks.pre-commit`.** Any change reruns every hook, and CI
  shows only that something failed.
- **`mkDefault` for atom values.** Conflicts with upstream defaults of the same
  priority.

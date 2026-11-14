# decisions

[![OpenSSF Scorecard][scorecard-badge]][scorecard-viewer]

Architecture decisions of the [flakeyard](https://github.com/flakeyard)
organization: the rules that bind its repositories and how its atoms, actions,
dev container image and templates work together.

- [Index of decisions](docs/adr/INDEX.md)
- [How records are organized](docs/adr/README.md)
- [FY-0001](docs/adr/records/0001-record-decisions.md): what belongs here and
  how decisions are made

## Scope

A decision is recorded here when it binds more than one repository or the
organization itself: organization settings, trust, versioning, naming,
conventions, and the contracts of families of repositories such as `atom-*` or
`image-*`. The design of a single product, including its public API, lives in
that product's repository under `docs/adr/` as `ADR-NNNN` and must not
contradict accepted `FY` decisions.

## Proposing a decision

1. Copy [the template](docs/adr/template.md) to
   `docs/adr/records/NNNN-short-title.md` with the next free number, or open an
   issue if you only want to raise the question.
2. Open a pull request with the status `proposed`. Discussion happens in the
   pull request.
3. When the decision is made, the status changes to `accepted` or `rejected` and
   `docs/adr/INDEX.md` is regenerated.

## Development

Open the repository in its dev container,
[`.devcontainer/devcontainer.json`](.devcontainer/devcontainer.json), with
"Reopen in Container" in VS Code or `devcontainer up --workspace-folder .`.
`direnv` loads the development shell of [`flake.nix`](flake.nix) through
[`.envrc`](.envrc) ([FY-0012][fy-0012]): [atom-workspace][atom-workspace] joins
the tools of the atoms, and installs the git hooks each time the shell starts.
The container runs `direnv allow` when it is created; run it once in every other
worktree, and run `direnv block` before you check out a branch you do not trust,
since entering the directory runs that branch's shell hook. Commit inside the
container, whose Nix store has the tools of the hooks. The checks of [CI](#ci)
and [Bootstrap](#bootstrap) run in it as in CI; tools that the shell does not
have come from nixpkgs, such as `nix run nixpkgs#prettier`. Nix on the host
works as well: `direnv allow` or `nix develop` gives the same shell.

## CI

[`test.yaml`](.github/workflows/test.yaml) calls
[workflow-check][workflow-check], which checks `flake.lock`, pull request
titles, `renovate.json` and the immutable release behind its preset tag
([FY-0034][fy-0034]), and builds the checks of the root
[`flake.nix`](flake.nix), which only imports atoms ([FY-0038][fy-0038]):
[atom-workflows][atom-workflows] lints the workflows with actionlint and zizmor
(`checks.workflows`). Run them with `nix flake check`. The weekly run of
`test.yaml` also checks the end of life of the nixpkgs branch in `flake.lock`
([FY-0036][fy-0036]). [`scorecard.yaml`](.github/workflows/scorecard.yaml) runs
OpenSSF Scorecard through [workflow-scorecard][workflow-scorecard].

## Bootstrap

This repository is bootstrapped under
[FY-0022](docs/adr/records/0022-bootstrap.md). Until the providers of its
remaining checks are released:

- Prettier, markdownlint, typos and `reuse lint` run locally before every pull
  request (provider: the atoms).

## License

The text of the records is licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), see
[LICENSE](LICENSE). The CI workflows, the dev container configuration, the
flake, `.envrc`, `.gitignore` and `renovate.json` are code, licensed under
[MIT](LICENSES/MIT.txt) ([FY-0013](docs/adr/records/0013-licenses.md)).
Licensing of every file is declared in [REUSE.toml](REUSE.toml).

[workflow-check]: https://github.com/flakeyard/workflow-check
[workflow-scorecard]: https://github.com/flakeyard/workflow-scorecard
[atom-workflows]: https://github.com/flakeyard/atom-workflows
[fy-0034]: docs/adr/records/0034-check-every-repository-with-workflow-check.md
[fy-0036]:
  docs/adr/records/0036-check-the-end-of-life-of-nixpkgs-in-flake-lock.md
[fy-0038]: docs/adr/records/0038-roll-out-the-atoms.md
[scorecard-badge]:
  https://api.scorecard.dev/projects/github.com/flakeyard/decisions/badge
[scorecard-viewer]:
  https://scorecard.dev/viewer/?uri=github.com/flakeyard/decisions
[atom-workspace]: https://github.com/flakeyard/atom-workspace
[fy-0012]: docs/adr/records/0012-develop-in-container.md

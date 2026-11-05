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

## CI

[`test.yaml`](.github/workflows/test.yaml) calls
[workflow-check][workflow-check], which checks pull request titles,
`renovate.json` and the immutable release behind its preset tag
([FY-0031][fy-0031]). The repository has no flake, so no flake checks are built.
[`scorecard.yaml`](.github/workflows/scorecard.yaml) runs OpenSSF Scorecard
through [workflow-scorecard][workflow-scorecard].

## Bootstrap

This repository is bootstrapped under
[FY-0022](docs/adr/records/0022-bootstrap.md). Until the providers of its
remaining checks are released:

- actionlint, zizmor, Prettier, markdownlint, typos and `reuse lint` run locally
  before every pull request (provider: the atoms).

## License

The text of the records is licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), see
[LICENSE](LICENSE). The CI workflows and `renovate.json` are code, licensed
under [MIT](LICENSES/MIT.txt) ([FY-0013](docs/adr/records/0013-licenses.md)).
Licensing of every file is declared in [REUSE.toml](REUSE.toml).

[workflow-check]: https://github.com/flakeyard/workflow-check
[workflow-scorecard]: https://github.com/flakeyard/workflow-scorecard
[fy-0031]:
  docs/adr/records/0031-check-pull-requests-and-pushes-with-workflow-check.md
[scorecard-badge]:
  https://api.scorecard.dev/projects/github.com/flakeyard/decisions/badge
[scorecard-viewer]:
  https://scorecard.dev/viewer/?uri=github.com/flakeyard/decisions

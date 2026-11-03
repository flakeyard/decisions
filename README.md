# decisions

[![OpenSSF Scorecard][scorecard-badge]][scorecard-viewer]

Architecture decisions of the [flakeyard](https://github.com/flakeyard)
organization: why the atoms, actions, dev container image and templates work the
way they do.

- [Index of decisions](docs/adr/INDEX.md)
- [How records are organized](docs/adr/README.md)
- [FY-0001](docs/adr/records/0001-record-decisions.md): what belongs here and
  how decisions are made

## Scope

A decision is recorded here when it affects more than one repository or the
contract with consumers. Decisions about a single repository live in that
repository's `docs/adr/` as `ADR-NNNN` and must not contradict accepted `FY`
decisions.

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

- actionlint, zizmor, Prettier, markdownlint and typos run locally before every
  pull request (provider: the atoms);
- rulesets require no status checks until every repository has adopted
  workflow-check.

## License

The text of the records is licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), see
[LICENSE](LICENSE).

[workflow-check]: https://github.com/flakeyard/workflow-check
[workflow-scorecard]: https://github.com/flakeyard/workflow-scorecard
[fy-0031]:
  docs/adr/records/0031-check-pull-requests-and-pushes-with-workflow-check.md
[scorecard-badge]:
  https://api.scorecard.dev/projects/github.com/flakeyard/decisions/badge
[scorecard-viewer]:
  https://scorecard.dev/viewer/?uri=github.com/flakeyard/decisions

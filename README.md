# decisions

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

## Bootstrap

This repository is bootstrapped under
[FY-0022](docs/adr/records/0022-bootstrap.md). It has no CI checks yet. Until
their providers are released:

- Prettier, markdownlint and typos run locally before every pull request
  (provider: `workflow-check` and the atoms);
- pull request titles are checked by the reviewer (provider: `workflow-check`);
- rulesets require no status checks (provider: `workflow-check`);
- CI does not check that the tag of the preset in `renovate.json` belongs to an
  immutable release ([FY-0028](docs/adr/records/0028-supply-chain.md))
  (provider: `workflow-check`);
- OpenSSF Scorecard is not run (provider: `workflow-check`).

## License

The text of the records is licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), see
[LICENSE](LICENSE).

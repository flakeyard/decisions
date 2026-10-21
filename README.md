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

## License

The text of the records is licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), see
[LICENSE](LICENSE).

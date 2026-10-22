---
id: FY-0015
title: Documentation rules
status: accepted
date: 2026-10-22
supersedes: []
superseded_by: []
related: [FY-0001, FY-0004, FY-0012]
affects: ["*:**/*.md", "atom-markdown:**"]
review_by:
---

## Context

flakeyard's documentation is spread over many repositories: READMEs, policy
documents in `.github`, decisions, changelogs and comments. Consumers read it to
decide whether to adopt a product and how to update it, often from diffs in pull
requests and from terminals. Inconsistent formatting produces noisy diffs, long
lines are hard to review, and characters that look alike, such as the em dash
and the hyphen, are easy to get wrong in plain text.

Most repositories are independent of programming languages: base atoms, actions,
workflows, the image, the base template and the Renovate preset. Text about one
language in them suggests a coupling that does not exist.

## Decision

- Everything written into a repository is in English: documentation, comments,
  commit messages and pull request descriptions.
- Markdown is formatted by prettier with prose wrapped at 80 characters and
  checked by markdownlint with its default rules. Lines may be longer only when
  they cannot be broken, such as URLs and tables; long URLs are written as
  reference links.
- The em dash is written as `&mdash;` in Markdown, never as the raw character.
- Spelling is checked by typos, and links by lychee as an app, since checks run
  without network access ([FY-0004](0004-atom-contract.md)).
- `atom-markdown` and `atom-text` provide these tools; until they are released,
  the same tools are run from the host.
- Repositories independent of programming languages do not mention specific
  languages. `decisions` and `.github` may, because they describe the whole
  organization.
- Every README states what the repository delivers, how consumers use it, and
  how to develop it ([FY-0012](0012-develop-in-container.md)). Atom READMEs also
  contain the sections required by [FY-0004](0004-atom-contract.md).
- Policy documents state rules briefly and link to the ADR that explains them
  ([FY-0001](0001-record-decisions.md)).

## Constraints

- Text in repositories MUST be in English.
- Markdown MUST be formatted by prettier with `proseWrap: always` at 80
  characters and MUST pass markdownlint with default rules. Long URLs MUST be
  reference links.
- Markdown MUST NOT contain the raw em dash character; `&mdash;` MUST be used.
- Every repository MUST pass typos. Links MUST be checked by the lychee app on a
  schedule.
- Repositories other than `decisions` and `.github` that do not deliver
  language-specific products MUST NOT mention specific programming languages.
- Every README MUST describe what the repository delivers, how to use it and how
  to develop it.

## Consequences

- (+) Uniform text with small, reviewable diffs.
- (+) Formatting and spelling are checked by tools, not by reviewers.
- (−) `&mdash;` is less readable in the source than the character it renders.
- (−) Broken links are found on schedule, not in the pull request that broke
  them.
- (±) Language-specific examples belong in language-specific repositories, even
  where an example would make a generic one clearer.

## Alternatives considered

- **No line limit (`proseWrap: preserve`).** Fewer reformatting diffs, but long
  lines in reviews and terminals.
- **markdownlint with a custom configuration.** Every repository would carry and
  synchronize it; the defaults fit once prettier formats the text.
- **Raw Unicode punctuation.** Readable in rendered text, but indistinguishable
  from hyphens in many fonts and editors.
- **Link checks in every pull request.** Need network access and fail on outages
  of unrelated sites.

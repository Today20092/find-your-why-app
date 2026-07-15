# Issue tracker: GitHub

Issues and PRDs for this repository live in GitHub Issues. Use the `gh` CLI for all operations and infer the repository from `git remote -v`.

## Common operations

- Create: `gh issue create --title "..." --body "..."`
- Read: `gh issue view <number> --comments`
- List: `gh issue list --state open`
- Comment: `gh issue comment <number> --body "..."`
- Label: `gh issue edit <number> --add-label "..."`
- Close: `gh issue close <number> --comment "..."`

When a skill says to publish to the issue tracker, create a GitHub issue. When it says to fetch a ticket, use `gh issue view <number> --comments`.

## Pull requests as a triage surface

External pull requests are not a request or triage surface.

## Wayfinding

Use one issue labelled `wayfinder:map` as the map and linked sub-issues as tickets. Use GitHub issue dependencies for blocking edges when available; otherwise record `Blocked by: #<number>` in the child issue. Claim work by assigning the issue, and resolve it with a concluding comment before closing it.

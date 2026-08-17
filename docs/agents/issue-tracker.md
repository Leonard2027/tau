# Issue tracker: GitHub

Development issues and PRDs for this fork live in the GitHub Issues for
`Leonard2027/tau`. Use the `gh` CLI for all operations.

The upstream repository, `huggingface/tau`, is reference material. Its issues,
pull requests, roadmap, and history may be read to understand the original
project, but agents must not create, edit, label, comment on, or close upstream
issues or pull requests unless the user explicitly requests that operation.

## Conventions

- **Create an issue**: `gh issue create --repo Leonard2027/tau --title "..." --body-file <file>`.
- **Read a fork issue**: `gh issue view <number> --repo Leonard2027/tau --comments`, also fetching labels when needed.
- **List fork issues**: `gh issue list --repo Leonard2027/tau --state open --json number,title,body,labels,comments` with appropriate label and state filters.
- **Comment on an issue**: `gh issue comment <number> --repo Leonard2027/tau --body-file <file>`.
- **Apply or remove labels**: `gh issue edit <number> --repo Leonard2027/tau --add-label "..."` or `--remove-label "..."`.
- **Close an issue**: `gh issue close <number> --repo Leonard2027/tau --comment "..."`.
- Write multiline Markdown bodies through a temporary file or heredoc and pass them with `--body-file`; do not encode line breaks as literal `\n` sequences.
- Verify a created or edited body with `gh issue view ... --json body` when practical.

Commands that write data specify `--repo Leonard2027/tau` deliberately so a
future `upstream` remote or a different default repository cannot redirect a
write to the original project.

## Pull requests as a triage surface

**PRs as a request surface: no.**

Pull requests are not treated as incoming feature requests by default. A user
can change this flag later if the fork begins accepting external contributions.

GitHub shares one number space across issues and pull requests. If a bare
reference such as `#42` is ambiguous, resolve it against the named repository
with `gh pr view 42 --repo <owner/repo>` and fall back to
`gh issue view 42 --repo <owner/repo>`.

## When a skill says "publish to the issue tracker"

Create an issue in `Leonard2027/tau`.

## When a skill says "fetch the relevant ticket"

Read the ticket from `Leonard2027/tau` unless the user or surrounding context
explicitly identifies it as an upstream ticket. Upstream references should use
`--repo huggingface/tau` and remain read-only by default.

## Wayfinding operations

When a skill uses a map with child tickets, keep the map and all child issues in
`Leonard2027/tau`.

- **Map**: one issue labelled `wayfinder:map`, holding Notes, Decisions-so-far, and Fog.
- **Child ticket**: a GitHub sub-issue linked to the map. If sub-issues are unavailable, add it to a task list in the map and put `Part of #<map>` at the top of the child body.
- **Blocking**: prefer GitHub's native issue dependencies. If unavailable, use a `Blocked by: #<n>` line at the top of the child body.
- **Frontier query**: choose the first open, unassigned child without an open blocker.
- **Claim**: `gh issue edit <n> --repo Leonard2027/tau --add-assignee @me`.
- **Resolve**: comment with the result, close the child, and add the resulting context pointer to the map's Decisions-so-far section.

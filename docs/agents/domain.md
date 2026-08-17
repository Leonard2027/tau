# Domain Docs

How engineering skills should consume Tau's existing design and architecture
documentation when exploring or extending this fork.

## Before exploring, read these

1. **`dev-notes/README.md`** — the contributor-oriented map of the repository's build journals and design records.
2. **`dev-notes/design/`** — read the roadmap and high-level design documents relevant to the task.
3. **`dev-notes/architecture/`** — read the phase or feature notes that cover the code being changed.
4. **`dev-notes/adr/`** — read ADRs that touch the area being changed.
5. **`website/content/`** — consult the published user documentation when the task affects user-facing behavior.
6. **`CONTEXT.md`**, if it is created later — use its glossary and domain terminology.

Read only the notes relevant to the current task after using the README and
indexes for orientation. Do not load every historical note automatically.

If an optional file does not exist, proceed silently. Do not suggest creating a
new `CONTEXT.md` or ADR merely because it is absent; create one when domain
terminology or an architectural decision is actually resolved.

## File structure

Tau is treated as a single-context repository whose established documentation
layout is:

```text
/
├── dev-notes/
│   ├── README.md
│   ├── design/          high-level design and roadmap
│   ├── architecture/    phase-by-phase and feature implementation notes
│   └── adr/             architecture decision records
├── website/content/     published user-facing documentation
├── CONTEXT.md           optional domain glossary, created when needed
└── src/
```

Do not introduce a parallel `docs/adr/` hierarchy. New architecture decisions
should follow the repository's existing convention under `dev-notes/adr/`.

## Distinguish upstream history from fork decisions

The existing `dev-notes/` content documents the design and evolution inherited
from upstream `huggingface/tau`. Preserve it as historical and architectural
context.

When this fork makes a materially different decision:

- add a new ADR under `dev-notes/adr/` rather than rewriting the historical rationale;
- identify that the decision applies to the fork;
- link the upstream note or ADR it extends, supersedes, or intentionally diverges from;
- update user-facing documentation under `website/content/` when behavior changes.

## Use the glossary's vocabulary

If `CONTEXT.md` exists, use its defined terms in issue titles, proposals,
hypotheses, test names, and implementation notes. Do not drift to synonyms the
glossary explicitly avoids.

If a needed concept is missing, first check whether the project already uses a
stable term in `dev-notes/`, source code, tests, or published documentation. A
real unresolved vocabulary gap can be captured later with the domain-modeling
workflow.

## Flag ADR conflicts

If proposed work contradicts an existing ADR, surface the conflict explicitly
rather than silently overriding it. For example:

> Contradicts `dev-notes/adr/0001-use-textual-for-tui.md`; this fork would reopen the decision because ...

Record an accepted divergence as a new ADR and state whether it supersedes the
older decision for this fork.

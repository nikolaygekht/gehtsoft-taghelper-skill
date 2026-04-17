# gehtsoft-taghelper-skill — Repo Context for Claude

## What This Repo Is

This repo **packages a skill** that teaches Claude (via Claude Code or OpenAI Codex) how to install and use the Gehtsoft.TagHelpers ASP.NET Core Razor library. The repo itself is *not* a consumer of the library — its only artifact is the skill in `skills/gehtsoft-taghelpers/` and the install/eval tooling around it.

## Layout

```
.
├── README.md                       — what the repo is, who it's for
├── INSTALL.md                      — install steps for Claude Code & Codex, global & per-project
├── CLAUDE.md                       — (this file) context for Claude working ON the repo
├── CLAUDE.LOCALS.md                — local paths to library source + example projects (gitignored)
├── LICENSE
├── .gitignore                      — ignores CLAUDE.LOCALS.md and *-workspace/ eval dirs
├── skills/
│   └── gehtsoft-taghelpers/
│       ├── SKILL.md                — slim entry point (compact body, points to references/)
│       └── references/
│           ├── setup.md            — NuGet, DI, _ViewImports, _Layout, Kendo via bower + manual
│           ├── forms.md            — x-form, x-control family, validation, ajax, layouts
│           ├── grids.md            — x-grid, columns, server-datasource, transport
│           ├── common.md           — x-bundle, x-popup, x-menu, x-script, x-tabstrip, etc.
│           └── troubleshooting.md  — common errors, model-binding gotchas, ajax pitfalls
└── evals/
    └── evals.json                  — test prompts (with assertions added during iteration)
```

## Local Paths (Not in Repo)

Paths to the library source and real consumer projects — used while authoring/iterating but **not shipped** with the skill — live in `CLAUDE.LOCALS.md`. That file is gitignored to keep machine paths off GitHub. Always reference local resources through `CLAUDE.LOCALS.md`, never inline.

## Skill Author Conventions

These rules apply when **editing the skill itself** (anything under `skills/`):

1. **Examples must be generic.** Use `Customer`, `Product`, `Invoice`, `User`, `Order` for model classes. Do **not** copy domain language, model names, or controller actions from the real consumer projects listed in `CLAUDE.LOCALS.md` (Coinaccountant, FFL Web). The skill is a template for new code; project-specific naming would mislead users.

2. **The skill must be self-contained at use-time.** When someone runs the skill, the Gehtsoft.TagHelpers source tree is *not* available to them. Every fact the skill asserts must live in the skill files (SKILL.md or references/). If you find yourself wanting to say "see the source for X", instead extract X into the references.

3. **`SKILL.md` stays compact.** Target <500 lines. It's the entry point: setup checklist, tag-family map, pointers into `references/`. Long-form per-tag docs go in `references/`.

4. **Per-reference table of contents.** Each file in `references/` is large enough to deserve a TOC at the top — Claude reads them à la carte and benefits from being able to jump.

5. **Razor examples use fenced ` ```cshtml ` or ` ```html ` blocks.** C# uses ` ```csharp `. JSON uses ` ```json `. This matters for syntax highlighting in the eval viewer.

6. **Don't invent attributes.** When unsure, check the library source (paths in `CLAUDE.LOCALS.md`) before adding an attribute to the docs. Better to omit an obscure attribute than to document one that doesn't exist.

## Iteration Workflow

The skill is developed iteratively against test prompts in `evals/evals.json`. Eval workspaces live as siblings to the skill at `gehtsoft-taghelpers-workspace/iteration-N/` (gitignored via the `*-workspace/` rule). Use `~/.claude/skills/skill-creator/` infrastructure (`scripts/aggregate_benchmark.py`, `eval-viewer/generate_review.py`) — **don't reinvent the eval harness**.

## Useful Pointers

- Library source, real consumer apps, library's own `CLAUDE.md`: see `CLAUDE.LOCALS.md`.
- The library has its own `CLAUDE.md` and `CLAUDE.LOCALS.md` documenting the same patterns we're using here — that's the prior art for the locals approach.

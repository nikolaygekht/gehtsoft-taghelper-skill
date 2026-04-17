# gehtsoft-taghelpers — Skill for Claude Code & OpenAI Codex

A skill that teaches AI coding assistants (Claude Code, OpenAI Codex) how to install and use the **Gehtsoft.TagHelpers** ASP.NET Core Razor library — a Kendo-UI-backed tag helper library that lets you build server-rendered forms, grids, popups, and menus with custom `<x-*>` tags.

## What's in This Repo

| Path | Purpose |
|------|---------|
| `skills/gehtsoft-taghelpers/SKILL.md` | The skill entry point. The agent reads this when triggered. |
| `skills/gehtsoft-taghelpers/references/` | Per-topic reference docs (setup, forms, grids, common tags, troubleshooting) loaded on demand. |
| `INSTALL.md` | Step-by-step install for Claude Code and OpenAI Codex, global and per-project. |
| `evals/evals.json` | Test prompts used to iterate on skill quality. |

## What the Skill Helps With

When triggered, the skill enables Claude/Codex to:

- **Install the library into a fresh ASP.NET Core MVC project** — NuGet package, DI registration (`AddTagHelperServices`, `AddKendo2022Driver`), `_ViewImports.cshtml` directive, `_Layout.cshtml` `<x-lib-includes>` snippet.
- **Set up Kendo UI assets** under `wwwroot/lib/Kendo.UI/` via Bower (preferred) or manual file placement.
- **Write Razor views** using the `<x-form>`, `<x-control>`, `<x-grid>`, `<x-popup>`, `<x-menu>`, `<x-bundle>`, `<x-script>` tag families with correct attributes and idiomatic patterns.
- **Wire client- and server-side validation** via `<x-validate>` and `<x-form-errors>`.
- **Bind grids to AJAX data sources** via `<x-server-datasource>` / `<x-server-transport>`.

## When the Skill Triggers

The skill description targets Razor/MVC contexts that mention or imply Gehtsoft.TagHelpers — for example, when the user asks how to add `x-form` or `x-grid` to a view, when `@addTagHelper *, Gehtsoft.TagHelpers` is present in `_ViewImports.cshtml`, or when the project's `.csproj` references the `Gehtsoft.TagHelpers` package.

## Install

See [INSTALL.md](INSTALL.md). Both Claude Code and OpenAI Codex are supported, in both global (all projects) and per-project modes.

## License

See [LICENSE](LICENSE).

## Authoring & Contributing

If you're working on the skill itself — fixing wrong examples, adding tag coverage, etc. — read [CLAUDE.md](CLAUDE.md) for repo conventions and `CLAUDE.LOCALS.md` for paths to the upstream library source and reference applications.

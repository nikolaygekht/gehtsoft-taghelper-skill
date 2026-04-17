# Installation

The skill ships as a folder at `skills/gehtsoft-taghelpers/`. Installation means making that folder discoverable by your assistant — Claude Code or OpenAI Codex. You can install **globally** (available in every project) or **per-project** (only this repo). Pick the model that matches how often you reach for the library.

> Throughout this doc, `<repo>` means the absolute path to your local clone of this repository.

---

## Claude Code

Claude Code discovers skills from two locations:

| Mode | Location | Visibility |
|------|----------|------------|
| Global (user-wide) | `~/.claude/skills/<skill-name>/` | All Claude Code sessions, every project |
| Per-project | `<your-project>/.claude/skills/<skill-name>/` | Only sessions started in `<your-project>` |

In both cases, the skill is "installed" simply by ensuring `SKILL.md` lives at the right path. The recommended approach is a **symlink** from the install location to the canonical copy in this repo — that way `git pull` here updates the installed skill.

### Global install (Claude Code)

```bash
mkdir -p ~/.claude/skills
ln -s <repo>/skills/gehtsoft-taghelpers ~/.claude/skills/gehtsoft-taghelpers
```

Verify:

```bash
ls -la ~/.claude/skills/gehtsoft-taghelpers/SKILL.md
```

You should see the symlink resolve to `<repo>/skills/gehtsoft-taghelpers/SKILL.md`.

If you'd rather copy than symlink:

```bash
cp -r <repo>/skills/gehtsoft-taghelpers ~/.claude/skills/
```

(Re-copy after each `git pull`.)

### Per-project install (Claude Code)

In the ASP.NET Core project where you actually use Gehtsoft.TagHelpers:

```bash
cd <your-aspnetcore-project>
mkdir -p .claude/skills
ln -s <repo>/skills/gehtsoft-taghelpers .claude/skills/gehtsoft-taghelpers
```

Add `.claude/skills/` to that project's `.gitignore` if you don't want to commit the install (the symlink target will differ between machines). Or commit a copy and accept the duplication.

### Windows (Claude Code)

Symlinks work in PowerShell with admin or developer mode:

```powershell
New-Item -ItemType SymbolicLink `
    -Path "$env:USERPROFILE\.claude\skills\gehtsoft-taghelpers" `
    -Target "<repo>\skills\gehtsoft-taghelpers"
```

Or just copy:

```powershell
Copy-Item -Recurse "<repo>\skills\gehtsoft-taghelpers" "$env:USERPROFILE\.claude\skills\"
```

### Verify it loaded

Start a Claude Code session in a project that uses (or wants to use) Gehtsoft.TagHelpers, then ask:

> Add a Gehtsoft x-form with first/last name fields to my Index view.

Claude should announce that it's consulting the `gehtsoft-taghelpers` skill.

---

## OpenAI Codex

Codex doesn't have a native "skill" concept with auto-triggering — it reads `AGENTS.md` files. The pattern is: install the skill folder somewhere, then point an `AGENTS.md` at it so Codex loads the skill into context whenever it works in a relevant project.

| Mode | `AGENTS.md` location | When skill loads |
|------|----------------------|------------------|
| Global (user-wide) | `~/.codex/AGENTS.md` | Every Codex session |
| Per-project | `<your-project>/AGENTS.md` | Only sessions in that project |

### Step 1 — Place the skill folder

Same as Claude, but the path doesn't need to be `.claude/skills/`. A common convention is `~/.codex/skills/`:

```bash
mkdir -p ~/.codex/skills
ln -s <repo>/skills/gehtsoft-taghelpers ~/.codex/skills/gehtsoft-taghelpers
```

(Or copy, or place it anywhere — the `AGENTS.md` snippet below will reference the absolute path.)

### Step 2 — Add an AGENTS.md snippet

#### Global (Codex, all projects)

Edit (or create) `~/.codex/AGENTS.md` and append:

```markdown
## Gehtsoft.TagHelpers skill

When working in an ASP.NET Core project that uses (or is about to use)
the Gehtsoft.TagHelpers library — i.e., the `.csproj` references
`Gehtsoft.TagHelpers`, `_ViewImports.cshtml` contains
`@addTagHelper *, Gehtsoft.TagHelpers`, or any Razor view uses `<x-*>` tags —
read and follow the skill at:

    ~/.codex/skills/gehtsoft-taghelpers/SKILL.md

That skill covers installation, DI wiring, Kendo UI asset placement, and
the full `<x-form>`, `<x-grid>`, `<x-popup>`, and related tag families.
Reference files in the same directory provide deeper coverage of each
topic — load them on demand as the skill instructs.
```

#### Per-project (Codex, this project only)

In your ASP.NET Core project, edit `AGENTS.md` (create it if needed) and append the same snippet, replacing the path with wherever you placed the skill:

```markdown
## Gehtsoft.TagHelpers skill

This project uses Gehtsoft.TagHelpers. When asked to add or modify Razor
views, configure the library, or adjust Kendo UI assets, read and follow
the skill at:

    <repo>/skills/gehtsoft-taghelpers/SKILL.md
```

You can use a relative path if the repo is checked out adjacent to the project, or an absolute path otherwise.

### Verify it loaded

Start a Codex session and ask the same prompt as the Claude Code verification step. Codex should mention reading the skill instructions before producing markup.

---

## Updating the skill

If you installed via **symlink**, just pull this repo:

```bash
cd <repo>
git pull
```

The installed skill will pick up the changes automatically on the next session.

If you installed via **copy**, re-run the copy command after `git pull`.

---

## Uninstall

### Claude Code

```bash
rm ~/.claude/skills/gehtsoft-taghelpers           # global
rm <your-project>/.claude/skills/gehtsoft-taghelpers   # per-project
```

### Codex

Remove the symlink/folder, then delete the snippet from `AGENTS.md`:

```bash
rm ~/.codex/skills/gehtsoft-taghelpers
# Then edit ~/.codex/AGENTS.md and remove the "Gehtsoft.TagHelpers skill" section.
```

---

## Troubleshooting

**"The skill doesn't trigger when I expect it to."**
The skill description targets clear signals — `@addTagHelper *, Gehtsoft.TagHelpers`, `<x-form>` / `<x-grid>` in a Razor file, the package reference in `.csproj`. If your scenario is more ambient ("help me build a CRUD page"), name the library or the tag explicitly, or add a project-level hint in your project's `CLAUDE.md` / `AGENTS.md`.

**"Claude Code says it can't find the skill."**
Check the symlink target resolves: `readlink -f ~/.claude/skills/gehtsoft-taghelpers/SKILL.md`. On Windows, confirm the symlink type is `SymbolicLink` (not `HardLink` or `Junction`).

**"Codex isn't reading the skill at all."**
Confirm `AGENTS.md` is at the documented location and that the path inside the snippet is absolute. Codex doesn't search for skills — it only follows pointers from `AGENTS.md`.

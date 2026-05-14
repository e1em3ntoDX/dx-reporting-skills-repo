# DevExpress Reporting Skills

Two AI coding skills for [DevExpress XtraReports](https://docs.devexpress.com/XtraReports/2162). Install them in Claude or GitHub Copilot to get accurate, copy-paste-ready DevExpress code without hunting through documentation.

| Skill | What it covers |
|---|---|
| **dx-report-designer** | Writing and editing `*.Designer.cs` layout files — `InitializeComponent` structure, band hierarchy, controls, expression bindings, styles, parameters, `SqlDataSource` |
| **dx-report-integration** | Embedding the viewer/designer in every platform — WinForms, WPF, ASP.NET Core MVC/Razor Pages, Angular, React, Blazor; export pipelines; security; troubleshooting |

Both skills target **DevExpress v25.2**.

---

## Install in Claude Code

Claude Code reads skills from `.claude/skills/` in your project, or `~/.claude/skills/` globally.

### Install by copying the skill folders

```bash
# Project-level (only active in this repo)
mkdir -p .claude/skills
cp -r dx-report-designer-skill    .claude/skills/
cp -r dx-report-integration-skill .claude/skills/

# Global (active in all projects)
mkdir -p ~/.claude/skills
cp -r dx-report-designer-skill    ~/.claude/skills/
cp -r dx-report-integration-skill ~/.claude/skills/
```

Skills activate automatically — no commands or flags needed. Just ask Claude Code about DevExpress Reports and it picks up the relevant skill.

---

## Install in Claude (claude.ai)

Skills are uploaded once in Settings and apply to all future conversations.

1. Download the `.skill` files from the `dist/` folder:
   - [`dist/dx-report-designer-skill.skill`](./dist/dx-report-designer-skill.skill)
   - [`dist/dx-report-integration-skill.skill`](./dist/dx-report-integration-skill.skill)
   - Or download both at once: [`dist/dx-reporting-skills-bundle.zip`](./dist/dx-reporting-skills-bundle.zip), then unzip.

2. In [claude.ai](https://claude.ai), click your profile → **Settings** → **Skills**.

3. Click **Add skill** and upload one `.skill` file. Repeat for the second.

That's it. Claude picks up the skills automatically on relevant questions — no special commands needed.

---

## Install for GitHub Copilot

GitHub Copilot uses the skills as files in your repository's `.github/skills/` directory (or globally in `~/.copilot/skills/`). The skills activate automatically when Copilot agent mode handles a relevant question.

### Install by copying the skill folders

Clone or download this repo, then copy the skill folders into your project:

```bash
# From the repo root
cp -r dx-report-designer-skill    your-project/.github/skills/
cp -r dx-report-integration-skill your-project/.github/skills/
```

Create the `.github/skills/` directory if it doesn't exist. The skills activate automatically — no further configuration needed in the IDE.

For a **global install** (available across all projects):

| OS | Global path |
|---|---|
| Windows | `%USERPROFILE%\.copilot\skills\` |
| macOS / Linux | `~/.copilot/skills/` |

---

## IDE-specific setup

### VS Code

After installing the skill files, enable skill discovery:

1. Open **Settings** (`Ctrl+,` / `Cmd+,`).
2. Search for `chat.agent.skills`.
3. Enable **Chat: Use Agent Skills**.

Then use Copilot Chat in **Agent mode** (`@workspace` or the agent picker). Skills trigger automatically based on your question.

### Visual Studio 2022 / 2026

Visual Studio uses the same `.github/skills/` folder. After copying the files:

1. Open Copilot Chat (`View` → `GitHub Copilot Chat`).
2. Switch to **Agent mode** using the mode picker in the chat window.
3. Skills are discovered automatically — no further setup.

> **Note:** Custom agents (`.agent.md`) require Visual Studio 2026 v18.4+, but skills (`SKILL.md` in `.github/skills/`) work in Visual Studio 2022 17.x and later.

### JetBrains Rider

Rider supports GitHub Copilot via the [GitHub Copilot plugin](https://plugins.jetbrains.com/plugin/17718-github-copilot). After installing the plugin and signing in:

1. Copy the skill folders into `.github/skills/` in your project root (same as the manual install above).
2. Open the Copilot Chat panel (`Tools` → `GitHub Copilot` → `Open GitHub Copilot Chat`).
3. Skills are discovered from the `.github/skills/` directory automatically when agent mode is active.

### GitHub Copilot CLI

Place the skills in `~/.copilot/skills/` for global use:

```bash
mkdir -p ~/.copilot/skills
cp -r dx-report-designer-skill    ~/.copilot/skills/
cp -r dx-report-integration-skill ~/.copilot/skills/
```

Then start a Copilot CLI session and ask naturally — the skills activate automatically.

---

## Usage examples

Once installed, just ask naturally. Both skills trigger automatically; no slash commands needed.

**Designer skill** (`.Designer.cs` work):
- *"Add an XRTable with 4 columns and alternating row styles to the detail band"*
- *"Write InitializeComponent for a report with a group header that shows OrderDate"*
- *"Why does my report crash in the VS designer after I added a parameter?"*
- *"How do I add a date range parameter with RangeStartParameter?"*

**Integration skill** (viewer/export setup):
- *"Set up the Document Viewer in my ASP.NET Core MVC project — show me Program.cs and the view"*
- *"Add a Blazor DxReportViewer page and wire it to a custom IReportProvider"*
- *"Export a report to PDF from a controller action without showing a preview"*
- *"The Angular viewer shows a blank page — how do I diagnose this?"*
- *"Hide the XLS export format from the viewer toolbar"*
- *"How do I prevent users from accessing other users' reports?"*

---

## Repository layout

```
dx-reporting-skills/
├── README.md
├── dist/
│   ├── dx-report-designer-skill.skill     ← Claude install
│   ├── dx-report-integration-skill.skill  ← Claude install
│   └── dx-reporting-skills-bundle.zip     ← both skills, one download
│
├── dx-report-designer-skill/              ← skill source
│   ├── SKILL.md
│   └── references/
│       ├── designer-file-patterns.md
│       ├── expressions-and-summaries.md
│       └── data-sources-and-parameters.md
│
└── dx-report-integration-skill/           ← skill source
    ├── SKILL.md
    └── references/
        ├── platform-setup/
        │   ├── aspnet-core-mvc.md
        │   ├── aspnet-core-razor-pages.md
        │   ├── angular-react.md
        │   ├── blazor.md
        │   ├── winforms-wpf.md
        │   └── report-storage.md
        ├── examples/
        │   ├── aspnet-core-viewer-page.md
        │   └── blazor-viewer-page.md
        ├── viewer-toolbar-customization.md
        ├── custom-parameter-editors.md
        ├── export-and-parameters.md
        ├── security-and-best-practices.md
        └── troubleshooting-and-diagnostics.md
```

---

## Building `.skill` files from source

`.skill` files are standard zip archives. To rebuild:

```bash
# Using the included script (requires Python 3)
python scripts/package_skill.py dx-report-designer-skill    dist/
python scripts/package_skill.py dx-report-integration-skill dist/

# Or with plain zip
zip -r dist/dx-report-designer-skill.skill    dx-report-designer-skill/
zip -r dist/dx-report-integration-skill.skill dx-report-integration-skill/
```

---

## Contributing

The source is plain Markdown — easy to read and edit directly. PRs and issues welcome for outdated API patterns, missing platforms, or incorrect examples. Keep `SKILL.md` under ~500 lines; detailed content lives in `references/` files.

---

## License

MIT

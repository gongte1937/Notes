# Repository Guidelines

## Project Structure & Module Organization
This repository is an Obsidian vault organized by topic and workflow rather than application modules.

- Root-level notes: general references such as `README.md`, `Markdown.md`, and dated notes like `2026-02-13.md`.
- Topic folders: long-term knowledge areas such as `Tech/`, `Skills/`, `Interview/`, and `Talentblocks/`.
- Workflow folders: recurring capture and templates, including `Daily Notes/` and `obsidian templates/`.

Keep related notes in the nearest topical directory and prefer moving files over duplicating content.

## Build, Test, and Development Commands
There is no build or runtime pipeline in this repository. Use lightweight checks before committing:

- `git status` — review changed files.
- `git diff --name-only` — verify scope is intentional.
- `rg "\[\["` — quickly inspect wiki-link usage patterns.
- `rg "#" Daily\ Notes` — review tags in daily notes.

Open the vault in Obsidian to validate backlinks, graph connections, and rendered Markdown.

## Coding Style & Naming Conventions
Use clean, portable Markdown that renders well in Obsidian.

- Headings: start at `#` once per file, then descend logically.
- Lists: use `-` consistently; avoid mixed bullet styles.
- Code fences: always specify language when possible (for example, ```bash).
- Filenames: use descriptive names; keep existing language conventions (Chinese and English are both used).
- Dates: keep daily notes in `YYYY-MM-DD.md` format.

Prefer concise sections and short paragraphs. Use internal links like `[[Note Name]]` and tags like `#topic/subtopic` where useful.

## Testing Guidelines
Validation is content-focused:

- Confirm every new link resolves to an existing note or a planned stub.
- Check that moved/renamed notes do not break important backlinks.
- Preview edited notes in Obsidian to catch formatting issues.

For large reorganizations, spot-check graph and search results after changes.

## Commit & Pull Request Guidelines
Recent history favors short, imperative commit messages (for example, `update daily note`, `add hooks note`). Keep this style, but make scope explicit.

- Good pattern: `<action> <scope>` (for example, `add interview system-design notes`).
- One logical change per commit.
- PRs should include: purpose, affected folders, and any note renames/moves.
- Add screenshots only when visual layout or graph behavior is relevant.

## Security & Configuration Tips
Do not commit secrets or personal credentials. If sensitive information is needed, redact it or store it outside version control.

## Agent-Specific Instructions

When editing or creating notes in this vault, first review related materials under `Skills/` and align terminology, structure, and linking style with existing notes there when applicable.

### Formatting / Cleanup Skill
When asked to format or clean up any Markdown file, follow the checklist in `Skills/format-markdown.md` before making changes.

### Explanation Skill
When asked to explain a concept or technology, follow the structure defined in `Skills/explaination.md`: start with background (what the world looked like before it existed), describe the concrete problem it solved, explain the solution approach, and optionally end with a one-line summary. Use plain language and everyday analogies throughout.

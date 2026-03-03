# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is an Obsidian vault — a personal knowledge base managed with [Obsidian](https://obsidian.md/). There is no build, compile, or test pipeline. All content is plain Markdown.

## Folder Structure

| Folder | Purpose |
| --- | --- |
| `Daily Notes/` | Daily captures, one file per day (`YYYY-MM-DD.md`) |
| `Tech/` | Long-term technical knowledge (auth, databases, etc.) |
| `Skills/` | Skill reference docs used during editing tasks |
| `Talentblocks/` | Project-specific notes for the Talentblocks project |
| `Interview/` | Interview preparation notes |
| `obsidian templates/` | Obsidian note templates |
| `.obsidian/` | Obsidian app config — do not edit manually |

Keep related notes in the nearest topical directory. Move files rather than duplicating content.

## Useful Inspection Commands

```bash
git status                        # review changed files
git diff --name-only              # verify scope before committing
rg "\[\["                         # inspect wiki-link usage
rg "#" "Daily Notes"             # review tags in daily notes
```

Open the vault in Obsidian to validate backlinks, graph connections, and rendered Markdown.

## Markdown Conventions

- Headings: one `#` title per file, then descend logically (`##`, `###`, …)
- Lists: use `-` consistently; no mixed bullet styles
- Code fences: always specify a language (e.g., ` ```bash `)
- Obsidian wiki links: `[[Note Name]]`, `[[Note Name|display text]]`, `[[Note Name#Heading]]`
- Tags: `#topic` or `#topic/subtopic`
- Filenames: descriptive; Chinese and English both acceptable
- Daily notes: `YYYY-MM-DD.md` inside `Daily Notes/`

## Formatting / Cleanup Skill

When asked to format or clean up any Markdown file, **always follow the checklist in `Skills/format-markdown.md`** before making changes.

## Commit Style

Short, imperative messages following `<action> <scope>`:

```
add interview system-design notes
update JWT auth migration phase 2
fix broken link in Tech/Database
```

One logical change per commit. Do not commit sensitive information (passwords, tokens, credentials).

# Skill: Format Markdown Files

## When to Use

Use this skill whenever a Markdown file needs to be cleaned up and reformatted.

---

## Checklist

Work through the following steps in order:

### 1. Fix Wrapping Code Fences

- Check if the file starts with a stray ` ```markdown ` or ` ````markdown ` fence
- If the entire document is wrapped inside a code block, remove the opening and closing fences so the content renders as normal Markdown

### 2. Clean Up List Item Spacing

- Remove trailing spaces and unnecessary blank lines between list items
- If every list item is separated by a blank line, consolidate them unless each item is long enough to warrant breathing room
- Replace single-item lists with inline text or bold formatting

### 3. Unify Heading Number Style

- Top-level sections (`##`): use consistent numbering (e.g. `一、二、三` or `1. 2. 3.`)
- Sub-sections (`###`): use plain numbers — avoid mixing emoji numbers (1️⃣) with regular ones

### 4. Convert Key-Value Lists to Tables

- If a list is structured as "concept = description" pairs, convert it to a two-column table for clarity
- Align column widths with spaces to keep the source readable

### 5. Use Blockquotes for Standalone Conclusions

- Wrap important one-liner takeaways in a `>` blockquote to create visual hierarchy

### 6. Merge Fragmented Paragraphs

- If two sentences are logically connected but separated by a blank line, merge them into one paragraph
- Example: a description followed immediately by its definition can be combined into a single sentence

---

## Output Standards

- Code blocks render correctly with proper language tags (`sql`, `bash`, etc.)
- Lists are compact with no unnecessary blank lines
- Heading levels are clear and numbering style is consistent
- Key terms are bolded (`**term**`)
- Important conclusions use blockquotes (`>`)

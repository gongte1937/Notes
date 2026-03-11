# Skill: Daily Work Plan

## When to Use

Use this skill when the user asks to generate a daily work plan, start the day, or create today's note.

---

## Steps

### 1. Determine Today's Date

Use the current date in `YYYY-MM-DD` format.

### 2. Review Recent Context

Before generating the plan, gather context from:

- **Yesterday's daily note** (`Daily Notes/YYYY-MM-DD.md`): look for incomplete tasks (`- [ ]`) and carry them forward
- **Talentblocks project notes** (`Talentblocks/`): check for any open action items or in-progress features
- **Recent git log**: `git log --oneline -10` to see what was last worked on

### 3. Generate the Daily Plan

Create or update the daily note at `Daily Notes/YYYY-MM-DD.md` using the structure below.

```markdown
# YYYY-MM-DD

## Focus

One sentence: the single most important thing to accomplish today.

## Tasks

### 今日重点 (Priority)

- [ ] Task 1 — brief description of why it matters
- [ ] Task 2

### 跟进事项 (Follow-up)

- [ ] Carried-over items from yesterday or ongoing threads

### 学习 / 研究 (Learning)

- [ ] Any reading, research, or skill-building planned

## Notes

Space for thoughts, blockers, or ideas that come up during the day.
```

### 4. Populate Tasks Intelligently

- Pull incomplete `- [ ]` items from the previous day's note into **跟进事项**
- Suggest 1–3 priority tasks based on recent project context (Talentblocks, interview prep, etc.)
- Keep the list realistic — no more than 5–7 total tasks
- If today is Monday, also suggest a brief weekly goal in the **Focus** line

### 5. Output

- Write the file to `Daily Notes/YYYY-MM-DD.md`
- Confirm what was carried over from yesterday and what is new
- Remind the user to open the vault in Obsidian to review backlinks

---

## Output Standards

- Use `- [ ]` for all actionable tasks
- Headings follow the vault convention: one `#` title, then `##` sections
- Keep Chinese section labels for consistency with existing daily notes
- Do not add tasks that are not grounded in actual context from the notes or git log

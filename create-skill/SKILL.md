---
name: create-skill
description: >
  Creates AND updates Claude Code skills under /Users/luis/dev/claude-code/skills/<skill-name>/.
  For new skills, scaffolds SKILL.md, registers via symlink into ~/.claude/skills/<skill-name>,
  and generates a Spanish mirror README.es.md. For existing skills, applies edits to
  SKILL.md and keeps README.es.md in sync so both languages never drift.
  Use this skill whenever the user asks to "create a skill", "crear una skill",
  "crear un skill", "nueva skill", "add a skill", "scaffold a skill", OR to "update a
  skill", "edit a skill", "modify a skill", "change a skill", "fix a skill",
  "actualizar una skill", "editar una skill", "modificar una skill", "cambiar una skill",
  "arreglar una skill", or any similar phrase about creating, adding, modifying, or
  editing a Claude Code skill (including its description, body, triggers, or rules).
---

# Create Skill

This skill scaffolds new Claude Code skills consistently and registers them so Claude
Code can discover and invoke them.

## Required outputs

For every new skill named `<skill-name>`:

1. **Skill file (English, canonical)**: `/Users/luis/dev/claude-code/skills/<skill-name>/SKILL.md`
   - Must have YAML frontmatter with `name` and `description` fields.
   - `name` must match the folder name exactly.
   - `description` must be specific and enumerate trigger phrases (including Spanish
     equivalents when relevant) so Claude routes to the skill reliably.
   - Body explains what the skill does, when to use it, required inputs, and any
     conventions or step-by-step instructions Claude should follow.

2. **Spanish mirror**: `/Users/luis/dev/claude-code/skills/<skill-name>/README.es.md`
   - A faithful Spanish translation of SKILL.md (body). Do **not** include YAML
     frontmatter — this file is human documentation, not a registered skill.
   - Start with a short note clarifying that it's a Spanish translation for human
     reading, not operational context for agents.
   - Purpose: human-readable reference for the user. Claude Code discovers skills by
     the exact filename `SKILL.md`, so `README.es.md` is never loaded as a skill.
   - The `README.es.md` name (instead of `SKILL.es.md`) prevents any agent from
     confusing it with a skill file during file listings.

3. **Registration symlink**: `~/.claude/skills/<skill-name>` →
   `/Users/luis/dev/claude-code/skills/<skill-name>`
   - Use `ln -s` with absolute paths.
   - Verify with `ls -la ~/.claude/skills/` afterwards.

## Workflow

1. Ask the user for (or infer from the request):
   - `<skill-name>` (kebab-case, matches folder).
   - One-sentence purpose.
   - Trigger phrases the user expects (English + Spanish if applicable).
2. Draft SKILL.md with a rich `description` (specific triggers, not vague) and a clear
   body. Avoid generic descriptions like "helps with X" — be explicit about *when* to
   trigger.
3. Translate the body to README.es.md (no frontmatter). Keep headings parallel so diffs
   are easy to spot. Add a one-line note at the top marking it as a Spanish translation
   for human reading.
4. Create the symlink in `~/.claude/skills/`.
5. Confirm success to the user with the three artifact paths and remind them the skill
   is available immediately (Claude Code picks up skills from `~/.claude/skills/` on the
   next prompt; no restart needed).

## Updating an existing skill

Whenever the user asks to change, edit, or update a skill (its description, body,
triggers, rules, or any content), you MUST apply the same change to both files so they
stay in sync:

1. Update `SKILL.md` (canonical English version).
2. Apply the equivalent change to `README.es.md`, translated to Spanish, keeping
   headings and structure parallel. `README.es.md` has no frontmatter.
3. If only one file exists for an older skill, create the missing mirror before
   finishing the edit.
4. Confirm to the user that both files were updated.

This rule applies even if the user only mentions one language — the Spanish mirror
must never drift from the English canonical version.

## Conventions

- Always work inside `/Users/luis/dev/claude-code` (never parent/sibling paths).
- Skill folder name = YAML `name` field. No spaces, kebab-case.
- Keep SKILL.md focused: what the skill does, when to trigger, required inputs, rules.
- Do NOT edit settings.json for skills — filesystem discovery handles it via the
  symlink.
- If the skill needs supporting files (templates, references), place them in a
  `references/` subfolder inside the skill directory.

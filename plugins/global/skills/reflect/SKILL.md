---
description: >
  Review the current conversation to extract learnings, patterns, and conventions worth
  preserving for future sessions. Use this skill when the user says things like "/reflect",
  "/retro", "what did we learn", "are there any learnings worth recording", "let's capture
  what we learned", or any request to distill conversation insights into persistent knowledge.
  Also consider suggesting this skill at the end of long or complex sessions where significant
  problem-solving occurred.
---

# Reflect

Review the current conversation, identify what was learned, and write it to the right place
so future sessions benefit. This is how the project's AI knowledge base improves over time.

## Mindset

Not every conversation produces learnings. A session where everything went smoothly and
nothing surprising happened is a perfectly valid outcome — just say "nothing worth recording
this time" and move on. The goal is quality over quantity: a few precise, well-placed insights
are worth more than a wall of notes that bloat context and go stale.

Think of yourself as a thoughtful colleague writing notes for your future self (and other
developers). Would this note actually help someone? Would it prevent a real mistake? If not,
skip it.

## Scope: project-level knowledge only

All learnings must be written to **project-scoped files** that are checked into the repo and
shared with all developers. The valid destinations are:

- `CLAUDE.md` (project root)
- `docs/` (project docs, AI guides)
- `.claude/skills/` (project skills)

**Do not write to personal/global memory** (e.g., `~/.claude/projects/.../memory/`). That
directory is per-user and not shared. When checking whether something is "already documented,"
only check project-scoped files — a learning that exists only in personal memory is not
documented from the project's perspective and may be worth capturing in project docs.

If you find useful knowledge in personal memory that isn't in project docs, consider proposing
to migrate it.

## Step 1: Review the conversation

Scan the full conversation for:

- **Mistakes and corrections** — What went wrong? What was the fix? Would the mistake recur
  without a note?
- **Non-obvious patterns** — Conventions or approaches that aren't documented but should be.
  Things that required trial-and-error or user correction to get right.
- **Architectural decisions** — Why was something done a particular way? Context that would
  be lost without recording it.
- **Tool/workflow discoveries** — Commands, sequences, or approaches that worked well
  (or didn't).
- **Outdated information** — Did anything in the conversation contradict existing docs,
  CLAUDE.md, or AI guides? Flag these for cleanup.

Ignore things that are:

- Already documented **in project-scoped files**
- Too specific to this one task to be useful again
- Obviously temporary (one-off workarounds, debugging steps)

## Step 2: Categorize and propose

Present your findings as a quick debrief. For each learning, state:

1. **What** — The insight, concisely
2. **Where** — Which destination (see below)
3. **Why** — Why it's worth recording (one sentence)

Also list anything you think should be **updated or removed** from existing docs/CLAUDE.md,
with reasoning.

If there's nothing worth recording, say so. Don't manufacture learnings.

### Write destinations

Choose the destination based on who benefits and how often it's needed:

**CLAUDE.md** — Rules that apply to virtually every session. The bar is high: if an AI agent
working on this project would need this knowledge regardless of what task they're doing, it
belongs here. Examples: "always run check-errors after changes," "connect signals in code."
Keep entries concise. CLAUDE.md is always loaded, so every line costs context window.

**Project docs** (`docs/specs/` for design specs, `docs/conventions/` for patterns) —
Knowledge useful for any developer, human or AI. Architecture explanations, design rationale,
domain concepts. Write these in a natural style for a general audience. See `docs/README.md`
for the full structure.

**AI guides** (`docs/ai-guides/`) — Knowledge specifically about how an AI agent should
operate in this codebase. Procedural instructions, common pitfalls when generating code,
patterns that an AI tends to get wrong without guidance. Written in imperative style,
optimized for an AI to follow. These are read on-demand when relevant, not every session.

**Skills** (`.claude/skills/`) — Repeatable multi-step workflows that keep coming up. This
is rare — only suggest a new skill if you've seen the same complex workflow needed multiple
times or if the user explicitly asks.

### Organizing doc files

For project docs and AI guides, think about topic grouping:

- Add to an existing file if the learning fits an established topic
- Create a new file if it's a genuinely new topic area
- Use descriptive filenames (`ui-patterns.md`, `godot-resources.md`, not `notes.md`)
- Keep individual files focused — if a file is growing past ~100 lines, consider splitting

## Step 3: Confirm with the user

Wait for explicit approval before writing anything. Present your proposals clearly enough
that the user can:

- Approve all
- Approve some and reject others
- Suggest rewording or a different destination
- Say "nothing worth keeping" and end the retro

## Step 4: Write the learnings

After approval, make the writes. For each destination:

- **CLAUDE.md**: Add concise entries under an appropriate existing section, or create a new
  section if needed. Avoid duplicating information that's in the docs — just add a pointer
  if needed (e.g., "See docs/ai-guides/scene-creation.md for detailed patterns").
- **Docs**: Create or update files. Include enough context that the learning makes sense
  in isolation — someone reading the doc shouldn't need to know about this conversation.
- **AI guides**: Write in imperative style. Be specific and actionable.
- **Cleanup**: Remove or update outdated entries. Show the user what you're removing and why.

## Step 5: Verify

After writing, do a quick sanity check:

- Read back CLAUDE.md to confirm it's not getting bloated
- Make sure new docs are discoverable (linked from CLAUDE.md or an index if appropriate)
- Run `tools/check-errors` if you modified any code-adjacent files

Give the user a brief summary of what was written and where.

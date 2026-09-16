---
name: storage-guidelines
description: >
  Use when deciding where any file/directory the bot creates should live.
  Covers the bot workspace rule (creates→/bot-workspace/, throwaway→/tmp/),
  the three bot-storage zones, the skills storage locations (profile-local and
  shared/default), what belongs where, what never goes where, the bot-workspace
  layout, and a decision flow for every file the bot outputs.
  That keeps the machine clean, keeps keepsakes easy to back up, and keeps
  throwaway work out of /home/zombie/ so a reformat never requires manual cleanup.
---

# Storage guidelines — where bot output lives and why

## Rule of thumb

- If it's worth keeping, it goes in `/bot-workspace/`.
- If it's disposable, one-shot, or won't matter after a reboot, it goes in `/tmp/`.
- Bot-authored **work product** does not scatter into `/home/zombie/` outside those two. Ever.

Everything below exists so a bot has a single, repeatable rule for every file it touches, so nothing gets parked in a random home-directory corner and so a reformat never leaves behind a trail of my own files.

---

## The three bot storage zones

### 1. Bot workspace — `/bot-workspace/`

**What it is:** the home for everything the bot creates that has value and should survive a reboot.

**What goes here:**
- output files the user asked for
- configs the bot authored for the user
- project files, notes, plans, databases, schemas, seeds, source for generated pages
- anything that would be a pain to lose and that the user would want backed up

**What does NOT go here:**
- throwaway temp files (those go to `/tmp/`)
- the bot's own framework state (Hermes profile internals, systemd units, plugin dirs the platform owns — those live where the platform put them)

**Current layout in use:**
```
/bot-workspace/
├── albums/                 # bot-authored project folders (e.g. test album output)
├── USER.md                 # bot-authored user profile / research notes
├── skills/                 # keepsake copies of bot-authored skills (for inventory / backup)
│   └── storage-guidelines.md
└── zombie/
    └── Bot-Database/       # bot database project dir(s)
        └── database-plans.md
```

Subfolders under `/bot-workspace/` should be created as the work requires. There is no fixed tree beyond what the work needs; the rule is just "kept things go here."

### 2. Temporary scratch — `/tmp/`

**What it is:** the one-shot scratch area. Contents are ephemeral by design.

**What goes here:**
- one-off verification scripts (e.g. a song verifier run once on one track)
- temp extraction dirs for a zip install that's already been imported
- scratch code a bot runs once to answer a question and doesn't need to keep
- any intermediate artifact that exists only to produce a final result and isn't the final result itself

**Why `/tmp/`:**
- `/tmp` is a tmpfs on this machine, so it's RAM-backed.
- Contents vanish on reboot.
- systemd-tmpfiles ages out `/tmp` entries older than 10 days.
- That means temp work self-cleans without the bot having to remember to delete it, and a reboot is a guaranteed scrub.

**What does NOT go here:**
- anything the user would want to keep
- anything that is itself a deliverable
- anything that another step depends on after the current turn

If there's doubt, assume keep and put it in `/bot-workspace/`. Only use `/tmp/` when the bot is confident the file is disposable.

### 3. Platform internals — leave alone

Some dirs are owned by the Hermes platform or by other tools, not by the bot's output. The bot may read them, and it may edit existing files in them when a task calls for it (for example, editing a config that already lives there), but it does not treat them as a place to park its own new files.

Examples (these are platform state, not bot work product):
- `~/.hermes/` — Hermes profile, skills the platform loads from, memories, config
- `~/.hermes/plugins/` — plugin dirs
- `/etc/systemd/user/` or `~/.config/systemd/user/` — systemd user units
- `~/.config/hypr/` — Hyprland config (bot may edit in place, e.g. a window rule, but doesn't create its own new unrelated files there)
- `~/.unipet/` — UniPet runtime state (the bot can import pets into it, and can edit config in place, but it's runtime state, not bot output storage)
- `/home/zombie/apps/UniPet/` — the UniPet install (bot may edit overlay files in place; that's modification of an existing install, not parking new bot work product there)

The operative distinction:
- **Editing a file that already lives in one of these dirs** — OK when the task requires it. That's modification of existing state.
- **Creating a brand-new bot-authored file and putting it there** — not the bot's output home. New keepsake files go to `/bot-workspace/`. New disposable files go to `/tmp/`.

---

## Where skills live

Skills are the one case where a keepsake file legitimately lives both in the skills dirs (so the platform and all agents can load it) and in the bot workspace (as the recorded keepsake / backup copy). Don't lose track of the distinction: the skills dirs are the **loading** location; the bot workspace copy is the **inventory / backup** location.

### Profile-local skills — `~/.hermes/profiles/<profile>/skills/`

- Each Hermes profile has its own skills dir under `~/.hermes/profiles/<profile-name>/skills/`.
- For the active profile this session that is: `~/.hermes/profiles/aria/skills/`
- A skill installed for this profile lives at `~/.hermes/profiles/aria/skills/<skill-name>/SKILL.md`
- This is where the **live, loadable** copy for this profile lives.

### Shared / default skills — `~/.hermes/skills/`

- `~/.hermes/skills/` is the shared skills store at the Hermes root — the place all profiles / agents can see skills from.
- A skill placed here lives at `~/.hermes/skills/<skill-name>/SKILL.md`
- Installing a skill into `~/.hermes/skills/` makes it available to all agents that load from the default skills location.
- This is the "default skills" store — put a skill here when you want it seen by more than just one profile.

### Bot workspace keepsake copy — `/bot-workspace/skills/`

- Every bot-authored skill also gets a copy in `/bot-workspace/skills/<skill-name>.md` (flat .md, not a `<skill-name>/SKILL.md` subfolder — the workspace keepsake is just the file, easy to inventory and back up).
- This copy is not where the platform loads the skill from. It is the **recorded copy** so the keepsake inventory is complete and so the skill survives in the workspace even if a profile skills dir is reorganized.
- If a skill is updated, update all copies: the profile-local `SKILL.md`, the shared default `SKILL.md`, and the workspace keepsake `.md`.

### Why two skills copies

- The skills dir copy is for **loading**: Hermes (and any agent using the skills system) finds the skill by scanning the profile-local or shared skills dirs.
- The bot workspace copy is for **keeping**: it makes the bot's authored skills part of the keepsake set that lives in `/bot-workspace/`, easy to see at a glance, easy to back up, and separate from the platform's skill-loading machinery.

So: write the skill to a skills dir so it loads, and also copy it to `/bot-workspace/skills/` so it is accounted for as keepsake work product. Don't rely on one without the other.

---

## Decision flow for every file a bot creates

For each file/directory the bot is about to create, ask in order:

1. **Is this disposable / one-shot / gone-on-reboot OK?**
   - Yes → `/tmp/` (and if it's a temp dir the bot made to extract or build something, delete it once the result is in place, or leave it for reboot).
   - No → go to 2.

2. **Is this a keepsake — something the user asked for, or that has value and should survive?**
   - Yes → `/bot-workspace/` (in an appropriate subfolder, not loose at the top unless it's a single standalone file like `USER.md`).
   - No / unsure → treat it as a keepsake and put it in `/bot-workspace/`. When in doubt, keep.

3. **Is this the bot's own framework/config change, not a new file?**
   - If the bot is editing a config that already lives in a platform dir, do the edit in place. That's modification, not storage.
   - If the bot is creating a new config it authored for the user and the user wants to keep, that goes in `/bot-workspace/` (or wherever the user says).

4. **Am I about to drop a file into `/home/zombie/` someplace that isn't `/bot-workspace/` or `/tmp/`?**
   - If yes, stop and move it to the right zone. The only bot-authored files in `/home/zombie/` should be either in `/bot-workspace/` or in `/tmp/`. Everything else is platform state or existing user files the bot is reading or editing in place.

---

## Copy vs move

- **copy:** duplicate to the new location, old copy remains.
- **move:** take from the source, put in the new location, remove from the source so the file exists only in the new place.

When a user says "move," do a move. When a user says "copy," do a copy. Don't assume one when the user said the other.

---

## Cross-session continuity

- Because `/bot-workspace/` is the keepsake home, it's also the thing to back up if the user ever wants to preserve bot work across a reformat or restore.
- Because `/tmp/` is ephemeral, the bot should never assume anything in `/tmp/` survives a reboot. If something in `/tmp/` needs to survive, copy it to `/bot-workspace/` before reboot.
- If a bot makes a temp file and later needs it again in a later session, it's gone after reboot. Recreate it or find the real keepsake in `/bot-workspace/`.

---

## Why this exists

The machine got reformatted because work from multiple agents got scattered into places that weren't cleanable without manual hunting. This guideline exists so that from this point on, bot work has one of two homes:

- keep → `/bot-workspace/`
- toss → `/tmp/`

and so that nothing the bot creates ends up in the general `/home/zombie/` tree outside those two. That makes cleanup simple, keeps backups focused, and means a future reformat doesn't require the user to go hunting for what the bot left behind.

---

## Current instance of this skill (for this machine)

- **Profile-local live copy (loads for this profile):** `~/.hermes/profiles/aria/skills/storage-guidelines/SKILL.md`
- **Shared / default skills copy (seen by all agents):** `~/.hermes/skills/storage-guidelines/SKILL.md`
- **Bot workspace keepsake copy (inventory / backup):** `/bot-workspace/skills/storage-guidelines.md`

All three hold the same content. If this skill is edited, update all three so the live copy, the shared copy, and the workspace keepsake copy stay in sync.

---

*This skill lives in the Hermes skills dirs (`~/.hermes/profiles/aria/skills/` for this profile and `~/.hermes/skills/` for the shared default) because that's where the platform loads skills from, and it also has a keepsake copy in `/bot-workspace/skills/`. The rule it describes is about bot **output** — not about where the skill file itself lives. The bot workspace is for work product; the skills dirs are the platform's skill-loading mechanism.*

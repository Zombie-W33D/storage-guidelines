---
name: storage-guidelines
description: >
  Use when deciding where any file/directory the bot creates should live.
  Covers the bot workspace rule (creates→/home/zombie/bot-workspace/, throwaway→/tmp/),
  the keep zones (/home/zombie/bot-workspace/, /home/zombie/bot-database/, /home/zombie/bot-skillcode/, tmp), the skills
  storage locations (profile-local and shared/default), what belongs where, what
  never goes where, the bot-workspace layout, and a decision flow for every file
  the bot outputs. That keeps the machine clean, keeps keepsakes easy to back up,
  and keeps throwaway work out of /home/zombie/ so a reformat never requires
  manual cleanup.
---

# Storage guidelines — where bot output lives and why

## Rule of thumb

- If it's worth keeping, it goes in `/home/zombie/bot-workspace/`.
- If it's disposable, one-shot, or won't matter after a reboot, it goes in `/tmp/`.
- Bot-authored **work product** does not scatter into `/home/zombie/` outside those zones. Ever.

Everything below exists so a bot has a single, repeatable rule for every file it touches, so nothing gets parked in a random home-directory corner and so a reformat never leaves behind a trail of my own files.

---

## The keep zones

### 1. Bot workspace — `/home/zombie/bot-workspace/`

**What it is:** the home for everything the bot creates that has value and should survive a reboot.

**What goes here:**
- output files the user asked for
- configs the bot authored for the user
- project files, notes, plans, databases, schemas, seeds, source for generated pages
- anything that would be a pain to lose and that the user would want backed up

**What does NOT go here:**
- throwaway temp files (those go to `/tmp/`)
- the bot's own framework state (Hermes profile internals, systemd units, plugin dirs the platform owns — those live where the platform put them)

**Current layout in use:** (on this machine these live under `/home/zombie/`)
```
/home/zombie/bot-workspace/
├── albums/                 # bot-authored project folders (e.g. test album output)
├── USER.md                 # bot-authored user profile / research notes
├── skills/                 # loadable skill folders installed here (for this profile only — until we install to shared default)
│   └── image-gen/          # image-gen skill — still in progress, keep here while we work on it
└── generated/              # output from image-generation and similar tools (artifacts)

/home/zombie/bot-database/              # user database project — sibling of bot-workspace, NOT inside it
└── Bot-Database/
    └── database-plans.md

/home/zombie/bot-skillcode/             # backend code for skill-driven tools we actually run (python scripts, repos)
├── agent-backup-skill/     # Agent-Backup-Skill: backup.py + SKILL.md (the reusable skill doc + backing script)
└── agent-backup-tool/      # Agent-Backup-Tool: restore.py (tkinter GUI + CLI fallback for gpg .env restore)
```

Subfolders under `/home/zombie/bot-workspace/` should be created as the work requires. There is no fixed tree beyond what the work needs; the rule is just "kept things go here."

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

If there's doubt, assume keep and put it in `/home/zombie/bot-workspace/`. Only use `/tmp/` when the bot is confident the file is disposable.

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
- **Creating a brand-new bot-authored file and putting it there** — not the bot's output home. New keepsake files go to `/home/zombie/bot-workspace/`. New disposable files go to `/tmp/`.

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

### Backend skill code — `/home/zombie/bot-skillcode/`

- Some skills are backed by python scripts, repos, or other code that we actually **run** rather than just load as documentation. Examples: `agent-backup-skill/` (backup.py), `agent-backup-tool/` (restore.py).
- These are not loadable Hermes skills in the platform sense — they are standalone repos/repos we invoke directly via `python3 <script>`.
- They live in `/home/zombie/bot-skillcode/` as the dedicated home for this kind of backend code. Each gets its own subfolder (e.g. `/bot-skillcode/agent-backup-skill/`), preserving the git repo intact.
- `/home/zombie/bot-skillcode/` is a sibling of `/home/zombie/bot-workspace/` and `/home/zombie/bot-database/`, not inside either. It is part of the "keep" zone — these are things we built and use, and they should survive a reboot/restore.
- If a backend repo is updated, update the repo in place (it's the real working copy, not an inventory copy).

### Skill build + install workflow

This is the workflow for bot-authored loadable Hermes skills (the kind that live in `~/.hermes/profiles/<profile>/skills/` or `~/.hermes/skills/`):

1. **Build the skill in a workspace area.** For backend repos (python scripts, etc.) that's `/bot-skillcode/<skill-name>/`. For pure-doc skills, build in a workspace folder under `/home/zombie/bot-workspace/` or `/tmp/` while working — wherever is convenient during development.

2. **Verify it works.** Test the skill or script from where it's built before pushing anything.

3. **Upload to GH.** Push the built skill to its GH repo (e.g. `Zombie-W33D/<skill-name>`). GH is the durable storage and the thing that survives a reformat — not a local workspace copy.

4. **Install as a loadable skill (when the user requests it).** Copy the skill into the right skills dir:
   - Profile-local: `~/.hermes/profiles/<profile>/skills/<skill-name>/SKILL.md` — for this profile only.
   - Shared/default: `~/.hermes/skills/<skill-name>/SKILL.md` — seen by all agents.
   Installation is something the user asks for ("install this skill so you can use it") — the agent does not auto-install its own skills.

5. **Remove the original work files once the skill is fully installed and committed to GH.** Once the GH repo is pushed AND the skill is installed in the skills dir the user wanted, clean up the build workspace. There is no separate "keepsake copy" to maintain — GH is the keepsake for the skill source, and the skills dir copy is what makes it usable.

Do NOT keep a flat `.md` keepsake copy in `/bot-workspace/skills/` as a separate inventory artifact. That is wasted space — the GH repo is the durable record, and the installed skills-dir copy is what's actually used. If a build-area copy still exists in `/home/zombie/bot-workspace/` after install, remove it (unless the user wants to keep it for some reason).

**Note on the existing storage-guidelines.md in `/bot-workspace/skills/`:** that file is a leftover from the old "keep a flat .md keepsake copy" pattern. It is wasted space under the new workflow. It should be removed — the GH repo (`Zombie-W33D/storage-guidelines`) is the durable record, and the skills-dir copies are the live usable copies. Do not re-create keepsake `.md` copies in `/bot-workspace/skills/` for any skill; clean them out instead.

### Why no keepsake copies

- The GH repo is the durable storage for the skill source. It survives a reformat. That's the keepsake.
- The skills-dir copy is the live, usable copy that Hermes loads. That's the thing that matters day-to-day.
- A third copy in `/bot-workspace/skills/` as an "inventory" is redundant — it's an older pattern that's no longer needed now that each skill has its own GH repo as the durable record.

---

## Decision flow for every file a bot creates

For each file/directory the bot is about to create, ask in order:

1. **Is this disposable / one-shot / gone-on-reboot OK?**
   - Yes → `/tmp/` (and if it's a temp dir the bot made to extract or build something, delete it once the result is in place, or leave it for reboot).
   - No → go to 2.

2. **Is this a keepsake — something the user asked for, or that has value and should survive?**
   - Yes → `/home/zombie/bot-workspace/` (in an appropriate subfolder, not loose at the top unless it's a single standalone file like `USER.md`).
   - No / unsure → treat it as a keepsake and put it in `/home/zombie/bot-workspace/`. When in doubt, keep.

3. **Is this the bot's own framework/config change, not a new file?**
   - If the bot is editing a config that already lives in a platform dir, do the edit in place. That's modification, not storage.
   - If the bot is creating a new config it authored for the user and the user wants to keep, that goes in `/home/zombie/bot-workspace/` (or wherever the user says).

4. **Am I about to drop a file into `/home/zombie/` someplace that isn't `/home/zombie/bot-workspace/`, `/home/zombie/bot-database/`, `/home/zombie/bot-skillcode/`, or `/tmp/`?**
   - If yes, stop and move it to the right zone. The only bot-authored files in `/home/zombie/` should be either in `/home/zombie/bot-workspace/`, `/home/zombie/bot-database/`, `/home/zombie/bot-skillcode/`, or `/tmp/`. Everything else is platform state or existing user files the bot is reading or editing in place.

---

## Copy vs move

- **copy:** duplicate to the new location, old copy remains.
- **move:** take from the source, put in the new location, remove from the source so the file exists only in the new place.

When a user says "move," do a move. When a user says "copy," do a copy. Don't assume one when the user said the other.

---

## Cross-session continuity

- Because `/home/zombie/bot-workspace/` is the keepsake home, it's also the thing to back up if the user ever wants to preserve bot work across a reformat or restore.
- Because `/tmp/` is ephemeral, the bot should never assume anything in `/tmp/` survives a reboot. If something in `/tmp/` needs to survive, copy it to `/home/zombie/bot-workspace/` before reboot.
- If a bot makes a temp file and later needs it again in a later session, it's gone after reboot. Recreate it or find the real keepsake in `/home/zombie/bot-workspace/`.

---

## Why this exists

The machine got reformatted because work from multiple agents got scattered into places that weren't cleanable without manual hunting. This guideline exists so that from this point on, bot work has one of four keep zones:

- `/home/zombie/bot-workspace/` — general bot-authored keepsakes (output files, configs, project files, notes, plans, albums, generated artifacts)
- `/home/zombie/bot-database/` — user database project (sibling of bot-workspace, NOT inside it)
- `/home/zombie/bot-skillcode/` — backend code for skill-driven tools we actually run (python scripts, repos; e.g. agent-backup-skill, agent-backup-tool)
- `/tmp/` — throwaway

and so that nothing the bot creates ends up in the general `/home/zombie/` tree outside those zones. That makes cleanup simple, keeps backups focused, and means a future reformat doesn't require the user to go hunting for what the bot left behind.

---

## Current instance of this skill (for this machine)

- **Profile-local live copy (loads for this profile):** `~/.hermes/profiles/aria/skills/storage-guidelines/SKILL.md`
- **Shared / default skills copy (seen by all agents):** `~/.hermes/skills/storage-guidelines/SKILL.md`

These are the two live copies. The profile-local one loads for this profile; the shared default one is seen by all agents. The GH repo (`Zombie-W33D/storage-guidelines`) is the durable storage — if both local copies are lost, re-install from GH. There is no separate "keepsake copy" to maintain.

If this skill is edited, update the profile-local copy and the shared default copy. Push to GH so the durable record stays current.

---

*This skill lives in the Hermes skills dirs (`~/.hermes/profiles/aria/skills/` for this profile and `~/.hermes/skills/` for the shared default) because that's where the platform loads skills from. The rule it describes is about bot **output** — not about where the skill file itself lives. The bot workspace is for work product; the skills dirs are the platform's skill-loading mechanism. Backend code repos for skills we run live in `/home/zombie/bot-skillcode/` instead. There are no flat .md keepsake copies in `/bot-workspace/skills/` — GH is the durable record for skill source, and the skills-dir copies are the live usable copies.*

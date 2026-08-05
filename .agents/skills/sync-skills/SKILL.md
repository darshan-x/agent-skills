---
name: sync-skills
description: Sync agent skills from a central GitHub repository so every project stays up to date. Run when the user says "sync my skills", "pull latest skills", or "update skills from my repo". Also covers one-time setup of the GitHub skills registry.
---

# Sync Skills from GitHub

This skill keeps `.agents/skills/` in sync with a central GitHub repository so skills you refine in one project automatically become available in all others.

## How It Works

- A GitHub repo (e.g. `yourname/agent-skills`) is the single source of truth
- Each project stores the registry URL in `.agents/skills-source` (one line: the raw GitHub base URL)
- Running the sync prompt overwrites local skill files with the latest from the registry

---

## Step 1 — One-Time Setup (do this once per GitHub repo)

### 1a. Create the GitHub registry repo

Create a new public (or private) GitHub repo. Suggested name: `agent-skills`.

Inside it, mirror the structure used here:
```
.agents/skills/task-bloat-audit/SKILL.md
.agents/skills/sync-skills/SKILL.md
... (any other skills you want shared)
```

Push your current skills to it:
```bash
# From this project's root, export current skills
cp -r .agents/skills /tmp/agent-skills-export
cd /tmp/agent-skills-export
git init
git remote add origin https://github.com/YOURNAME/agent-skills.git
git add .
git commit -m "Initial skills export"
git push -u origin main
```

### 1b. Record the registry URL in this project

Create `.agents/skills-source` with one line — the raw GitHub content base URL:
```
https://raw.githubusercontent.com/YOURNAME/agent-skills/main
```

Example for a user named `jdoe`:
```
https://raw.githubusercontent.com/jdoe/agent-skills/main
```

### 1c. Seed new projects

When starting a new project, copy two files from your starter repl:
- `.agents/skills/` (the whole directory)
- `.agents/skills-source` (the registry URL)

That's it — the new project is wired up.

---

## Step 2 — Running a Sync

When the user says **"sync my skills"** or **"pull latest skills from my repo"**, follow these steps:

### Read the registry URL
```bash
cat .agents/skills-source
# Should output: https://raw.githubusercontent.com/YOURNAME/agent-skills/main
```

If the file doesn't exist, run the one-time setup above first.

### Fetch the skill manifest
The registry repo must contain a `.agents/skills/MANIFEST` file listing every skill directory, one per line:
```
task-bloat-audit
sync-skills
```

Fetch it:
```bash
BASE_URL=$(cat .agents/skills-source)
curl -fsSL "$BASE_URL/.agents/skills/MANIFEST"
```

### Sync each skill
For each skill name in the manifest:
```bash
BASE_URL=$(cat .agents/skills-source)
SKILL=task-bloat-audit   # repeat for each skill in MANIFEST

mkdir -p ".agents/skills/$SKILL"
curl -fsSL "$BASE_URL/.agents/skills/$SKILL/SKILL.md" \
     -o ".agents/skills/$SKILL/SKILL.md"
echo "✓ $SKILL"
```

Or as a shell one-liner (run after reading MANIFEST into a variable):
```bash
BASE_URL=$(cat .agents/skills-source)
for SKILL in $(curl -fsSL "$BASE_URL/.agents/skills/MANIFEST"); do
  mkdir -p ".agents/skills/$SKILL"
  curl -fsSL "$BASE_URL/.agents/skills/$SKILL/SKILL.md" \
       -o ".agents/skills/$SKILL/SKILL.md"
  echo "✓ synced $SKILL"
done
echo "Sync complete."
```

---

## Step 3 — Publishing a New or Updated Skill to the Registry

When you write a new skill or update an existing one and want it available everywhere:

1. Copy the skill file into the GitHub repo (via the Replit Git panel, or push manually)
2. Add the skill name to `.agents/skills/MANIFEST` in the repo if it's new
3. Commit and push to `main`
4. In any other project, run the sync prompt

---

## MANIFEST format

The `.agents/skills/MANIFEST` file in the GitHub registry is plain text, one skill directory name per line, no trailing slashes, no blank lines:

```
task-bloat-audit
sync-skills
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `curl` returns 404 | Check the raw URL — GitHub raw URLs are case-sensitive. Confirm `main` vs `master`. |
| `.agents/skills-source` not found | Run one-time setup (Step 1b) |
| Skill file overwrites local changes | Commit or back up local edits before syncing — sync always overwrites |
| Private repo returns 401 | Use a GitHub personal access token: `curl -H "Authorization: token YOURTOKEN" ...` Store the token in a Replit secret named `GITHUB_SKILLS_TOKEN` and read it with `$GITHUB_SKILLS_TOKEN` in the shell |

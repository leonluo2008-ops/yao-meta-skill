# OpenWork Plugin Compatibility

OpenWork Den imports skills from GitHub repos using the **Claude Code plugin format**.
A Hermes skill repo needs two additions before Den can preview and import it.

> Verified against Den v0.17.34 source (`github-discovery.js`). 2026-08-04.

## Required Directory Layout

```
your-skill-repo/
├── .claude-plugin/
│   └── plugin.json               # Required — Den identifies repo as Claude plugin
├── skills/
│   └── <skill-name>/
│       ├── SKILL.md              # Required — Den discovers this
│       ├── references/           # Optional
│       ├── templates/            # Optional
│       └── scripts/              # Optional
├── SKILL.md                      # Original Hermes root file (keep for Hermes)
├── references/                   # Original Hermes files (keep for Hermes)
└── ...
```

Both additions are **mandatory**:
- Only `plugin.json`, no `skills/` → Den classifies correctly but finds **0 skills**.
- Only `skills/<name>/SKILL.md`, no `plugin.json` → Den returns `unsupported` + warning.

### Den source scan paths

`github-discovery.js` checks exactly two locations for skill directories:
- `skills/` (root-level)
- `.claude/skills/` (hidden config dir)

Root-level `SKILL.md` is **not** scanned.

## plugin.json Minimal Template

```json
{
  "name": "your-skill-name",
  "version": "1.0.0",
  "description": "One-sentence description",
  "author": { "name": "Hermes" }
}
```

Rules:
- `name`: kebab-case, unique, no spaces. Maps to plugin name in Den.
- Only `name` is strictly required by the Claude spec, but Den uses `description` for display.

## Adaptation Workflow

When packaging a Hermes skill repo for OpenWork import:

1. **Create `.claude-plugin/plugin.json`** at repo root.
2. **Create `skills/<skill-name>/`** directory.
3. **Copy** (not move) these into the new directory:
   - `SKILL.md`
   - `references/` (if exists)
   - `templates/` (if exists)
   - `scripts/` (if exists)
4. **Keep originals at root** — Hermes loads from root; Den loads from `skills/`.
5. Commit and push to the repo's **default branch** (Den reads default branch only).

### Quick adapt script

```bash
# Run from repo root
SKILL_NAME=$(basename $(pwd))
mkdir -p .claude-plugin "skills/$SKILL_NAME"
cat > .claude-plugin/plugin.json << EOF
{"name":"$SKILL_NAME","version":"1.0.0","description":"<one sentence>","author":{"name":"Hermes"}}
EOF
cp SKILL.md "skills/$SKILL_NAME/"
cp -r references/ "skills/$SKILL_NAME/references/" 2>/dev/null
cp -r templates/ "skills/$SKILL_NAME/templates/" 2>/dev/null
cp -r scripts/ "skills/$SKILL_NAME/scripts/" 2>/dev/null
git add .claude-plugin/ skills/
git commit -m "Add Claude plugin structure for OpenWork compatibility"
```

## Import via Den API (Automation)

```bash
# 1. Login → extract session cookie
COOKIE="better-auth.session_token=<from Set-Cookie>"

# 2. Preview → get skillKey
curl -s -X POST http://<den-host>:8788/v1/plugins/import-mcps-from-github-url/preview \
  -H "Content-Type: application/json" -H "Cookie: $COOKIE" \
  -d '{"githubUrl":"https://github.com/user/repo"}'
# → item.skills[].skillKey = "manifest%3Aroot:skills%2F<name>%2FSKILL.md"

# 3. Import — param name is selectedSkillKeys (string array), NOT selectedSkills
curl -s -X POST http://<den-host>:8788/v1/plugins/import-mcps-from-github-url \
  -H "Content-Type: application/json" -H "Cookie: $COOKIE" \
  -d '{"githubUrl":"https://github.com/user/repo","selectedSkillKeys":["<skillKey>"],"name":"plugin-name","access":{"orgWide":true,"memberIds":[],"teamIds":[]}}'
```

## GitHub App Requirement

Den must have a GitHub App configured (5 env vars) to avoid 60 req/hour anonymous API rate limit.
Without it, Preview fails with `github_request_failed: API rate limit exceeded`.

See `openwork-den-specialist` skill → `references/github-plugin-import.md` for full setup.

## Pitfalls

1. **Default branch only** — Den reads the repo's default branch. Changes on feature branches are invisible.
2. **`selectedSkillKeys` not `selectedSkills`** — Import API expects a flat string array, not objects.
3. **SKILL.md in root is ignored** — Must be at `skills/<name>/SKILL.md`.
4. **Anonymous rate limit** — 60 req/hour per IP without GitHub App; 5000 req/hour with App.
5. **Web UI "0 selected"** — Components need manual checkbox selection before "Continue to create".

---
description: Forgein — stop explaining yourself to AI. Subcommands: auth, who, ctx, optimize, mem, setup, hook, sec.
---

Parse `$ARGUMENTS` — first word is the subcommand. Route to the correct section below.

If no argument or unrecognized subcommand: print the usage table and stop.

```
FORGEIN — AI context that follows you everywhere

  No more pasting the same context block every session.
  No more rebuilding your AI from scratch when you switch machines.
  No more explaining your team's conventions to every new hire's AI.

  ──────────────────────────────────────────────────────────────────
  COMMAND                          WHAT IT DOES                TIER
  ──────────────────────────────────────────────────────────────────
  /forgein auth                    Connect to your account     free
  /forgein who                     What does AI know about     free
                                   you right now?
  /forgein ctx                     Show your active context    free
  /forgein ctx <name>              Switch context              free
                                   (work · home · family)

  /forgein optimize                Find skills for your        free
                                   workflow from the registry
  /forgein mem                     List your memory files      free
  /forgein mem list                List grouped by context     free
  /forgein mem search <query>      Search memory bodies        free
  /forgein mem add <content>       Add a new memory entry      free
    [--ctx work|home|family]
  /forgein mem prune               Remove stale memories       free
  /forgein mem audit               Check index integrity       free
  /forgein mem sync                Sync to cloud — all         free
                                   machines, all contexts
  /forgein mem sync --team         Sync team-shared folder     PRO ★
  /forgein mem delete <file>       Remove a file from cloud    free

  /forgein setup                   First-run setup: dirs,      free
                                   auto-inject hook, CLAUDE.md
  /forgein hook install            Wire auto-inject into       free
                                   Claude Code sessions
  /forgein hook status             Check hook installation     free
  /forgein hook remove             Remove the hook             free

  /forgein sec                     Security scan of staged     free
                                   git changes
  /forgein sec <path>              Deep scan of any path       PRO ★
  ──────────────────────────────────────────────────────────────────
  PRO: $4.99/month · app.forgein.ai/upgrade
  ──────────────────────────────────────────────────────────────────
```

---

## Context model

Before executing any subcommand, understand the three-context model:

- **💼 Work** — codebase conventions, team standards, sprint context. Has a `team-shared/` subfolder that teammates load too.
- **🏠 Home** — side projects, learning notes, personal hobby context. Never visible to work team.
- **👨‍👩‍👧 Family** — kids' schedules, household context, personal life AI uses. Always private.

Context files live in `~/.forgein/contexts/<name>/`. The active context is stored in `~/.config/forgein/active-context`.

Auto-detection: read `~/.config/forgein/active-context`. If missing, infer from working directory:
- Path contains employer/work project name or is under `~/work/`, `~/projects/<org>/` → `work`
- Path under `~/home/`, `~/personal/`, `~/side/` → `home`
- No match → `work` (safe default for pro users; free users always use `work`)

---

## auth

Authenticate with a forgein API token. Token stored at `~/.config/forgein/token`.

**Step 1 — Check existing token**

```bash
cat ~/.config/forgein/token 2>/dev/null
```

If found, verify:
```bash
curl -sf -H "Authorization: Bearer $(cat ~/.config/forgein/token)" https://api.forgein.ai/api/auth/user
```

- Valid JSON with `email` field → print `✓ Already authenticated as <email> · plan: <plan>` and stop.
- 401 or empty → token is stale, proceed to re-authenticate.

**Step 2 — Guide user to authenticate**

Print:
```
To authenticate forgein, choose a method:

  [1] Device login (recommended — token never passes through this chat)
      Run in your terminal:  forgein login
      This opens a browser, you approve, the token is stored locally.

  [2] Paste a token manually
      Open https://app.forgein.ai/dashboard/tokens
      Click "New token", copy the full token (starts with fg_)
      Then paste it here.

Choice [1/2]:
```

If the user chooses **1** or presses Enter: print `Run \`forgein login\` in your terminal to complete authentication, then return here and run /forgein auth to verify.` and stop.

If the user chooses **2**: print `Paste your token:` and read input. If it doesn't start with `fg_`: print `✗ That doesn't look like a forgein token (should start with fg_). Try again.` and re-prompt once.

**Step 3 — Verify and store**

```bash
curl -sf -H "Authorization: Bearer <token>" https://api.forgein.ai/api/auth/user
```

- HTTP 200 with JSON:
  ```bash
  mkdir -p ~/.config/forgein
  echo "<token>" > ~/.config/forgein/token
  chmod 600 ~/.config/forgein/token
  ```
  Print: `✓ Authenticated as <email> · plan: <plan>. Token saved.`

- 401: `✗ Token rejected — check that you copied the full token.` and stop.
- Network error: `✗ Could not reach api.forgein.ai — check your connection.` and stop.

---

## setup

First-run setup that wires forgein end-to-end: directories, auto-inject hook, CLAUDE.md import, and initial context. Safe to re-run — idempotent at every step.

Print header:
```
FORGEIN SETUP
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Step 1 — Authentication

Check `~/.config/forgein/token`. If present, verify:
```bash
curl -sf -H "Authorization: Bearer $(cat ~/.config/forgein/token)" https://api.forgein.ai/api/auth/user
```

- Valid → print `  ✓ Authenticated as <email> (plan: <plan>)` and continue.
- Invalid or missing → run the full `auth` flow (see `## auth`), then continue.

### Step 2 — Directory structure

Create all required directories:
```bash
mkdir -p ~/.forgein/bin
mkdir -p ~/.forgein/contexts/work/team-shared
mkdir -p ~/.forgein/contexts/home
mkdir -p ~/.forgein/contexts/family
mkdir -p ~/.config/forgein
```

Print: `  ✓ Directory structure created at ~/.forgein/`

### Step 3 — Auto-inject hook script

Write the following content to `~/.forgein/bin/auto-inject.sh` and make it executable. Skip if the file already exists and is identical.

```bash
cat > ~/.forgein/bin/auto-inject.sh << 'HOOKEOF'
#!/usr/bin/env bash
# forgein auto-inject — keeps ~/.forgein/active-context.md fresh
# Claude Code reads this via CLAUDE.md @import at every session.
set -euo pipefail
TOKEN_FILE="$HOME/.config/forgein/token"
CTX_FILE="$HOME/.forgein/active-context.md"
MARKER="$HOME/.forgein/.last-inject"
[ -f "$TOKEN_FILE" ] || exit 0
TOKEN=$(cat "$TOKEN_FILE")
[ -n "$TOKEN" ] || exit 0
if [ -f "$MARKER" ]; then
  LAST=$(cat "$MARKER" 2>/dev/null || echo 0)
  NOW=$(date +%s)
  if [ $((NOW - LAST)) -lt 3600 ]; then exit 0; fi
fi
RESPONSE=$(curl -sf --max-time 5 \
  -H "Authorization: Bearer $TOKEN" \
  "https://api.forgein.ai/api/contexts/active" 2>/dev/null) || exit 0
[ -z "$RESPONSE" ] && exit 0
command -v jq &>/dev/null || exit 0
CTX=$(echo "$RESPONSE" | jq -r '.context // "work"' 2>/dev/null || echo "work")
FILE_COUNT=$(echo "$RESPONSE" | jq '.files | length' 2>/dev/null || echo 0)
mkdir -p "$(dirname "$CTX_FILE")"
if [ "$FILE_COUNT" -eq 0 ]; then
  printf '<!-- forgein: %s context — no files synced yet -->\nRun /forgein mem sync to push your memories.\n' "$CTX" > "$CTX_FILE"
else
  { echo "<!-- forgein: $CTX context | $FILE_COUNT files | $(date -u '+%Y-%m-%dT%H:%M:%SZ') -->"
    echo ""
    echo "$RESPONSE" | jq -r '.files[] | "## " + .filePath + "\n\n" + .content + "\n"' 2>/dev/null
  } > "$CTX_FILE"
  echo "[forgein] $CTX context loaded ($FILE_COUNT files)" >&2
fi
date +%s > "$MARKER"
exit 0
HOOKEOF
chmod +x ~/.forgein/bin/auto-inject.sh
```

Print: `  ✓ Auto-inject script written to ~/.forgein/bin/auto-inject.sh`

### Step 4 — Wire hook into Claude Code settings

Read `~/.claude/settings.json` (create `{}` if missing). Add the PreToolUse hook using `jq` if available, otherwise print manual instructions.

**If jq is available:**
```bash
SETTINGS="$HOME/.claude/settings.json"
[ -f "$SETTINGS" ] || echo '{}' > "$SETTINGS"

# Merge hook — safe even if hooks already exist
jq '
  .hooks.PreToolUse //= [] |
  if (.hooks.PreToolUse | map(select(.hooks[]?.command? | contains("auto-inject"))) | length) == 0
  then .hooks.PreToolUse += [{"matcher": ".*", "hooks": [{"type": "command", "command": "bash \"$HOME/.forgein/bin/auto-inject.sh\""}]}]
  else . end
' "$SETTINGS" > "$SETTINGS.tmp" && mv "$SETTINGS.tmp" "$SETTINGS"
```

Print: `  ✓ PreToolUse hook added to ~/.claude/settings.json`

**If jq is not available:** print:
```
  ⚠ jq not found — add this to ~/.claude/settings.json manually:

    "hooks": {
      "PreToolUse": [{
        "matcher": ".*",
        "hooks": [{"type": "command", "command": "bash \"$HOME/.forgein/bin/auto-inject.sh\""}]
      }]
    }
```

### Step 5 — CLAUDE.md import

Check if `~/.claude/CLAUDE.md` already contains `@~/.forgein/active-context.md`. If not, prepend:
```bash
CLAUDEMD="$HOME/.claude/CLAUDE.md"
if ! grep -qF '@~/.forgein/active-context.md' "$CLAUDEMD" 2>/dev/null; then
  printf '@~/.forgein/active-context.md\n\n' > "$CLAUDEMD.tmp"
  [ -f "$CLAUDEMD" ] && cat "$CLAUDEMD" >> "$CLAUDEMD.tmp"
  mv "$CLAUDEMD.tmp" "$CLAUDEMD"
  echo "  ✓ @~/.forgein/active-context.md prepended to ~/.claude/CLAUDE.md"
else
  echo "  ✓ CLAUDE.md already imports active-context.md (skipped)"
fi
```

### Step 6 — Print completion summary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ Setup complete — forgein is now active.

  Every Claude Code session will auto-load your context.
  No manual /forgein mem sync needed at session start.

  Next steps:
    /forgein mem sync       Push your memory files to cloud
    /forgein who            See what AI knows about you now
    /forgein ctx home       Switch to your Home context (Pro)
    /forgein optimize       Find skills for your workflow
```

If plan is `free`, append:
```
  Upgrade to Pro for Home/Family contexts + team sync:
    app.forgein.ai/upgrade
```

---

## hook

Manage the forgein PreToolUse hook in Claude Code. Subcommands: `install`, `status`, `remove`.

If no subcommand: run `status`.

### hook install

Performs Steps 3–5 from `## setup` only (skips auth and directory creation). Use after a fresh Claude Code install or on a new machine where setup was already run.

1. Write `~/.forgein/bin/auto-inject.sh` (skip if already exists). The script uses parent-PID session detection — it fires exactly once per Claude Code session by writing a marker file keyed to the parent process PID, then checking for that marker on subsequent tool calls within the same session. Stale markers (for dead PIDs) are cleaned up on each run.
2. Wire PreToolUse hook into `~/.claude/settings.json`
3. Ensure `~/.claude/CLAUDE.md` imports `@~/.forgein/active-context.md`

Print on success:
```
✓ forgein hook installed.
  Auto-inject will run at the start of every Claude Code session.
  Verify with: /forgein hook status
```

### hook status

Check each component and print status table:
```bash
# Check script exists
[ -f "$HOME/.forgein/bin/auto-inject.sh" ] && echo "script: ok" || echo "script: missing"

# Check settings.json hook
grep -q "auto-inject" "$HOME/.claude/settings.json" 2>/dev/null && echo "settings: ok" || echo "settings: missing"

# Check CLAUDE.md import
grep -qF '@~/.forgein/active-context.md' "$HOME/.claude/CLAUDE.md" 2>/dev/null && echo "claudemd: ok" || echo "claudemd: missing"

# Check active-context.md freshness
[ -f "$HOME/.forgein/active-context.md" ] && stat "$HOME/.forgein/active-context.md" || echo "ctx-file: not yet created"
```

Print:
```
FORGEIN HOOK STATUS

  Script       ~/.forgein/bin/auto-inject.sh     ✓ present
  Settings     ~/.claude/settings.json hook      ✓ wired
  CLAUDE.md    @~/.forgein/active-context.md     ✓ imported
  Context file ~/.forgein/active-context.md      ✓ fresh (4 min ago)

  → Hook is fully operational. Context auto-loads at session start.
```

Or for each missing component: `✗ missing — run /forgein hook install to fix`

### hook remove

Remove the hook — useful when debugging or temporarily disabling forgein injection.

1. Remove PreToolUse hook entry from `~/.claude/settings.json`:
```bash
jq 'del(.hooks.PreToolUse[] | select(.hooks[]?.command? | contains("auto-inject")))' \
  ~/.claude/settings.json > /tmp/settings.tmp && mv /tmp/settings.tmp ~/.claude/settings.json
```

2. Remove `@~/.forgein/active-context.md` line from `~/.claude/CLAUDE.md`:
```bash
sed -i.bak '/@~\/.forgein\/active-context\.md/d' ~/.claude/CLAUDE.md
```

3. Print:
```
✓ forgein hook removed.
  Your context files are still intact — re-run /forgein hook install to restore.
```

---

## who

Show a plain-language summary of exactly what AI sees when a session starts. Eliminates the need to re-read CLAUDE.md to know what context is loaded.

**Step 1 — Read active context**

```bash
cat ~/.config/forgein/active-context 2>/dev/null
```

If missing, infer from cwd using the context model above.

**Step 2 — Count loaded files**

```bash
CONTEXT_DIR="$HOME/.forgein/contexts/$(cat ~/.config/forgein/active-context 2>/dev/null || echo 'work')"
find "$CONTEXT_DIR" -name "*.md" -type f 2>/dev/null
```

Separate team-shared (under `team-shared/`) from personal files.

**Step 3 — Check token + plan**

```bash
cat ~/.config/forgein/token 2>/dev/null
```

If token exists, fetch:
```bash
curl -sf -H "Authorization: Bearer $(cat ~/.config/forgein/token)" https://api.forgein.ai/api/auth/user
```

Extract `plan`, `orgName`, `orgMemberCount` if available.

**Step 4 — Print summary**

```
WHAT AI KNOWS ABOUT YOU RIGHT NOW

  Context:    💼 Work  (auto-detected: ~/projects/acme)
  Machine:    work-macbook
  Loaded:     6 files  (2 team-shared · 4 personal)
  Team:       acme-eng · 4 members see your shared files
  AI tools:   Claude Code (active) · ChatGPT (coming) · Gemini (coming)
  Synced:     2 hours ago

  Personal files:
    📄 my-architecture-notes.md
    📄 career-preferences.md
    📄 sprint-2-context.md
    📄 tools-i-use.md

  Team-shared files:
    📄 codebase-conventions.md
    📄 review-standards.md

Run /forgein ctx home to load your home context instead.
Run /forgein mem add "..." to add something new right now.
```

If not authenticated: show local files only, omit Team and Synced lines, add:
```
  → Run /forgein auth to unlock cloud sync across machines.
```

If not on Pro and Home/Family contexts are loaded: show only Work files, omit other contexts, add:
```
  → Upgrade to Pro to load Home and Family contexts: app.forgein.ai/upgrade
```

---

## ctx

Show or switch the active context.

Parse next word as context name. If none: show current context and available options.

### ctx (no argument) — show

```bash
cat ~/.config/forgein/active-context 2>/dev/null
```

Print:
```
ACTIVE CONTEXT: 💼 Work  (auto-detected: ~/projects/acme)

Available contexts:
  💼  work    6 files  (2 team-shared · 4 personal)
  🏠  home    4 files
  👨‍👩‍👧  family  5 files  [private — never shared]

Switch:  /forgein ctx home
Details: /forgein who
```

### ctx \<name\> — switch

Valid names: `work`, `home`, `family`.

**Step 1 — Check context exists**

```bash
ls "$HOME/.forgein/contexts/<name>/" 2>/dev/null
```

If missing: print `✗ No context named "<name>". Available: work, home, family` and stop.

**Step 2 — Check plan for non-work contexts (free tier)**

```bash
cat ~/.config/forgein/token 2>/dev/null
```

If no token or plan is `free` and context is `home` or `family`:
```
✗ Multiple contexts (Home · Family) require forgein Pro.
  Your Work context is always available free.
  Upgrade: app.forgein.ai/upgrade
```
Stop.

**Step 3 — Switch**

```bash
echo "<name>" > ~/.config/forgein/active-context
```

List files in the new context directory and print:
```
✓ Switched to 🏠 Home context

  Loaded 4 files:
    📄 forgein-platform.md     (side-projects/)
    📄 open-source.md          (side-projects/)
    📄 rust-notes.md           (learning/)
    📄 book-deep-work.md       (learning/)

  These files are private — not shared with any team.
  Run /forgein who for a full summary.
```

If context has no files yet:
```
✓ Switched to 🏠 Home context (empty — no files yet)

  Start building your home context:
    /forgein mem add "I work on forgein in evenings — Next.js 15, Drizzle, Postgres"
    /forgein mem add "Side project goals: launch forgein to 100 users by August"
```

**Step 4 — Sync active context to cloud (if authenticated)**

```bash
TOKEN=$(cat ~/.config/forgein/token 2>/dev/null)
```

If token exists:
```bash
curl -sf -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"context\": \"<name>\"}" \
  https://api.forgein.ai/api/contexts/switch
```

- On 200: silent — context is synced, any machine you authenticate on will see this context.
- On 403 (PRO_REQUIRED): you should have already caught this in Step 2; treat as Step 2 failure — show upgrade prompt and revert the local file.
- On network error: print `  ⚠ Could not sync context to cloud — working locally.` and continue (local switch still works).

---

## sync-adapter

Sync forgein context to a specific AI tool's native format. Sub-subcommands: `copilot`, `chatgpt`, `gemini`.

Usage: `/forgein sync-adapter <tool> [--project <path>] [--out <file>]`

**Step 1 — Load token**
```bash
cat ~/.config/forgein/token 2>/dev/null
```
If missing: print `✗ Run /forgein auth first.` and stop.

**Step 2 — Get active context and project path**
```bash
cat ~/.config/forgein/active-context 2>/dev/null
PROJECT_PATH=$(pwd | sed 's|/|-|g')
```

**Step 3 — Call adapter API**

For `copilot`:
```bash
TOKEN=$(cat ~/.config/forgein/token)
PROJECT=$(pwd | sed 's|/|-|g')
curl -sf \
  -H "Authorization: Bearer $TOKEN" \
  "https://api.forgein.ai/api/adapters/copilot?projectPath=$PROJECT"
```

Write the output to `.github/copilot-instructions.md`:
```bash
mkdir -p .github
curl -sf -H "Authorization: Bearer $TOKEN" \
  "https://api.forgein.ai/api/adapters/copilot?projectPath=$PROJECT" \
  > .github/copilot-instructions.md
```

Print:
```
✓ Copilot instructions synced → .github/copilot-instructions.md
  Context: 💼 Work · 6 files (2 team-shared · 4 personal)
  Reload Copilot in VS Code to pick up the new instructions.
```

For `chatgpt`:
```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  "https://api.forgein.ai/api/adapters/chatgpt?projectPath=$PROJECT"
```
Parse JSON response. Print:
```
CHATGPT CUSTOM INSTRUCTIONS — copy into ChatGPT Settings → Personalization

── What would you like ChatGPT to know about you? ──
<whatToKnow content>

── How would you like ChatGPT to respond? ──
<howToRespond content>

→ Open: https://chat.openai.com/#settings/Personalization
```

For `gemini`:
```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  "https://api.forgein.ai/api/adapters/gemini?projectPath=$PROJECT"
```
Parse JSON and print the `systemPrompt` with instructions to paste it into a Gem.

---

## optimize

Discover and install skills from the forgein public registry that match your workflow.

**Step 1 — Fetch registry**

Try in order, use first that succeeds:
```bash
gh api repos/forgeinai/forgein/contents/registry.json --jq '.content' | base64 -d
```
Fallback: WebFetch `https://raw.githubusercontent.com/forgeinai/forgein/main/registry.json`

Parse JSON. Extract `skills` array.

**Step 2 — Inventory installed skills**

```bash
ls ~/.claude/commands/ 2>/dev/null
```

Strip `.md` extensions. Store as set of installed command names. Exclude from recommendations.

**Step 3 — Gather context signals**

```bash
cat ~/.claude/CLAUDE.md 2>/dev/null
cat CLAUDE.md 2>/dev/null
find ~/.claude/projects -name "MEMORY.md" 2>/dev/null | head -3 | xargs cat 2>/dev/null
find "$HOME/.forgein/contexts" -name "*.md" -type f 2>/dev/null | head -10 | xargs cat 2>/dev/null
git log --oneline -20 2>/dev/null
```

Concatenate all output into one context string.

**Step 4 — Score skills (whole-word matching)**

For each uninstalled skill:
```
score = 0
for term in (skill.signals + skill.tags):
    if whole-word match of term in context (case-insensitive):
        score += 1
```

Whole-word: character before and after the match must be non-alphanumeric (not just whitespace). `"pr"` must NOT match inside `"sprint"` or `"improve"`.

**Step 5 — Present top 5**

```
RECOMMENDED SKILLS FOR YOUR WORKFLOW

#  Skill        Score  What it does                        Why it fits you
1  pr-monitor     6    CI status across all open PRs       Matched: "open prs", "github actions"
2  code-review    4    Multi-dimensional diff review       Matched: "pull request", "review"
3  commit         3    Smart conventional commit messages  Matched: "commits", "git"
4  vibe-sec       2    Security scan of staged changes     Matched: "security", "auth"
5  standup        0    Daily standup from git + PRs        No direct signals — ranked by registry order

All skills above are from the public registry — free to install.
```

For skills with score 0, the "Why it fits you" column must read `No direct signals — ranked by registry order` (never show `Matched:` with an empty list).

If plan is `pro`: also show private skills the user has created, labeled `[private]`.

**Step 6 — Install**

Ask: `Install all 5? [Enter = yes / numbers e.g. 1,3 / none]:`

Default (Enter with no input) = install all.

For each selected skill:
```bash
gh api repos/forgeinai/forgein/contents/<file> --jq '.content' | base64 -d
```
Write to `~/.claude/commands/<install_as>`.

Confirm: `✓ /commit installed — type /commit to use it.`

If user enters `none` or `n`: exit without installing.

---

## mem

Parse the next word as the mem subcommand. Default (no word): run `list`.

Subcommands: `list`, `search`, `add`, `prune`, `audit`, `delete`, `sync`.

**Locate memory directory:**

Claude Code normalizes the working directory path to a project key by replacing all non-alphanumeric characters (slashes, spaces, underscores, etc.) with hyphens. Derive the project directory by walking up from `pwd` until a matching `~/.claude/projects/` subdirectory is found:

```bash
SEARCH_DIR=$(pwd)
MEMORY_DIR=""
while [ "$SEARCH_DIR" != "/" ]; do
  PROJECT_PATH=$(echo "$SEARCH_DIR" | sed 's|[^a-zA-Z0-9]|-|g')
  CANDIDATE="$HOME/.claude/projects/$PROJECT_PATH/memory"
  if [ -d "$CANDIDATE" ]; then
    MEMORY_DIR="$CANDIDATE"
    break
  fi
  SEARCH_DIR=$(dirname "$SEARCH_DIR")
done
```

If `MEMORY_DIR` is still empty after the loop: print `Memory not configured. Enable auto-memory in Claude Code settings → Memory.` and stop.

---

### mem list

Read MEMORY.md. For each line matching `- [<Title>](<file>) — <description>`, read the file's frontmatter for its `type` and `context` fields.

Group by context first (work → home → family → untagged), then by type within each group.

```
MEMORY — 8 entries  ·  💼 work (5)  🏠 home (2)  👨‍👩‍👧 family (1)

💼 WORK
  USER (1)
    • user-profile.md           — Who Sudhir is, working style, preferences
  FEEDBACK (2)
    • feedback-commits.md       — Commit message rules for this project
    • feedback-working-style.md — Autonomy level, post without asking
  PROJECT (1)
    • project-portfolio-sprint.md — Active OSS contribution sprint
  REFERENCE (1)
    • reference-blog-repo.md    — shadowmodder.github.io setup notes

🏠 HOME
  PROJECT (2)
    • project-forgein-evening.md — Evening work on forgein platform
    • project-rust-learning.md   — Learning Rust in spare time

👨‍👩‍👧 FAMILY
  USER (1)
    • family-kids-context.md    — Kids' schedule and school context
```

---

### mem search \<query\>

Read all linked files. Return entries whose body contains the query as a whole word or phrase (case-insensitive). Show context tag next to each result.

```
2 matches for "commit"

[feedback-commits.md]  type: feedback  context: 💼 work
  "Commit messages must not mention Claude or AI..."

[feedback-working-style.md]  type: feedback  context: 💼 work
  "...post without asking, commit autonomously..."
```

---

### mem add \<content\> [--ctx \<name\>] [--type \<type\>]

Add a new memory entry. Flags:
- `--ctx work|home|family` — which context this belongs to. Defaults to active context.
- `--type user|feedback|project|reference` — memory type. Inferred from content if omitted.

**Step 1 — Determine context and type**

If `--ctx` not specified: read `~/.config/forgein/active-context`.
If `--type` not specified: infer from content keywords:
- Contains "prefer", "style", "always", "never", "I like" → `feedback`
- Contains project name, version, deadline, goal → `project`
- Contains URL, path, "lives in", "stored at" → `reference`
- Otherwise → `user`

**Step 2 — Write file**

Generate kebab-case slug from first 5 significant words of content. Write:

```markdown
---
name: <slug>
description: <one-line summary>
metadata:
  type: <type>
  context: <work|home|family>
---

<content body — for feedback/project: lead with rule/fact, then **Why:** and **How to apply:** lines>
```

Append to MEMORY.md: `- [Title](slug.md) — one-line hook`

Confirm:
```
✓ Saved as <slug>.md  ·  context: 💼 work  ·  type: feedback
  Run /forgein mem sync to push to cloud.
```

---

### mem prune

Read all files linked from MEMORY.md. Check each referenced artifact:
- File paths → `test -f <path>`
- GitHub PR/issue URLs → `gh pr view <number> --repo <repo> --json state`
- Function/class names → `grep -r "<name>" .`
- Deadlines where the date has clearly passed

Show one at a time:
```
[project-portfolio-sprint.md]
  References PR #32199 — still open ✓
  → OK

[reference-old-server.md]
  References path ~/servers/old-api/ — does not exist ✗
  Delete this memory? (y/n)
```

On `y`: delete file and remove line from MEMORY.md.

---

### mem audit

Check for structural issues and offer to auto-fix each:

| Check | Auto-fix |
|-------|----------|
| Duplicate slugs in MEMORY.md | Remove duplicate line |
| Link points to non-existent file | Remove broken line |
| Memory file has empty body | Flag for user review |
| MEMORY.md line over 150 chars | Truncate description |
| File missing frontmatter fields | Infer from body and write |
| File missing `context` frontmatter | Prompt user to assign one |

```
MEMORY AUDIT

✓ No duplicate slugs
✗ Broken link: user-expertise.md does not exist → remove? (y/n)
✓ All bodies non-empty
✓ All lines under 150 chars
⚠ family-kids-context.md missing context tag → assign to family? (y/n)

2 issues found.
```

---

### mem delete \<filePath\>

Remove a synced memory file from the cloud. Requires authentication.

```bash
TOKEN=$(cat ~/.config/forgein/token 2>/dev/null)
PROJECT_PATH=$(echo "$(pwd)" | sed 's|[^a-zA-Z0-9]|-|g')
curl -sf -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "https://api.forgein.ai/api/memory/files/<filePath>?projectPath=$PROJECT_PATH"
```

On success: `✓ <filePath> removed from cloud.`
On 404: `✗ File not found in cloud — may already be deleted.`
On 401: `✗ Token invalid. Run /forgein auth.`

This only removes the file from the cloud manifest. The local file is untouched. To also remove it locally and from MEMORY.md, delete the file and remove its line from MEMORY.md manually.

---

### mem sync

Sync memory files for the current project across machines and contexts.

**Step 1 — Authenticate**

```bash
cat ~/.config/forgein/token 2>/dev/null
```

If missing:
```
✗ Not authenticated. Run /forgein auth first.
```
Stop.

Verify token:
```bash
curl -sf -H "Authorization: Bearer $(cat ~/.config/forgein/token)" https://api.forgein.ai/api/auth/user
```

On 401 or network error: `✗ Token invalid. Run /forgein auth.` and stop.

**Step 2 — Detect project**

Claude Code normalizes the working directory path by replacing ALL non-alphanumeric characters with hyphens. Walk up from `pwd` to find the closest matching project directory:

```bash
SEARCH_DIR=$(pwd)
MEMORY_DIR=""
PROJECT_PATH=""
while [ "$SEARCH_DIR" != "/" ]; do
  PROJECT_PATH=$(echo "$SEARCH_DIR" | sed 's|[^a-zA-Z0-9]|-|g')
  CANDIDATE="$HOME/.claude/projects/$PROJECT_PATH/memory"
  if [ -d "$CANDIDATE" ]; then
    MEMORY_DIR="$CANDIDATE"
    break
  fi
  SEARCH_DIR=$(dirname "$SEARCH_DIR")
done
```

If `MEMORY_DIR` is still empty: `✗ No memory directory for this project. Open this directory in Claude Code first.` and stop.

**Step 3 — Check `--team` flag**

If `--team` was passed:
- Re-fetch user if not already done; if plan is `free`: print `✗ Team sync requires forgein Pro. app.forgein.ai/upgrade` and stop.
- Sync only files in `team-shared/` subfolder to the org's shared space. Requires org membership (checked via API). Skip to Step 5b.

**Step 4 — Get cloud manifest**

```bash
TOKEN=$(cat ~/.config/forgein/token)
curl -sf -H "Authorization: Bearer $TOKEN" \
  "https://api.forgein.ai/api/memory/files?projectPath=$PROJECT_PATH"
```

Parse JSON array of `{filePath, sha256, updatedAt}`. 401 → `✗ Token invalid. Run /forgein auth.` and stop.

**Step 5a — Compute local state and diff**

```bash
find "$MEMORY_DIR" -name "*.md" -not -name "MEMORY.md" -type f
```

For each: `shasum -a 256 "$file"` (macOS) or `sha256sum "$file"` (Linux).

Build:
- **push**: local files with sha256 different from cloud (or absent from cloud)
- **pull**: cloud files absent locally or with different sha256

If both empty: `✓ Memory already in sync (N files).` and stop.

Print plan:
```
MEMORY SYNC — 💼 work  ·  project: acme-platform

  PUSH (2)
    user-profile.md          local → cloud
    feedback-commits.md      local → cloud  (updated)

  PULL (1)
    reference-blog.md        cloud → local  (not on this machine)

Proceed? [Y/n]
```

**Step 5b — Team sync (`--team`)**

```bash
curl -sf -H "Authorization: Bearer $TOKEN" \
  "https://api.forgein.ai/api/memory/team-projects"
```

Diff only files under `team-shared/`. Print plan with `[team-shared]` label on each row. Confirm and execute same as Step 6.

**Step 6 — Execute**

Push:
```bash
TOKEN=$(cat ~/.config/forgein/token)
CONTENT=$(cat "$MEMORY_DIR/$filePath")
curl -sf -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"projectPath\": \"$PROJECT_PATH\", \"filePath\": \"$filePath\", \"content\": $(echo "$CONTENT" | jq -Rs .)}" \
  https://api.forgein.ai/api/memory/files
echo "  ↑ $filePath"
```

Pull:
```bash
RESULT=$(curl -sf -H "Authorization: Bearer $TOKEN" \
  "https://api.forgein.ai/api/memory/files/$filePath?projectPath=$PROJECT_PATH")
echo "$RESULT" | jq -r '.content' > "$MEMORY_DIR/$filePath"
echo "  ↓ $filePath"
```

Print: `✓ Sync complete — 2 pushed, 1 pulled.`

---

## sec

Security scan on your code. Free tier: staged git changes only. Pro tier: any path.

**Step 1 — Get code to scan**

If no argument or argument is `staged`:
```bash
git diff --staged 2>/dev/null
```

If output is empty: print
```
No staged changes found.
Enter a path to scan (file or directory), or press Enter to cancel:
```
Wait for input. If user gives a path → check plan (path scan is PRO):

```bash
cat ~/.config/forgein/token 2>/dev/null
```

If plan is `free`:
```
✗ Deep path scan requires forgein Pro.
  Free tier scans staged git changes only.
  Upgrade: app.forgein.ai/upgrade
```
Stop.

If plan is `pro` and path given: read that path (`find <path> -type f` if directory, then Read each file).

If argument is a path (not `staged`):
- Free: print pro-required message above and stop.
- Pro: read the path.

**Step 2 — Scan for vulnerability classes**

Check all 8 classes. Record: severity, location (file:line), class name, code snippet ≤80 chars, one-sentence attack scenario.

| Class | Severity | Pattern |
|-------|----------|---------|
| Secrets | critical | Vars named `key/secret/password/token/api_key/auth` with long alphanumeric, base64, or prefixed (`sk-`, `ghp-`, `xox-`, `AKIA`) values |
| Command injection | critical | User input in `subprocess`/`os.system`/`exec`/`eval` with `shell=True` or string concatenation |
| SQL injection | high | User input concatenated into SQL via f-string, `.format()`, or `+` instead of parameterized queries |
| Path traversal | high | User-controlled strings in `open()`/`os.path.join` without `realpath` normalization |
| SSRF | high | User-controlled URLs in `requests`/`httpx`/`fetch`/`urllib` without allowlist |
| XSS | medium | User input in `innerHTML`/`dangerouslySetInnerHTML`/unescaped template output |
| Insecure deserialization | medium | `pickle.loads`, `yaml.load` (not `safe_load`), `eval()` on external data |
| Hardcoded credentials | medium | Literal passwords/tokens in source, config, or test fixtures |

**Step 3 — Output**

Sort: critical → high → medium → low.

```
SECURITY FINDINGS — 1 critical, 1 high, 0 medium, 0 low

CRITICAL  src/auth/login.py:47   Command injection
  snippet: subprocess.run(f"convert {filename}", shell=True)
  attack:  filename="x; rm -rf /" executes arbitrary shell commands

HIGH      config/db.py:12   SQL injection
  snippet: f"SELECT * FROM users WHERE id={user_id}"
  attack:  user_id="1 OR 1=1" dumps entire users table

Summary: 1 critical, 1 high, 0 medium, 0 low
```

If no findings: `✓ Clean — no issues found in the scanned code.`

Always end with:
> Note: heuristic scan only — run Semgrep, Bandit, or CodeQL for production audits.

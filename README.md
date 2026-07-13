# Forgein

**The portable context layer for AI coding assistants.** Write your context once — every tool you use gets it.

→ [forgein.ai](https://forgein.ai) · [app.forgein.ai](https://app.forgein.ai) · [github.com/forgeinai/forgein](https://github.com/forgeinai/forgein)

---

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/forgeinai/forgein/main/install.sh | bash
```

One file drops into `~/.claude/commands/forgein.md`. Open a new Claude Code session — `/forgein` appears immediately. No server dependency. Works offline.

---

## What it does

**Local (zero server, MIT, free forever):**
- `/forgein mem` — manage memory files across Claude sessions
- `/forgein sec` — heuristic security scan on staged git changes
- `/forgein optimize` — discover and install skills matched to your workflow

**Cloud sync (free account at [app.forgein.ai](https://app.forgein.ai)):**
- `/forgein auth` — connect your local CLI to your forgein account
- `/forgein mem sync` — push your memory files to the cloud
- `/forgein export <target>` — write your context into any AI tool's native format

Once synced, every adapter below stays current automatically after every `mem sync`.

---

## Adapters — 7 tools, one context store

| Adapter | What it writes | How |
|---|---|---|
| **Claude Code** | Auto-injected via `UserPromptSubmit` hook | Always on |
| **MCP Server** | JSON-RPC 2.0 over stateless HTTP, auto-discovered at `/.well-known/mcp` | `api.forgein.ai/api/adapters/mcp` |
| **Cursor** | `.cursorrules` (or `~/.cursor/rules` globally) | `/forgein export cursor` |
| **Windsurf** | `.windsurfrules` (or Cascade global memory) | `/forgein export windsurf` |
| **GitHub Copilot** | `.github/copilot-instructions.md` | `/forgein export copilot` |
| **ChatGPT** | Custom Instructions (two-field format) | `/forgein export chatgpt` |
| **Gemini** | Gems system prompt | `/forgein export gemini` |

All 7 adapters are free. None are gated to Pro.

The MCP adapter is protocol-native — any MCP-compatible client (Claude Desktop, Cursor, Windsurf Cascade) can connect directly without the CLI.

---

## Commands

```
/forgein auth                    Authenticate CLI with your forgein account

/forgein mem                     List all memories (default)
/forgein mem list                List memories grouped by type
/forgein mem search <query>      Search memory bodies
/forgein mem add <type> <text>   Add a new memory
/forgein mem prune               Remove stale memories interactively
/forgein mem audit               Check index for structural issues
/forgein mem sync                Sync memory files with forgein cloud

/forgein export cursor           Write context to .cursorrules
/forgein export windsurf         Write context to .windsurfrules
/forgein export copilot          Write context to .github/copilot-instructions.md
/forgein export chatgpt          Format context as ChatGPT Custom Instructions
/forgein export gemini           Format context as a Gemini Gem system prompt

/forgein sec                     Security check on staged git changes
/forgein sec <path>              Security check on a file or directory

/forgein optimize                Discover and install skills for your workflow
```

---

## How it works

### Cloud sync flow

```
/forgein auth
  → checks ~/.config/forgein/token
  → if missing or stale: prompts user to create token at app.forgein.ai/tokens
  → stores token at ~/.config/forgein/token (chmod 600)

/forgein mem sync
  → computes sha256 of each local memory file
  → diffs against cloud manifest from api.forgein.ai/api/memory/files
  → pushes changed files, pulls files added from another machine
  → auto-runs /forgein export for any adapter files already present in project

/forgein export cursor
  → reads token from ~/.config/forgein/token
  → calls api.forgein.ai/api/adapters/cursor
  → writes response to .cursorrules in project root
```

### `/forgein optimize` — algorithm

```
1. FETCH REGISTRY
   gh api repos/forgeinai/forgein/contents/registry.json | base64 -d
   → fallback: WebFetch raw.githubusercontent.com

2. INVENTORY INSTALLED
   ls ~/.claude/commands/
   → set of already-installed skill names (exclude from results)

3. GATHER CONTEXT SIGNALS
   cat ~/.claude/CLAUDE.md          (global instructions)
   cat CLAUDE.md                    (project instructions)
   find ~/.claude/projects -name MEMORY.md | xargs cat  (memory index)
   git log --oneline -20            (recent commit topics)
   → single context string

4. SCORE SKILLS (word-boundary matching)
   for each skill not installed:
     score = 0
     for term in (skill.signals + skill.tags):
       if term matches as a whole word in context (case-insensitive):
         score += 1
   
   Whole-word rule: char before and after match must be non-alphanumeric.
   "pr" does NOT match "sprint". "git" does NOT match "digital".

5. RANK + RECOMMEND
   sort by score desc → take top 5
   show: skill name, score, description, matched terms

6. INTERACTIVE INSTALL
   default: Enter = install all
   or pick by number: "1,3"
   or "none" to cancel
   
   install: gh api .../contents/<file> | base64 -d → write to ~/.claude/commands/
```

---

### `/forgein mem` — architecture

Claude Code's auto-memory system stores context across sessions as markdown files. `MEMORY.md` is the index; each linked file is one memory with frontmatter:

```markdown
---
name: feedback-commits
description: Commit message rules — no Claude/AI attribution
metadata:
  type: feedback        # user | feedback | project | reference
---

Commit messages must not mention Claude, AI, or any collaboration.

**Why:** commits must appear as the user's own organic work.
**How to apply:** never add Co-Authored-By lines. Write messages in first person.
```

**Operations:**

| Subcommand | What it does |
|-----------|-------------|
| `list` | Read MEMORY.md, fetch each file's type from frontmatter, group and display |
| `search <q>` | Read all linked files, return whole-word matches in body |
| `add <type> <content>` | Write new file with frontmatter, append pointer to MEMORY.md |
| `prune` | For each memory, verify referenced artifacts still exist (PRs, files, functions). Flag stale. |
| `audit` | Structural checks: broken links, duplicates, missing frontmatter, lines > 150 chars |
| `sync` | Push/pull from forgein cloud. Requires `/forgein auth` first. |

Memory types:
- **user** — who the user is, expertise, working style
- **feedback** — how Claude should behave (corrections AND confirmed approaches)
- **project** — active work, goals, deadlines
- **reference** — where to find things (Linear boards, dashboards, docs)

---

### `/forgein sec` — scan algorithm

```
1. GET CODE
   git diff --staged          (default)
   <path>                     (if argument given)
   
   If staged diff is empty → ask user for a path. Never silently scan.

2. SCAN 8 VULNERABILITY CLASSES
   
   Class              Severity  What to look for
   ─────────────────────────────────────────────────────────────────
   Secrets            critical  Long string literals in key/secret/token vars
   Command injection  critical  User input in subprocess/eval/shell=True
   SQL injection      high      User input concatenated into SQL strings
   Path traversal     high      User input in open()/join() without realpath
   SSRF               high      User-controlled URLs fetched server-side
   XSS                medium    User input in innerHTML/unescaped templates
   Insecure deser.    medium    pickle.loads / yaml.load / eval on external data
   Hardcoded creds    medium    Literal passwords/tokens in source or config

3. OUTPUT
   Sort: critical → high → medium → low
   Each finding: severity, file:line, class, snippet (≤80 chars), attack scenario
   
   If no findings: "✓ Clean"
   Always: remind user to run a real SAST tool for production
```

---

## Registry format

`registry.json` is the skill index. Each entry:

```json
{
  "id": "commit",
  "command": "commit",
  "name": "Smart Commit",
  "description": "Generates conventional commit messages from staged changes.",
  "tags": ["git", "productivity", "conventional commits"],
  "signals": ["commits", "commit messages", "changelog", "semantic versioning"],
  "author": "forgein-ai",
  "version": "1.0.0",
  "file": "skills/commit.md",
  "install_as": "commit.md"
}
```

- **`tags`** — broad categories (used in scoring)
- **`signals`** — specific phrases that indicate a user needs this skill (used in scoring)
- **`file`** — path to the skill `.md` file in this repo
- **`install_as`** — filename written to `~/.claude/commands/`

**Scoring weight:** `tags` and `signals` are treated equally. A skill with 3 tag matches and 2 signal matches scores 5.

---

## Skills in the registry

| Skill | Command | Description |
|-------|---------|-------------|
| Forgein | `/forgein` | Auth, memory sync, multi-tool export, security check, skill optimizer |
| Vibe Sec | `/sec` | Standalone security check |
| Claude Mem | `/mem` | Standalone memory manager |
| Code Review | `/review` | Bugs, perf, security, style — parallel passes |
| Smart Commit | `/commit` | Conventional commit messages from staged diff |
| Standup | `/standup` | Daily standup from git log + open PRs |
| PR Monitor | `/pr-monitor` | CI status across all open PRs |
| Test Gen | `/test-gen` | Generate tests — auto-detects pytest/jest/go |
| Explain Codebase | `/explain` | Architecture map for new contributors |
| Changelog | `/changelog` | CHANGELOG.md from git log between tags |

---

## Submit a skill

1. Add your `.md` file to `skills/`
2. Add an entry to `registry.json`
3. Open a PR

**Skill file format:** A markdown file with a YAML frontmatter `description` field. The body is instructions to Claude — what to do, what tools to use, what output to produce. Written in second person imperative. See any existing skill file as a template.

Good signals are specific phrases that appear in someone's CLAUDE.md or git log when they *need* your skill. Bad signals are things everyone has ("code", "files", "project").

---

## How Claude skills work

Skills are markdown files in `~/.claude/commands/`. When you type `/forgein`, Claude reads `forgein.md` and follows its instructions — using your local shell, files, and APIs. No server. No telemetry. Runs entirely in your Claude Code session.

The skill file is the product. It tells Claude: what the user wants, what tools to call, what output format to use.

---

## Free vs. Pro

| | Free | Pro ($4.99/mo) | Team |
|---|---|---|---|
| All 7 adapters + MCP server | ✓ | ✓ | ✓ |
| Memory projects | 3 | Unlimited | Unlimited |
| Files per project | 100 | Unlimited | Unlimited |
| Contexts (Work/Home/Family) | Work only | All 3 | All 3 |
| Private skills | — | ✓ | ✓ |
| Team memory sharing | — | — | ✓ |
| Org baseline templates | — | — | ✓ |
| AI adoption analytics | — | — | ✓ |
| API tokens | 1 | 5 | Unlimited |

The portability mechanism — every adapter, the full MCP server — is free. Pro gates scale (more projects, more contexts). Team gates organizational features.

---

## License

MIT — CLI and skill files in this repo ([forgeinai/forgein](https://github.com/forgeinai/forgein))

The hosted sync service at [app.forgein.ai](https://app.forgein.ai) is separate and not MIT-licensed.

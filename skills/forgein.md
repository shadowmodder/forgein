---
description: Forgein — Claude productivity toolkit. Subcommands: optimize (discover and install skills), mem (manage memory), sec (security check).
---

Parse `$ARGUMENTS` — first word is the subcommand. Route to the correct section. If no argument or unrecognized subcommand: print the usage table below and stop.

```
FORGEIN — Claude productivity toolkit

  /forgein optimize        Discover and install skills that fit your workflow
  /forgein mem             List all memories (default)
  /forgein mem list        List memories grouped by type
  /forgein mem search <q>  Search memory bodies for a keyword
  /forgein mem add <type> <content>  Add a new memory
  /forgein mem prune       Remove stale memories interactively
  /forgein mem audit       Check index for broken links and duplicates
  /forgein sec             Security check on staged git changes
  /forgein sec <path>      Security check on a specific file or directory
```

---

## optimize

### Algorithm

**Step 1 — Fetch registry**

Try in order, use first that succeeds:
```bash
gh api repos/shadowmodder/forgein/contents/registry.json --jq '.content' | base64 -d
```
Fallback: WebFetch `https://raw.githubusercontent.com/shadowmodder/forgein/main/registry.json`

Parse the JSON. Extract the `skills` array.

**Step 2 — Inventory installed skills**

```bash
ls ~/.claude/commands/ 2>/dev/null
```

Strip `.md` extensions. Store as a set of installed command names. Exclude these from recommendations.

**Step 3 — Gather context signals**

Run all four in one Bash call:
```bash
cat ~/.claude/CLAUDE.md 2>/dev/null
cat CLAUDE.md 2>/dev/null
find ~/.claude/projects -name "MEMORY.md" 2>/dev/null | head -3 | xargs cat 2>/dev/null
git log --oneline -20 2>/dev/null
```

Concatenate all output into one context string.

**Step 4 — Score skills (word-boundary matching)**

For each skill not already installed, compute:

```
score = 0
for term in (skill.signals + skill.tags):
    if whole-word match of term in context (case-insensitive):
        score += 1
```

A **whole-word match** means the term is surrounded by word boundaries — spaces, punctuation, newlines, or string start/end. The term `"pr"` must NOT match inside `"sprint"` or `"improve"`. Use this test: the character before and after the match (if it exists) must be non-alphanumeric.

If context is empty (no CLAUDE.md, no memory, no git log): assign all skills a score of 0 and rank by registry order.

**Step 5 — Present top 5**

Sort non-installed skills by score descending. Take top 5. Display:

```
RECOMMENDED SKILLS FOR YOUR WORKFLOW

#  Skill        Score  What it does                        Why it fits you
1  pr-monitor     6    CI status across all open PRs       Matched: "open prs", "github actions", "ci"
2  code-review    4    Multi-dimensional diff review       Matched: "pull request", "review", "pr"
3  commit         3    Smart conventional commit messages  Matched: "commits", "git"
4  vibe-sec       2    Security check on staged changes    Matched: "security", "auth"
5  standup        1    Daily standup from git + PRs        Matched: "sprint"
```

Show the matched terms in "Why it fits you" — be specific.

**Step 6 — Install**

Ask: **"Install all 5? [Enter = yes / numbers e.g. 1,3 / none]:"**

Default (Enter with no input) = install all recommendations.

For each selected skill, fetch and install:
```bash
gh api repos/shadowmodder/forgein/contents/<file> --jq '.content' | base64 -d
```
Write output to `~/.claude/commands/<install_as>`.

Confirm each install: `✓ /commit installed — type /commit to use it.`

If user enters `none` or `n`: exit without installing.

---

## mem

Parse the next word as the mem subcommand. Default (no word given): run `list`.

**Locate memory directory:**
```bash
find ~/.claude/projects -name "MEMORY.md" 2>/dev/null | head -1
```
Use the parent directory of the first result as the memory directory. If none found: print "Memory not configured. Enable auto-memory in Claude Code settings → Memory." and stop.

---

### mem list

Read MEMORY.md. For each line matching `- [<Title>](<file>) — <description>`, read the file's frontmatter to get its `type` field.

Group entries by type in this order: `user` → `feedback` → `project` → `reference`.

```
MEMORY — 5 entries  (updated 2026-07-05)

USER (1)
  • user-profile.md       — Who Sudhir is, working style, preferences

FEEDBACK (2)
  • feedback-commits.md   — Commit message rules for this project
  • feedback-working-style.md — Autonomy level, post without asking

PROJECT (1)
  • project-portfolio-sprint.md — Active OSS contribution sprint

REFERENCE (1)
  • reference-blog-repo.md — shadowmodder.github.io setup notes
```

---

### mem search \<query\>

Read MEMORY.md to get all linked filenames. Read each linked file in full. Return entries whose body contains the query as a whole word or phrase (case-insensitive).

```
2 matches for "commit"

[feedback-commits.md]  type: feedback
  "Commit messages must not mention Claude or AI..."

[feedback-working-style.md]  type: feedback
  "...post without asking, commit autonomously..."
```

---

### mem add \<type\> \<content\>

Valid types: `user`, `feedback`, `project`, `reference`.

1. Generate a kebab-case slug from the first 5 significant words of content.
2. Write a new file with this frontmatter structure:
```markdown
---
name: <slug>
description: <one-line summary>
metadata:
  type: <type>
---

<content body — for feedback/project: lead with the rule/fact, then **Why:** and **How to apply:** lines>
```
3. Append to MEMORY.md: `- [Title](slug.md) — one-line hook`
4. Confirm: `✓ Saved as <slug>.md and added to index.`

---

### mem prune

Read all files linked from MEMORY.md. For each, check whether the artifacts it references still exist:
- File paths → check with `test -f <path>`
- GitHub PR/issue URLs → check with `gh pr view <number> --repo <repo> --json state`
- Function/class names → `grep -r "<name>" .` from the project root
- Deadlines or dated facts where the date has clearly passed

Flag entries where artifacts are gone or facts are stale. Show one at a time:

```
[project-portfolio-sprint.md]
  References PR #32199 — still open ✓
  References branch fix/reasoning-tokens — still exists ✓
  → OK

[reference-blog-repo.md]
  References branch "master" — exists ✓
  → OK
```

For flagged entries ask: `Delete this memory? (y/n)`
On `y`: delete the file and remove the line from MEMORY.md.

---

### mem audit

Check for and report each structural issue. For each issue, offer to auto-fix:

| Check | Auto-fix |
|-------|----------|
| Duplicate slugs in MEMORY.md | Remove the duplicate line |
| Link points to non-existent file | Remove the broken line |
| Memory file has empty body | Flag for user review |
| MEMORY.md line over 150 chars | Truncate description to fit |
| File missing frontmatter fields | Infer from body and write |

Report format:
```
MEMORY AUDIT

✓ No duplicate slugs
✗ Broken link: user-expertise.md does not exist → remove? (y/n)
✓ All bodies non-empty
✓ All lines under 150 chars
✓ All frontmatter complete

1 issue found.
```

---

## sec

### Algorithm

**Step 1 — Get code to scan**

If no second argument or argument is `staged`:
```bash
git diff --staged 2>/dev/null
```
If the output is **empty**, do NOT silently scan something else. Instead print:
```
No staged changes found.
Enter a path to scan (file or directory), or press Enter to cancel:
```
Wait for input. If user gives a path: read that path (use `find <path> -type f` if directory, then Read each file). If user presses Enter with no input: exit cleanly.

If a path argument was given: read that path directly.

**Step 2 — Scan for vulnerability classes**

Check all 8 classes. For each finding, record: severity, location (file:line where available), class name, code snippet ≤ 80 chars, and a one-sentence attack scenario.

| Class | Severity | Pattern |
|-------|----------|---------|
| Secrets | critical | String literals in vars named `key/secret/password/token/api_key/auth` — long alphanumeric, base64, or prefixed (`sk-`, `ghp-`, `xox-`, `AKIA`) |
| Command injection | critical | User input in `subprocess`/`os.system`/`exec`/`eval` with `shell=True` or string concatenation |
| SQL injection | high | User input concatenated into SQL via f-string, `.format()`, or `+` instead of parameterized queries |
| Path traversal | high | User-controlled strings in `open()`/`os.path.join` without `realpath` normalization |
| SSRF | high | User-controlled URLs in `requests`/`httpx`/`fetch`/`urllib` without allowlist |
| XSS | medium | User input in `innerHTML`/`dangerouslySetInnerHTML`/unescaped template output |
| Insecure deserialization | medium | `pickle.loads`, `yaml.load` (not `safe_load`), `eval()` on external data |
| Hardcoded credentials | medium | Literal passwords/tokens in source, config, or test fixtures |

**Step 3 — Output**

Sort findings: critical → high → medium → low.

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
> Note: heuristic check only — run Semgrep, Bandit, or CodeQL for production audits.

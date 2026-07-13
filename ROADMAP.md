# Forgein Roadmap

This file covers what's shipped, what's in progress, and what's coming. It's updated with each release. For the full changelog, see [forgein.ai/changelog](https://forgein.ai/changelog).

---

## Shipped

### CLI + Skills
- [x] `/forgein mem` — memory list, search, add, prune, audit
- [x] `/forgein sec` — heuristic security scan on staged diffs (8 vulnerability classes)
- [x] `/forgein optimize` — word-boundary skill scoring + interactive install
- [x] `/forgein auth` — token-based auth against forgein cloud
- [x] `/forgein mem sync` — push/pull memory files to cloud, auto-re-export adapter files
- [x] `/forgein export <target>` — write context into any tool's native format

### Adapters (all free)
- [x] Claude Code — auto-injected via `UserPromptSubmit` hook
- [x] MCP Server — JSON-RPC 2.0, stateless HTTP, auto-discovery at `/.well-known/mcp`
- [x] Cursor — writes `.cursorrules`
- [x] Windsurf — writes `.windsurfrules`
- [x] GitHub Copilot — writes `.github/copilot-instructions.md`
- [x] ChatGPT — formats Custom Instructions (two-field)
- [x] Gemini — formats Gems system prompt

### Platform
- [x] Work / Home / Family contexts — memory that doesn't cross between work and personal sessions
- [x] Team memory sharing — org members share a common project memory
- [x] Org baseline templates — admin sets base context that all members inherit
- [x] Webhooks — event delivery for memory syncs, with retry queue
- [x] AI adoption analytics — per-member usage breakdown across all AI tools, audit log
- [x] `@forgein/adapter-sdk` on npm — build your own adapter or integration
- [x] Context inheritance — adapter output layers org baseline → team → project → personal
- [x] Org context templates — admin CRUD dashboard for managing the org baseline context layer
- [x] Browser extension — Chrome MV3, auto-injects context into ChatGPT and Gemini on click

### Skills in the registry (10)
`/forgein`, `/sec`, `/mem`, `/review`, `/commit`, `/standup`, `/pr-monitor`, `/test-gen`, `/explain`, `/changelog`

---

## In progress

- [ ] **Public GitHub Projects board** — mirror of this file with community-voted priorities.

---

## Planned

### Near-term (next 60 days)
- [x] **Browser extension** — Chrome MV3 extension with floating button on chatgpt.com and gemini.google.com; injects your context on click
- [x] **Policy / constraint layer** — org owners set block rules (redact regex matches) and require rules (prepend text) that apply to every member's adapter output
- [ ] **Context versioning** — who changed which org context, when, with rollback
- [ ] **Compliance export** — full audit trail as CSV/JSON (SOC2 evidence)
- [ ] **SAML / SSO** — Okta, Google Workspace, Azure AD (schema exists; hardening in progress)
- [ ] **VS Code / JetBrains adapter** — native IDE context injection without the CLI

### Longer-term
- [ ] **Context pipeline API** — pipe internal docs, runbooks, Linear tickets into forgein automatically
- [ ] **Confluence / Notion / Linear integration** — pull current project state, keep context fresh without manual sync
- [ ] **Context health monitoring** — detect stale or policy-violating org context, alert the platform admin
- [ ] **Template marketplace** — publish and discover org-level context templates

---

## How to contribute

**Submit a skill:** add your `.md` to `skills/`, add an entry to `registry.json`, open a PR. See [README.md](README.md) for the file format and signal guidelines.

**Propose a feature:** open a GitHub Discussion in this repo. Accepted proposals move to "In progress" here.

**Build an adapter:** `npm install @forgein/adapter-sdk` and follow the [adapter SDK docs](https://forgein.ai/docs/api).

---

## What won't be here

Features that are org-internal, compliance-sensitive (exact SSO implementation details), or that haven't been committed to a timeline. This roadmap is honest about what's speculative.

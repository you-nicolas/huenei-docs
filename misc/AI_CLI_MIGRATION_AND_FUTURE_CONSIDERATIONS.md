# AI CLI Migration: Gemini → Cursor, and Considerations for Claude

**Author:** Nicolás (Huenei IT Services — personal tooling notes)  
**Date:** June 2026  
**Context:** Work notes archive (`huenei-docs`) before job change  
**Status:** Cursor migration complete (Phases 0–5). Next workplace: **Claude**.

---

## Credits and lineage

The agent operating system documented here **originates from [gemini-cli-config](https://github.com/ksprashu/gemini-cli-config)** by **Prashanth Subrahmanyam** (`ksprashu`). That repository defined the core protocols (`AGENTS.md`), slash-command palettes (`commands/*.toml`), MCP setup, and the `// GO GEMINI` workflow for Google Gemini CLI.

My work (June 2026) was a **fork and migration**, not a greenfield design:

- Fixed corrupted sections in `AGENTS.md`
- Migrated Gemini CLI wiring to **Cursor CLI** (rules, skills, hooks, MCP)
- Documented the process in `docs/ADAPTATION_ANALYSIS.md`

If you use or extend this system, credit the original repo and author. Upstream: https://github.com/ksprashu/gemini-cli-config

Local fork / migration workspace: `~/gemini-cli-config`

---

## 1. Executive summary

At Huenei I adopted and extended Prashanth's **gemini-cli-config** into a personal agent operating system. Gemini CLI later became impractical to use; the same ideas were migrated to **Cursor CLI** in five phases.

The **durable asset is the markdown**, not the tool wiring:

- How to think (deconstruct → plan → execute → reflect)
- How to teach (explain why, grounded debriefs)
- How to use git (`repo-save` / `repo-sync` inner/outer loop)
- How to scaffold docs (`setup-context`, `setup-readme`, `setup-design`)
- Protocol interrupts (`trigger-freeze`, `trigger-architect`, etc.)

Roughly **80% transfers to Claude** with a new install path and naming. The cognitive framework is tool-agnostic; hooks, rule formats, and slash-command packaging differ per product.

---

## 2. Background: what existed

### 2.1 Original stack (Gemini CLI — ksprashu/gemini-cli-config)

| Piece | Location | Role |
|-------|----------|------|
| Core protocol | `AGENTS.md` | Handshake, cognitive framework, modes, git/MCP mandates |
| Preferences | `GEMINI.md` | Stack prefs, TDD, SDK defaults |
| Slash commands | `commands/**/*.toml` | `/repo:save`, `/trigger:elite`, `/setup:gemini`, … |
| Runtime | `~/.gemini/settings.json` | MCP (context7, sequential-thinking) |
| Session file | `.gemini/session.md` | Operational state persistence |
| Boot directive | `// GO GEMINI` | Force protocol reload |

### 2.2 Why migrate

- Gemini CLI is **no longer viable** as a daily driver
- **Cursor CLI** replaced it on personal machine
- New job will use **Claude** — need portable protocols, not Cursor-only lock-in

### 2.3 Huenei vs personal scope

| Item | Scope |
|------|--------|
| Client notes in `huenei-docs` (this repo) | Huenei work archive |
| `gemini-cli-config` fork | Personal tooling |
| `~/.cursor/` install | Personal machine |
| Vaultwarden / Keycloak runbooks | Huenei client work — stays in `devops/` |

Do **not** copy Huenei client secrets or proprietary docs into personal Claude projects.

---

## 3. Cursor migration (completed)

Full log: `~/gemini-cli-config/docs/ADAPTATION_ANALYSIS.md`

### Phase summary

| Phase | Deliverable |
|-------|-------------|
| **0** | Fixed `AGENTS.md` corruption; archived Gemini originals |
| **1** | **15 Cursor rules** → `~/.cursor/rules/` |
| **2** | **10 skills** (from TOML commands) → `~/.cursor/skills/` |
| **3** | MCP + CLI config templates |
| **4** | Session hooks + **architect** subagent |
| **5** | README, `MIGRATION.md`, rebrand |

### Install (user-global)

```bash
cd ~/gemini-cli-config
./scripts/install.sh --with-optional-mcp
export CONTEXT7_API_KEY="..."
agent --approve-mcps
```

| Component | Installed to |
|-----------|--------------|
| Rules | `~/.cursor/rules/*.mdc` |
| Skills | `~/.cursor/skills/<name>/SKILL.md` |
| MCP | `~/.cursor/mcp.json` |
| Hooks | `~/.cursor/hooks.json` |
| Subagents | `~/.cursor/agents/architect.md` |

### Mapping (Gemini → Cursor)

| Gemini (original) | Cursor (migration) |
|-------------------|-------------------|
| `~/.gemini/AGENTS.md` | `AGENTS.md` + `~/.cursor/rules/` |
| `~/.gemini/GEMINI.md` | `PREFERENCES.md` |
| `commands/*.toml` | `skills/*/SKILL.md` |
| `settings.json` | `config/mcp.json` → `~/.cursor/mcp.json` |
| `.gemini/session.md` | `.cursor/session.md` |
| `// GO GEMINI` | `// GO CURSOR` |

### Skills (invoke via `/` in Cursor CLI)

| Skill | Original command | Purpose |
|-------|------------------|---------|
| `repo-save` | `/repo:save` | Atomic local commits |
| `repo-sync` | `/repo:sync` | Upstream/origin sync |
| `setup-context` | `/setup:gemini` | Project `AGENTS.md` |
| `setup-readme` | `/setup:readme` | README refresh |
| `setup-design` | `/setup:design` | `docs/design.md` |
| `trigger-*` | `/trigger:*` | Protocol interrupts |

---

## 4. Expected workflow (Cursor CLI — now)

### Daily loop

1. `cd <project> && agent`
2. Hooks bootstrap protocol; resume `.cursor/session.md` if present
3. Rules apply automatically
4. Task → plan (`TodoWrite`) → execute → debrief
5. **`repo-save`** often; **`repo-sync`** when sharing
6. Ambiguous work → **`trigger-architect`** first

### Explicit handshake

```text
// GO CURSOR

<task>
```

### Session persistence

Operational STATE block in agent output → hook writes `.cursor/session.md`.

### Test checklist

| # | Test | Pass |
|---|------|------|
| 1 | `ls ~/.cursor/{rules,skills,mcp.json,hooks.json}` | Symlinks OK |
| 2 | Agent lists AGENTS.md directives | 15 rules |
| 3 | `// GO CURSOR` | Protocol Sync response |
| 4 | `repo-save` on tiny change | Conventional commit |
| 5 | OPERATIONAL STATE in reply | `.cursor/session.md` exists |
| 6 | New session asks prior goal | Context resumed |

---

## 5. Porting to Claude (next workplace)

Packaging differs; **protocol content** from ksprashu's design transfers well.

### Mapping (Cursor → Claude Code, typical)

| Cursor | Claude Code (typical) |
|--------|----------------------|
| `~/.cursor/rules/*.mdc` | `CLAUDE.md` + `.claude/commands/` |
| Skills | `.claude/commands/*.md` |
| `~/.cursor/mcp.json` | `~/.claude/settings.json` / `claude mcp` |
| Hooks | Claude Code hooks (SessionStart, Stop, …) |
| `// GO CURSOR` | `// GO CLAUDE` (suggested) |
| `.cursor/session.md` | `.claude/session.md` or project-local |

### Copy verbatim (high value)

From the original + migration repos:

1. Cognitive framework (Deconstruct → Plan → Execute → Reflect)
2. Teaching mandate
3. Git workflows (`repo-save` / `repo-sync` prompt bodies in `skills/`)
4. Mode heuristics (`#mode/*`)
5. Setup and trigger skill content
6. Stack preferences (adjust for new employer)

### Rewrite for Claude

- Tool names (`Read` / `Shell` → Claude equivalents)
- Hook script JSON schemas
- Install paths and MCP env syntax
- Always-on vs slash-command packaging

### Suggested personal repo structure (future)

```text
agent-config/                    # tool-neutral fork
├── protocols/                   # source of truth (from AGENTS.md lineage)
├── cursor/                      # current install scripts
├── claude/                      # CLAUDE.md + commands + install-claude.sh
└── archive/gemini/              # ksprashu-era + local archive
```

Credit upstream when publishing: **Prashanth Subrahmanyam**, https://github.com/ksprashu/gemini-cli-config

---

## 6. New job considerations (Claude)

### Day one

- Read employer AI, secrets, and MCP policies **before** installing personal hooks
- Start with read-only protocols; add `repo-save` only when git policy allows
- Keep **personal** and **employer** configs separate

### Keep from Huenei practice

- Postmortem / RCA discipline (`devops/POSTMORTEM_KEYCLOAK_2026-05-14.md`)
- Atomic commits and conventional messages
- Spec-before-code (`trigger-architect`)
- Verify-last-action culture (`trigger-freeze`)

### Revisit

- 100% test coverage mandate — soften to meaningful coverage
- Gemini/Google SDK defaults — replace with new stack
- Always-on "elite" prompt — keep as optional slash command only

### If only Claude web/IDE (no CLI)

- Protocols → project instructions or Project knowledge
- Skills → saved prompt templates
- Hooks/MCP may be unavailable — manual session saves

---

## 7. Repository index

| Repo | Purpose |
|------|---------|
| [ksprashu/gemini-cli-config](https://github.com/ksprashu/gemini-cli-config) | **Original** — Prashanth Subrahmanyam |
| `~/gemini-cli-config` | Local Cursor migration fork |
| `~/docs` (`huenei-docs`) | Huenei work notes (this file) |

---

## 8. Quick reference

**Cursor (now):**

```bash
cd ~/gemini-cli-config && ./scripts/install.sh
agent --approve-mcps
cd my-project && agent
```

**Claude (planned):** TBD after employer policy — port `skills/*/SKILL.md` → `.claude/commands/`, core text → `CLAUDE.md`.

| Era | Handshake |
|-----|-----------|
| Gemini (original) | `// GO GEMINI` |
| Cursor (migration) | `// GO CURSOR` |
| Claude (suggested) | `// GO CLAUDE` |

---

## 9. Changelog

| Date | Change |
|------|--------|
| 2026-06-23 | Initial doc: Cursor migration complete; Claude porting plan; credits to ksprashu/gemini-cli-config |

---

*Personal tooling notes. Not Huenei client proprietary data. Original agent OS by [Prashanth Subrahmanyam](https://github.com/ksprashu/gemini-cli-config).*

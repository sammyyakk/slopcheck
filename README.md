# slopcheck

AI vibe-coded sites are always slow. **slopcheck** is an evidence-based performance audit workflow for AI coding agents — Claude Code, Cursor, Windsurf, GitHub Copilot, Cline, and any agent that reads an `AGENTS.md`. It audits, fixes, and verifies 20 standard performance optimizations, and refuses to mark anything done without evidence.

Covers:

**Database** — indexing, connection pooling, N+1 removal, query caching, pagination
**Backend/API** — response caching, payload compression, server-side caching, load balancer, CDN
**Frontend build** — minification, code splitting, lazy loading, deferred scripts, image compression, debounced input, memoized re-renders, loading skeletons, unused dependency removal
**Measurement** — Lighthouse audit (score + top opportunities, not a raw JSON dump)

Every item follows **Detect → Fix → Verify**. Nothing is marked done without a concrete check behind it (a grep hit, a response header, an `EXPLAIN ANALYZE` plan, a bundle size delta, a Lighthouse score). Partial work is reported as partial, not a false done.

## Why

Most "optimize my site" prompts produce a checklist of claims with no evidence. slopcheck forces the agent to actually check each item against the live repo/app before claiming it's done, and re-check after fixing.

## Supported agents

| Agent | File | How it's picked up |
|---|---|---|
| [Claude Code](https://claude.com/claude-code) | `skills/slopcheck/SKILL.md` | Claude Code Skill — install into `~/.claude/skills/` or a project's `.claude/skills/` |
| Cursor | `.cursor/rules/slopcheck.mdc` | Project rule, auto-attached when the agent judges it relevant |
| Windsurf | `.windsurfrules` | Read automatically at the repo root |
| GitHub Copilot (Chat / coding agent) | `.github/copilot-instructions.md` | Read automatically at the repo root |
| Cline | `.clinerules` | Read automatically at the repo root |
| Any AGENTS.md-compatible agent (Codex CLI, Amp, Jules, etc.) | `AGENTS.md` | Standard root-level agent instructions file |

`AGENTS.md` at the repo root is the single source of truth for the full 20-item checklist. The Cursor/Windsurf/Copilot/Cline files are thin pointers that tell those agents to read it — this keeps the checklist in one place instead of drifting across five copies. The Claude Code Skill (`skills/slopcheck/SKILL.md`) is a self-contained variant with Claude Code-specific subagent delegation for larger repos.

## Install

### Claude Code

```bash
git clone https://github.com/sammyyakk/slopcheck.git
cp -r slopcheck/skills/slopcheck ~/.claude/skills/
```

Project-scoped instead of user-scoped (only active inside one repo):

```bash
cp -r slopcheck/skills/slopcheck /path/to/your/project/.claude/skills/
```

Restart Claude Code (or start a new session) so it picks up the new skill.

### Cursor, Windsurf, GitHub Copilot, Cline, or any AGENTS.md-reading agent

Drop the relevant file(s) at the root of your project:

```bash
git clone https://github.com/sammyyakk/slopcheck.git tmp-slopcheck

cp tmp-slopcheck/AGENTS.md .
cp -r tmp-slopcheck/.cursor .                 # Cursor
cp tmp-slopcheck/.windsurfrules .             # Windsurf
mkdir -p .github && cp tmp-slopcheck/.github/copilot-instructions.md .github/   # Copilot
cp tmp-slopcheck/.clinerules .                # Cline

rm -rf tmp-slopcheck
```

Only copy the files for the agent(s) you actually use — `AGENTS.md` is required in every case since the tool-specific files point to it; the rest are optional pointers.

If your project already has an `AGENTS.md`, `.cursor/rules/`, `.windsurfrules`, `.github/copilot-instructions.md`, or `.clinerules`, merge the content in rather than overwriting.

## Usage

**Claude Code**: invoke directly with `/slopcheck`, or just describe the task — it also triggers on:

- "optimize performance"
- "speed up the site/app"
- "run a perf audit"
- "clean up the slop"
- pasting a checklist like the one this covers

**Other agents**: ask them the same way ("optimize this app's performance", "run a perf audit") — the rules/instructions file tells them to read `AGENTS.md` and follow it.

Any agent following this workflow will:

1. Scope the target (repo, and a live URL if real Lighthouse/network measurements are wanted).
2. Run Detect on all 20 items.
3. Fix everything found missing.
4. Re-verify what it touched.
5. Report back the checklist with one line of evidence per item.

## Requirements

- One of the agents listed above.
- For live-URL measurements (Lighthouse, network panel, traces): a browser automation tool (`lighthouse` CLI, Playwright, or Chrome DevTools/an MCP server exposing it). Without one, the workflow falls back to static verification (grep, config inspection, migration files).

## License

MIT — see [LICENSE](LICENSE).

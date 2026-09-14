# slopcheck

Your AI wrote you a slow website. Shocking, we know.

slopcheck is a performance audit workflow for AI coding agents. Claude Code, Cursor, Windsurf, GitHub Copilot, Cline, whatever reads an `AGENTS.md`. It rips through 20 standard performance checks, fixes what's broken, and refuses to say "done" unless it can prove it.

No vibes. No "looks good to me." Receipts only.

Covers:

**Database** indexing, connection pooling, N+1 removal, query caching, pagination
**Backend/API** response caching, payload compression, server-side caching, load balancer, CDN
**Frontend build** minification, code splitting, lazy loading, deferred scripts, image compression, debounced input, memoized re-renders, loading skeletons, unused dependency removal
**Measurement** Lighthouse audit (score plus top opportunities, not a 4000 line JSON dump)

Every item runs Detect, then Fix, then Verify. Nothing gets marked done without proof: a grep hit, a response header, an `EXPLAIN ANALYZE` plan, a bundle size drop, a Lighthouse number going up. Half-finished work gets reported as half-finished. No fake checkmarks.

## Why this exists

"Optimize my site" prompts usually get you a checklist full of claims and zero evidence. slopcheck makes the agent actually check, actually fix, then check again. If it can't prove it, it doesn't get to say it's done.

## Works with

| Agent | File | How it loads |
|---|---|---|
| [Claude Code](https://claude.com/claude-code) | `skills/slopcheck/SKILL.md` | Claude Code Skill. Drop into `~/.claude/skills/` or a project's `.claude/skills/` |
| Cursor | `.cursor/rules/slopcheck.mdc` | Project rule, auto-attaches when relevant |
| Windsurf | `.windsurfrules` | Read automatically at repo root |
| GitHub Copilot (Chat / coding agent) | `.github/copilot-instructions.md` | Read automatically at repo root |
| Cline | `.clinerules` | Read automatically at repo root |
| Google Antigravity | `AGENTS.md` | Read automatically at repo root, no extra file needed |
| Anything else that reads AGENTS.md (Codex CLI, Amp, Jules, etc) | `AGENTS.md` | Standard root level agent instructions file |

`AGENTS.md` is the one true source for the full checklist. The Cursor, Windsurf, Copilot, and Cline files are just pointers telling those agents to go read it, so the checklist lives in one place instead of five slightly different copies. The Claude Code Skill (`skills/slopcheck/SKILL.md`) is its own self-contained version with subagent delegation baked in for bigger repos.

## Install

### Claude Code

```bash
git clone https://github.com/sammyyakk/slopcheck.git
cp -r slopcheck/skills/slopcheck ~/.claude/skills/
```

Project only, not user wide:

```bash
cp -r slopcheck/skills/slopcheck /path/to/your/project/.claude/skills/
```

Restart Claude Code (or start a fresh session) so it picks up the skill.

### Google Antigravity

Antigravity reads `AGENTS.md` straight from the repo root automatically, no extra file, no config. Just drop it in:

```bash
git clone https://github.com/sammyyakk/slopcheck.git tmp-slopcheck
cp tmp-slopcheck/AGENTS.md .
rm -rf tmp-slopcheck
```

Open the project in Antigravity and ask it to run a perf audit. It picks up `AGENTS.md` for every agent in the workspace on its own.

One catch: Antigravity caps individual rules files at 12,000 characters, and this `AGENTS.md` runs a bit over that. If it gets truncated on your end, split it: keep the Workflow and Report format sections in `AGENTS.md`, and move the 20 item checklist into `.agent/rules/slopcheck.md`, which Antigravity also reads as a workspace rule.

### Cursor, Windsurf, Copilot, Cline, or anything reading AGENTS.md

Grab what you need from the repo root:

```bash
git clone https://github.com/sammyyakk/slopcheck.git tmp-slopcheck

cp tmp-slopcheck/AGENTS.md .
cp -r tmp-slopcheck/.cursor .                 # Cursor
cp tmp-slopcheck/.windsurfrules .             # Windsurf
mkdir -p .github && cp tmp-slopcheck/.github/copilot-instructions.md .github/   # Copilot
cp tmp-slopcheck/.clinerules .                # Cline

rm -rf tmp-slopcheck
```

Only grab the files for the agent(s) you actually run. `AGENTS.md` is required no matter what, since every other file just points at it.

Already have an `AGENTS.md`, `.cursor/rules/`, `.windsurfrules`, `.github/copilot-instructions.md`, or `.clinerules`? Merge the content in, don't nuke what's there.

## Usage

**Claude Code**: run `/slopcheck`, or just ask. It also fires on:

- "optimize performance"
- "speed up the site/app"
- "run a perf audit"
- "clean up the slop"
- pasting a checklist like the one this covers

**Everyone else**: ask the same way ("optimize this app's performance", "run a perf audit"). The rules file points the agent at `AGENTS.md` and it takes it from there.

Any agent running this will:

1. Scope the target. Repo, and a live URL if you want real Lighthouse or network numbers.
2. Run Detect on all 20 items.
3. Fix whatever's missing.
4. Re-verify everything it touched.
5. Hand you the checklist back with one line of proof per item.

## Requirements

- One of the agents above.
- For live URL measurements (Lighthouse, network panel, traces): a browser automation tool, `lighthouse` CLI, Playwright, or Chrome DevTools / an MCP server exposing it. No browser tooling, no problem, it falls back to static checks (grep, config, migration files).

## License

MIT. See [LICENSE](LICENSE).

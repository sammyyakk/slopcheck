# web-perf-audit

A [Claude Code](https://claude.com/claude-code) Skill that audits, fixes, and verifies web/webapp performance across 20 standard optimizations — and refuses to mark anything done without evidence.

Covers:

**Database** — indexing, connection pooling, N+1 removal, query caching, pagination
**Backend/API** — response caching, payload compression, server-side caching, load balancer, CDN
**Frontend build** — minification, code splitting, lazy loading, deferred scripts, image compression, debounced input, memoized re-renders, loading skeletons, unused dependency removal
**Measurement** — Lighthouse audit (score + top opportunities, not a raw JSON dump)

Every item follows **Detect → Fix → Verify**. Nothing is marked ✅ without a concrete check behind it (a grep hit, a response header, an `EXPLAIN ANALYZE` plan, a bundle size delta, a Lighthouse score). Partial work gets ⚠️, not a false ✅.

## Why

Most "optimize my site" prompts produce a checklist of claims with no evidence. This skill forces the model to actually check each item against the live repo/app before claiming it's done, and re-check after fixing.

## Install

Copy the skill folder into your Claude Code skills directory:

```bash
git clone https://github.com/sammyyakk/web-perf-audit-skill.git
cp -r web-perf-audit-skill/skills/web-perf-audit ~/.claude/skills/
```

Project-scoped instead of user-scoped (only active inside one repo):

```bash
cp -r web-perf-audit-skill/skills/web-perf-audit /path/to/your/project/.claude/skills/
```

Restart Claude Code (or start a new session) so it picks up the new skill.

## Usage

Invoke it directly:

```
/web-perf-audit
```

Or just describe the task in plain language — it also triggers on:

- "optimize performance"
- "speed up the site/app"
- "run a perf audit"
- pasting a checklist like the one this skill covers

Claude will:

1. Scope the target (repo, and a live URL if you want real Lighthouse/network measurements).
2. Run Detect on all 20 items, delegating recon to a lightweight subagent to keep context small.
3. Fix everything found missing.
4. Re-verify what it touched.
5. Report back the checklist with one line of evidence per item.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- For live-URL measurements (Lighthouse, network panel, traces): a browser automation MCP server (e.g. `chrome-devtools` or `playwright`). Without one, the skill falls back to static verification (grep, config inspection, migration files).

## License

MIT — see [LICENSE](LICENSE).

# Fork Notes

Fork of [thedotmack/claude-mem](https://github.com/thedotmack/claude-mem), maintained at
[SirProdigle/claude-mem](https://github.com/SirProdigle/claude-mem).

**claude-mem is kept for what it is uniquely good at — persistent cross-session memory — and stripped
of its planning and code-exploration layer**, which duplicated (and in one case directly contradicted)
[claude-superpowers](https://github.com/SirProdigle/claude-superpowers), the pipeline used here.

## Removed skills (4)

| Removed | Why |
|---|---|
| `make-plan` | A second, incompatible planning pipeline. Its plans (Phase 0 doc discovery, phased structure) cannot be consumed by `superpowers:executing-plans`, which expects superpowers task metadata — and vice versa. Both matched the same trigger ("plan a feature, task, or multi-step implementation"), so which one fired was a coin flip. |
| `do` | The execution half of that pipeline. Superseded by `/execute-plan`, `subagent-driven-development` and `orchestrating-execution`. |
| `learn-codebase` | "Read EVERY SOURCE FILE IN FULL, no matter how many there are. This is critical and non negotiable." Enormous token cost for a cache that a `/compact` discards. Use `spec-miner` (targeted archaeology) or the built-in Explore agent. |
| `smart-explore` | Directly contradicted `learn-codebase` from the same plugin: "Do NOT run Grep, Glob, Read, or find to discover files first" and "**This skill overrides your default exploration behavior**". Whichever loaded first won, with the loser's instructions still in context. The `smart_search` / `smart_outline` / `smart_unfold` MCP tools are **unaffected** — only the directive to override normal file reading is gone. |

## Repointed handoffs

Removing `make-plan` and `do` orphaned four surviving skills that ended by telling you to run them.
Those handoffs now point at the superpowers equivalents:

| Was | Now |
|---|---|
| `/make-plan <prompt>` | `/write-plan <prompt>` |
| `/make-plan` for pre-design ideation with no artifact yet | `/brainstorm` — superpowers' actual entry point for creative work |
| `/do` | `/execute-plan` |
| `/learn-codebase` suggestions | removed |

Affected: `pathfinder` (6 refs), `design-is` (11 — its entire Phase 4 exists to emit a plan-handoff
prompt), `standup` (4), `how-it-works` (1).

Also updated in `src/` so a rebuild keeps them correct: `SearchRoutes.ts` welcome hint,
`install.ts` post-install output, `worker/onboarding-explainer.md`, and the three tests asserting the
old strings.

## Kept

Everything memory-shaped, which is the reason to run claude-mem at all: `mem-search`,
`knowledge-agent`, `timeline-report`, `weekly-digests`, `mode-creator`, `cloud-sync`, `how-it-works`,
plus the standalone utilities `babysit`, `oh-my-issues`, `standup`, `version-bump`, `what-the`,
`wowerpoint`, `design-is` and `pathfinder`.

`pathfinder` is worth keeping despite superficially resembling `spec-miner` in the skills fork — it
detects duplicated concerns *across* features and proposes a unified architecture, which is a
refactor-planning job, not a comprehension job.

## Merging upstream

```bash
git fetch upstream
git merge upstream/main
```

If upstream restores `plugin/skills/{make-plan,do,learn-codebase,smart-explore}`, delete them again
and re-check the handoff references:

```bash
grep -rn "make-plan\|/do\b\|learn-codebase\|smart-explore" plugin/skills --include='*.md'
```

The only legitimate hit is `timeline-report/SKILL.md`, which matches `smart_search` as a stored
`source_tool` value in historical observations — that is data, not a handoff, and must stay.

`push` on the `upstream` remote is deliberately disabled.

## Installing this fork

Unlike `claude-skills` (pure markdown), claude-mem needs its build artifacts present before a
directory-source install will work:

```bash
cd plugin && bun install     # ~519 MB of runtime deps, not in the repo
cd .. && npm run build       # generates plugin/ui and rebuilds plugin/scripts/*.cjs
```

Then point `extraKnownMarketplaces.thedotmack` in `~/.claude/settings.json` at this clone. The
marketplace name (`thedotmack`) and plugin name (`claude-mem`) are unchanged, so the plugin id stays
`claude-mem@thedotmack` and `enabledPlugins` needs no edit.

`scripts/build-hooks.js` verifies a list of required distribution files before finishing; the
`smart-explore` entry is removed there, and any other skill dropped from this fork must be removed
from that list too or the build fails.

After cutting over, restart the worker so it runs from the new root rather than a deleted cache path:

```bash
node plugin/scripts/bun-runner.js plugin/scripts/worker-service.cjs restart
ps aux | grep worker-service.cjs   # should show this clone's path
```

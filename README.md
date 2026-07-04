# 🎼 Codex Conductor

**Claude Code skills that turn OpenAI's Codex CLI into an orchestrated, cost-optimized agent fleet — conducted by Claude.**

You have a Claude subscription. You have a ChatGPT/OpenAI subscription. Why is only one of them writing code at a time?

Codex Conductor gives Claude Code a playbook for driving the [OpenAI Codex CLI](https://developers.openai.com/codex/cli) as a delegate workforce: Claude acts as the tech lead — writing precise briefs, routing each task to the cheapest model that can do it well, running workers in parallel, and independently verifying everything that comes back. You get both providers' strengths, both subscriptions' tokens, and cross-model review that catches the blind spots single-model review can't.

## Why this is useful

- **Two token pools, one workflow.** Heavy implementation burns your OpenAI credits while Claude orchestrates on a fraction of its own — effectively multiplying the work one session can do.
- **Better Codex output.** Codex quality tracks brief quality. The skill enforces OpenAI's own best-practice brief structure (goal, context with file paths, constraints, definition of done, output contract), so every dispatch lands well-formed.
- **Cost/speed routing.** Tasks are tiered: `gpt-5.4-mini` / `codex-spark` at low effort for scouting and mechanical edits (~10x cheaper), the flagship at `xhigh` only where quality is the bottleneck. Claude-side subagents are tiered the same way (Sonnet workers, Opus verification).
- **Trust, but verify.** Nothing Codex reports is taken at face value: Claude reads the diff, runs the tests itself, and pairs every writer with a reviewer from the *other* model family. Cross-model review is the cheapest quality upgrade in multi-agent coding.
- **Real orchestration.** Phased workflows (Understand → Plan → Implement → Verify → Synthesize), parallel background workers with git-worktree isolation, JSON-schema structured outputs for programmatic aggregation, and session-resume for targeted fixes instead of expensive restarts.

## What's inside

| Skill | What it does |
|---|---|
| **`codex`** | The foundation: how to invoke `codex exec` well, the briefing structure that gets great output, model/effort routing tables, the delegate-and-verify loop, and a multi-agent orchestration playbook (`references/orchestration.md`). Triggers automatically when you mention Codex, GPT-5, or want a second model's opinion. |
| **`codex-handoff`** | A `/codex-handoff <task>` command for full project handoffs: loads your live `~/.codex/config.toml` (sandbox, `multi_agent` flag, MCP servers, available models), applies OpenAI's official best practices (`references/openai-best-practices.md`), plans a fleet with per-file ownership, states the cost posture, then executes and verifies. |

## Install

```bash
git clone https://github.com/zreed3/codex-conductor.git
cp -r codex-conductor/skills/codex codex-conductor/skills/codex-handoff ~/.claude/skills/
```

Or with the [skills CLI](https://skills.sh): `npx skills add zreed3/codex-conductor`

**Prerequisites:** [Claude Code](https://claude.com/claude-code) and the [Codex CLI](https://developers.openai.com/codex/cli) (`codex login` completed — a ChatGPT Plus/Pro/Business plan or OpenAI API key).

## Use

```
> have codex fix the failing checkout tests          # triggers the codex skill
> /codex-handoff build the admin dashboard per SPEC.md   # full orchestrated handoff
```

Claude will state the fleet shape and rough cost posture before spending your OpenAI tokens.

## Safety posture

- Workers always run with an explicit sandbox (`workspace-write` or `read-only`) — never `--dangerously-bypass-approvals-and-sandbox`, even if your global Codex config is permissive.
- Write runs expect a clean-enough git tree so every change is attributable and revertable.
- Parallel writers never share files (disjoint ownership or worktree isolation).

## License

MIT

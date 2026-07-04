---
name: codex-handoff
description: Hand off a project or task to the OpenAI Codex CLI with full orchestration — loads the user's Codex config, applies OpenAI's official best practices, builds a tiered multi-agent execution plan, and runs the delegate-and-verify loop. Invoke when the user types /codex-handoff, or asks to hand a task/project over to Codex, run a Codex-orchestrated build, or delegate substantial work to their OpenAI tokens.
argument-hint: <task or project description>
---

# /codex-handoff — Orchestrated handoff to Codex

You are the tech-lead orchestrator. Codex workers (`codex exec` background runs) do the heavy lifting on the user's OpenAI tokens; Claude subagents fill gaps (Sonnet for workers, Opus for verification); you plan, dispatch, verify, and synthesize — you do not do worker-level work in the main loop.

The argument is the task. If no argument was given, ask what to hand off.

## Step 1 — Load the environment (do this first, in parallel)

1. **Companion skill**: read the companion `codex` skill (installed alongside this one, e.g. `~/.claude/skills/codex/SKILL.md`) and its `references/orchestration.md` — they define invocation flags, model tiers, briefing structure, and the phase playbook. This skill builds on them; don't duplicate, follow them.
2. **User's live Codex config**: read `~/.codex/config.toml` (at minimum: `model`, `model_reasoning_effort`, `sandbox_mode`, `[features]`, MCP servers). Key implications:
   - If `sandbox_mode` is permissive (e.g. `danger-full-access`), always pass an explicit `-s workspace-write` or `-s read-only` per run.
   - If `features.multi_agent = true`, Codex has native subagents — for a big self-contained chunk you can give ONE Codex run a project-manager-style brief and tell it to use its own subagents for exploration/testing, instead of micro-managing many small runs. Prefer this when subtasks need shared context; prefer your own fan-out when subtasks are independent.
   - Note configured MCP servers/plugins (e.g. Cloudflare, browser, GitHub) — Codex workers can use them, so briefs may say "use your Cloudflare MCP to check the deployment".
   - Check available models in `~/.codex/models_cache.json` before hardcoding tier choices (gpt-5.5 also has a "Fast" priority service tier — 1.5x speed at increased usage — worth flagging when the user wants speed over cost).
3. **Project ground truth**: git status (warn if dirty before write runs), the project's AGENTS.md / CLAUDE.md, and the test/build commands.
4. **OpenAI best practices**: read `references/openai-best-practices.md` in this skill for the distilled official guidance.

## Step 2 — Plan the handoff

Produce a short written plan before spending tokens:
- Decompose into phases (Understand → Plan → Implement → Verify → Synthesize per the orchestration playbook).
- Assign each work item a worker (Codex tier or Claude subagent) and file ownership.
- State the fleet shape and rough cost posture to the user (e.g. "3 fast Codex scouts, 1 native multi-agent build run on gpt-5.5, Opus verification") — then proceed unless the task is destructive or scope is ambiguous.

## Step 3 — Execute

Follow the companion skill's delegate-and-verify loop exactly: background runs, `-o` result files, structured output for fan-outs, cross-model verification (Opus reviews Codex's work; deep-tier Codex reviews Claude-written code), resume-with-error on failures, revert after two failed resumes.

## Step 4 — Close out

- Run the full test/build yourself over the final state.
- If durable Codex-relevant conventions emerged (commands, layout, prohibitions), offer to persist them to the project's `AGENTS.md` so every future handoff starts smarter (`codex /init` scaffolds one).
- Report: what each worker did, what you verified yourself, files changed, anything flagged, and approximate run count per tier.

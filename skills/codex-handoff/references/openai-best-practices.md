# OpenAI's Official Codex Best Practices (distilled)

Sourced from developers.openai.com (Codex best-practices guide, AGENTS.md guide, prompting guide), current as of July 2026. Verify against the live docs if something seems stale.

## Task briefing (OpenAI's four elements)

Every prompt to Codex should contain:
1. **Goal** — what change or feature you're building.
2. **Context** — relevant files, folders, docs, examples, or errors. Reference specific files by path.
3. **Constraints** — standards, architecture requirements, safety guidelines, conventions.
4. **Done when** — explicit completion criteria: passing tests, resolved bug, verified behavior.

GPT-5.x models respond especially well to an explicit **output contract** (what the final message must contain), **tool-use expectations**, and a precise definition of "done". The biggest quality gains come from matching reasoning effort to the task and from grounding rules (cite file:line, verify before claiming).

## Reasoning effort selection (OpenAI's guidance)

- **low** — faster results for well-scoped tasks
- **medium/high** — complex changes or debugging
- **xhigh** — long, agentic, reasoning-intensive work

## AGENTS.md (durable guidance)

- Codex auto-loads AGENTS.md files from `~/.codex/` plus every directory from repo root to CWD, merged in order with deeper directories overriding shallower ones. The model is trained to adhere closely to these instructions.
- Recommended content: repository layout, setup/run instructions, build/test/lint commands, engineering conventions and PR expectations, prohibited practices, and how to verify completed work.
- `codex /init` scaffolds a starter template.
- A `code_review.md` referenced from AGENTS.md gives consistent review criteria.

## Multi-agent / subagents

- Use subagents to offload bounded work (exploration, testing, triage) and keep the primary agent focused on the core problem.
- OpenAI's project-manager pattern for big work: a coordinating agent writes REQUIREMENTS.md, TEST.md, and AGENT_TASKS.md, then hands off to specialized agents that each write scoped artifacts in their own folder before returning control. Adopt this pattern when giving one native multi-agent Codex run a large brief.

## Review workflows

- Ask for test creation AND execution; run lint and type checks.
- `codex exec review` / the `/review` command gives a PR-style assessment by a separate reviewer agent.

## Automation

- Only automate workflows that already work reliably when run manually.
- Good `codex exec` automation candidates: commit summaries, bug scanning, release-note drafting, CI failure analysis.

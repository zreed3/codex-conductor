---
name: codex
description: Drive the OpenAI Codex CLI (`codex`) as a delegate worker on coding projects — implementing features, fixing bugs, running code reviews, or getting a second-model opinion — using the user's OpenAI subscription tokens. Use this whenever the user mentions Codex, GPT-5, OpenAI, "second opinion from another model", wants to delegate work to another agent, or wants to parallelize work across model providers. Also use it when the user asks to "have codex do it", "ask codex", or compare Claude's and Codex's approaches.
---

# Driving the Codex CLI

Codex is OpenAI's coding agent, available here as the `codex` CLI (authenticated via the user's OpenAI account — usage draws on their OpenAI tokens, not Anthropic's). Your job is to act as the **tech lead**: craft precise briefs, dispatch Codex to do the work, then verify and integrate what comes back. Codex's output quality is highly sensitive to prompt quality — the whole point of this skill is that a well-briefed Codex dramatically outperforms a lazily-briefed one.

## Core invocation

Always use non-interactive mode:

```bash
codex exec [OPTIONS] "PROMPT"
```

Key options:

| Option | Use |
|---|---|
| `-s workspace-write` | Let Codex edit files in the repo (default to this for implementation tasks) |
| `-s read-only` | For analysis/review/opinion tasks — no file changes possible |
| `-C <dir>` | Working root (use the project directory) |
| `-m <model>` | Override model; omit to use the user's configured default |
| `-c model_reasoning_effort="high"` | Dial reasoning effort (`low`/`medium`/`high`/`xhigh`) |
| `-o <file>` | Write Codex's final message to a file (always do this — see below) |
| `--output-schema <file>` | JSON Schema for structured final output |
| `--skip-git-repo-check` | Needed outside a git repo |
| `-i <image>` | Attach screenshots/mockups |

Always pass `-s` explicitly — the user's global config may default to a permissive sandbox, so an explicit `workspace-write` (or `read-only`) keeps each run appropriately scoped. Never use `--dangerously-bypass-approvals-and-sandbox`.

Long prompts: pipe via stdin (`codex exec -s workspace-write - < brief.md`) instead of shell-quoting a giant string.

**Run in the background.** Codex tasks routinely take 2–15 minutes. Use `run_in_background: true` with a generous `timeout`, and continue other work (or verification prep) while it runs. Capture the final message with `-o /tmp/codex-out.md` so you can read the result cleanly instead of parsing the full log.

**Make workers watchable.** Pipe every worker's live stream to a log the user can follow read-only from another terminal:

```bash
LOGS=~/.codex-conductor/logs/$(date +%Y%m%d-%H%M%S); mkdir -p "$LOGS"
codex exec -s workspace-write -o "$LOGS/worker-auth.result.md" "..." 2>&1 | tee "$LOGS/worker-auth.log"
```

After launching workers, tell the user the watch command — one worker: `tail -f "$LOGS/worker-auth.log"`; whole fleet in one pane: `tail -f "$LOGS"/*.log`. The logs show Codex's reasoning, commands, and diffs as they happen without any way to interfere with the run.

## Getting the best output from Codex

Codex does not see your conversation. It starts cold in the repo. Everything it needs must be in the brief. Write briefs like a great ticket:

1. **Context** — what the project is, what's relevant, exact file paths you already know (`src/lib/foo.ts:42`). Every path you supply saves Codex an exploration detour and improves focus.
2. **Task** — precise, single-outcome statement of what to change or produce.
3. **Constraints** — conventions to follow, files NOT to touch, libraries to use/avoid, style of the surrounding code.
4. **Definition of done** — how it will be verified: which command must pass (`npm test`, `tsc --noEmit`), what behavior to demonstrate. Tell Codex to run that command itself before finishing.
5. **Output contract** — what its final message must contain (e.g. "end with a summary of every file changed and the test output").

## Model routing — optimize cost and speed per task

Don't send every task to the flagship. Pick model + reasoning effort by the difficulty of the task, not habit:

| Tier | Model / effort | Use for |
|---|---|---|
| Deep | `gpt-5.5` + `xhigh` | Architecture, gnarly debugging, security-sensitive changes, final adjudication |
| Standard | `gpt-5.5` + `medium` (or `gpt-5.4`) | Typical feature implementation, refactors |
| Fast | `gpt-5.4-mini` or `gpt-5.3-codex-spark` + `low` | Mechanical edits, boilerplate, renames, doc updates, scouting/summarizing files, first-pass triage |

```bash
codex exec -m gpt-5.4-mini -c model_reasoning_effort="low" -s read-only "..."
```

The fast tier is dramatically cheaper and faster — route high-volume fan-out work (see orchestration below) there, and reserve the deep tier for the few calls where quality is the bottleneck. Confirm available models with `codex exec -m <model>` failing fast, or check `~/.codex/models_cache.json`.

## Multi-agent orchestration

You are the orchestrator, and multiple concurrent `codex exec` background runs are your agent fleet. (If the user's config enables `features.multi_agent`, Codex also has native subagents — one run can be given a project-manager brief and spawn its own; prefer that when subtasks need shared context, and your own fan-out when they're independent.) This replicates Claude's workflow/fan-out ability with Codex workers. When a task is big enough to decompose (audits, migrations, multi-part features, N-perspective reviews), read [references/orchestration.md](references/orchestration.md) for the full playbook. The short version:

- **Fan out** independent subtasks as parallel background `codex exec` runs, each with its own brief, `-o` result file, and fast-tier model where possible.
- **Isolate writers**: parallel write runs must never share files — use disjoint file sets or `git worktree` copies per worker, then merge.
- **Phase like a workflow**: Understand (parallel read-only scouts) → Plan (you, or one deep-tier run) → Implement (parallel writers) → Verify (cross-check with a different model than the one that wrote the code).
- **Structured results**: give fan-out workers `--output-schema` so you can aggregate their answers programmatically instead of parsing prose.
- You can also mix providers: spawn Claude subagents (Agent tool) and Codex workers side-by-side, and use each model to review the other's output — cross-model review catches blind spots same-model review misses.
- **Keep the orchestrator cheap**: you coordinate and synthesize, but delegate the heavy reading/writing. Claude-side workers run as Sonnet subagents (`model: "sonnet"` on the Agent tool); escalate to Opus (`model: "opus"`) only for verification and adjudication passes. Don't do worker-level work in the main loop.

## The delegate-and-verify loop

Never treat Codex output as done. After each run:

1. Read the final message (`-o` file) and `git diff --stat` to see what changed.
2. Review the diff yourself for correctness, scope creep, and convention violations.
3. Run the project's tests/build yourself — don't trust Codex's claim that they pass.
4. If issues are found, resume the same session with targeted corrections rather than starting over (Codex keeps its context):
   ```bash
   codex exec resume --last "The tests fail with <paste error>. Fix X without touching Y."
   ```
5. Report to the user what Codex did, what you verified, and anything you fixed or would flag.

If Codex has gone in a wrong direction twice, stop resuming — revert its changes (`git checkout -- <files>` or ask the user), rewrite the brief with what you learned, and start fresh.

## Task patterns

**Implementation** — `-s workspace-write`, full brief, background run, verify loop above. For larger projects, decompose into sequential Codex runs (one coherent change each) rather than one mega-prompt; verify between runs.

**Second opinion / analysis** — `-s read-only`, ask a pointed question ("Is this locking strategy in src/db/pool.ts sound under connection churn? Cite line numbers"). Compare its answer with your own view and present both to the user with your assessment.

**Code review** — Codex has a built-in reviewer:
```bash
codex exec review --help   # check flags, then e.g. review current diff
```
Or brief it manually in read-only mode against a diff. Cross-check any finding before relaying it as fact.

**Parallel work** — you and Codex can work simultaneously on disjoint parts (e.g. Codex writes tests while you implement), but never let both edit the same files at once; the diffs will collide.

## Housekeeping

- Check it's a git repo with a clean-enough tree before a write run, so Codex's changes are attributable and revertable. Suggest committing pending work first.
- Codex reads `AGENTS.md` files (its analogue of CLAUDE.md). If the project has repeated Codex-relevant conventions, offer to persist them there.
- Costs: each run spends the user's OpenAI tokens/subscription credits. Be deliberate — one well-briefed run beats three vague ones.

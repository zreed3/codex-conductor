# Codex Multi-Agent Orchestration Playbook

You (Claude) are the orchestrator; each `codex exec` background run is a worker. The goal is to replicate Claude's workflow ability — phased fan-out, verification, synthesis — while routing each call to the cheapest model that can do it well.

## Ground rules

1. **One brief, one outcome per worker.** Workers don't see each other or your conversation. Each brief must be self-contained (context, task, constraints, definition of done, output contract).
2. **Every worker gets its own `-o` file** in the scratchpad (`worker-<name>.md`), so results are cleanly readable.
3. **Launch all independent workers in the same turn** with `run_in_background: true`, then process completion notifications as they arrive. Don't launch serially what can run in parallel.
4. **Cap concurrency at ~4–6 workers.** More burns tokens faster than you can verify and risks rate limits.
5. **Route by tier** (see SKILL.md table): scouts and mechanical work → fast tier; implementation → standard; adjudication/hard problems → deep tier.
6. **Claude-side workers are tiered too.** When a phase calls for a Claude subagent (Agent tool), default to Sonnet (`model: "sonnet"`) for scouting, implementation, and test-writing; escalate to Opus (`model: "opus"`) for verification, adjudication, and security-sensitive review. The main-loop model orchestrates and synthesizes — it shouldn't burn its context doing worker-level reading and writing.

## The phase pattern (mirrors Claude workflows)

### Phase 1 — Understand (parallel read-only scouts, fast tier)
Fan out cheap scouts to map the territory:
```bash
codex exec -s read-only -m gpt-5.4-mini -c model_reasoning_effort="low" \
  -o "$SCRATCH/scout-auth.md" "Map how authentication works in this repo: entry points, middleware, session storage. List exact file:line references. Be terse — bullet points only."
```
One scout per subsystem/question. Aggregate their reports yourself.

### Phase 2 — Plan (you, or one deep-tier run)
Synthesize scout reports into a decomposition: subtasks, file ownership per subtask, verification command per subtask. You usually do this yourself; for genuinely hard architecture, get a deep-tier second opinion and reconcile.

**File ownership is the critical output**: every file may be owned by at most one writer in the next phase.

### Phase 3 — Implement (parallel writers, standard tier)
- **Disjoint files** → run writers concurrently in the same repo with `-s workspace-write`, each brief explicitly listing "you may ONLY modify these files: ...".
- **Overlapping files** → serialize those workers, or isolate with worktrees:
  ```bash
  git worktree add /tmp/codex-w1 -b codex/w1
  codex exec -C /tmp/codex-w1 -s workspace-write -o "$SCRATCH/w1.md" - < brief-w1.md
  # ...merge branch after verification, then: git worktree remove /tmp/codex-w1
  ```
- Include in every writer's brief: the verification command it must run before finishing, and "end with a list of every file you changed".

### Phase 4 — Verify (cross-model, cheap first)
- Run the project's tests/build yourself — never trust worker claims.
- Review each diff yourself for scope creep and convention violations.
- For important changes, add a cross-model reviewer: a different model than the writer (`codex exec review`, a read-only deep-tier Codex run over the diff, or an Opus Claude subagent — `model: "opus"` on the Agent tool). Same-model review shares the writer's blind spots, so pair Codex writers with Opus reviewers and Claude/Sonnet writers with deep-tier Codex reviewers.
- Failures → `codex exec resume` the responsible worker's session with the exact error pasted in. Two failed resumes → revert and re-brief fresh.

### Phase 5 — Synthesize
Merge results, run the full test suite once more over the combined state, and report to the user: what each worker did, what you verified, total cost posture (how many runs at which tiers).

## Structured fan-out results

When you'll aggregate many workers' answers, force machine-readable output:
```bash
cat > /tmp/finding-schema.json <<'EOF'
{"type":"object","properties":{"findings":{"type":"array","items":{"type":"object",
 "properties":{"file":{"type":"string"},"line":{"type":"integer"},
 "severity":{"type":"string","enum":["high","medium","low"]},"summary":{"type":"string"}},
 "required":["file","summary","severity"]}}},"required":["findings"]}
EOF
codex exec -s read-only --output-schema /tmp/finding-schema.json -o "$SCRATCH/audit-sql.json" "..."
```
Then aggregate/dedupe the JSON yourself before spending tokens on verification.

## Ready-made orchestration shapes

- **Audit/review sweep**: N read-only scouts, one dimension each (security, perf, correctness, conventions) → dedupe findings → verify each surviving finding with one adversarial deep-tier run ("try to refute this finding") → report only confirmed items.
- **Migration**: one scout enumerates all call sites → chunk into disjoint file groups → parallel fast-tier writers, one group each → you run the build/tests between merges.
- **Design bake-off**: 2–3 deep-tier read-only runs propose approaches from different angles → you (or one more run) judge and synthesize → single standard-tier writer implements the winner.
- **Cross-provider pair**: Codex implements while a Sonnet subagent writes tests against the spec (or vice versa), on disjoint files; an Opus subagent (or deep-tier Codex run) then reviews the combined result.

## Cost discipline

- Prefer one well-briefed run over retries: the brief is where quality per token is won.
- Scouts and triage at fast tier can be ~10x cheaper — never scout with the flagship.
- If a phase's output won't change your next action, skip the phase.
- Tell the user the shape before launching a big fleet ("6 workers: 4 fast scouts, 2 standard writers") — parallel Codex runs spend their OpenAI tokens quickly.

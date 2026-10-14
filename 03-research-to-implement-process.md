# Research -> Implement: how we work

You give instructions and review. The coding agent (Codex / OmniRoute / any) types.
Rule: never accept a diff you can't explain; never skip the "plan first" reply.

## Same lifecycle as the repo

`AGENTS.md` defines UNDERSTAND -> PLAN -> IMPLEMENT -> VALIDATE -> REVIEW -> RECORD -> COMPLETE. Each prompt in `04-prompt-guide.md` runs one full turn of it.

| Phase | In practice |
|---|---|
| UNDERSTAND | You read the 1-2 concepts listed for that step (15-20 min, 5 lines of notes). The agent reads repo rules and docs. |
| PLAN | Agent replies with files, dependencies, validation plan. You say "go" or correct it. |
| IMPLEMENT | Agent codes only that step. |
| VALIDATE | Agent runs the smallest useful pnpm command and shows output. |
| REVIEW | You use the gate checklist for that step. |
| RECORD | Decisions and results go to `.project/story-agent/`. |
| COMPLETE | One standup line. |

## Build sequence (no days; move on only when the gate passes)

| Step | Prompt | You read first (concepts doc) | Gate |
|---|---|---|---|
| 0 | Setup: env, docs, keys | none | Each provider answers a curl test |
| 1 | Recon, no code | none | Report matches your picture of the repo; conflicts resolved |
| 2 | Single-provider client + schemas | 1-6 | Smoke script returns a real reply; tests pass |
| 3 | Router: profiles, failover, circuit breaker | 17-19 | Fake-provider tests: 429 -> next provider, cooldown, all-down error |
| 4 | Pipeline + repair loop | 8-10, 13 | Fake-model tests: happy / clarify / repair / fail / budget |
| 5 | Atlas + API | 14-15 | curl creates a run; run saved even when it fails |
| 6 | UI | none | You click through every state including queued |
| 7 | Queue + worker | 20 | Kill all providers -> run queued -> restore -> run completes |
| 8 | Eval + hardening | 12, 16 | Eval table printed; `check` and `scan` clean |
| 9 | Record + demo | none | Docs match reality; demo of all paths works |

## Review checklist (use after every step)

- Only allowed files touched? Nothing in `codebase/`, `.agent/`, `architecture.yaml`, AGENTS.md, CLAUDE.md?
- Pure functions, explicit dependencies, flat feature files, no classes/global state?
- Model output validated before use or save?
- Limits present: retries, repairs, calls, tokens, timeout?
- Prompts in their own versioned files?
- Keys only from env; nothing secret in diff?
- Can I explain each file in one sentence?

## If the agent goes wrong (paste these)

- "You changed files outside scope: <paths>. Revert them and explain why you touched them."
- "Explain <file> in 5 lines and where its limits are enforced."
- "Your plan adds <dependency>. Justify it or do it without."
- "Stop. Re-read AGENTS.md section <x> and fix your approach."
- Weak/free coding model drifting? Split the step into smaller tasks (one file at a time).

## Standup template

- Understood: <concept>, including how it fails.
- Built: <step> by <approach>, verified by <check>.
- Next: <step>. Blocked on: <decision or nothing>.

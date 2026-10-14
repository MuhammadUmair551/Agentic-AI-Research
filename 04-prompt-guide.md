# Prompt guide: paste these into Codex / OmniRoute, one step at a time

Order matters. One branch per step. After each step: check the gate, then the review checklist in `03-research-to-implement-process.md`.

## Step 0: Setup (you, no agent)

1. Docs live in `.project/story-agent/` (replace with these new versions).
2. `.env` has Gemini and Groq keys + models already. Add **Ollama Cloud** as backup [Likely, verify]:
   - Key: ollama.com/settings/keys. Base URL: `https://ollama.com/v1`.
   - Model name: list current ones (they get retired): `curl https://ollama.com/api/tags -H "Authorization: Bearer $OLLAMA_KEY"` and pick one cloud model.
3. Test every provider with curl before involving any agent (example for one provider):
```
curl "$BASE_URL/chat/completions" -H "Authorization: Bearer $API_KEY" -H "Content-Type: application/json" -d "{\"model\":\"$MODEL\",\"messages\":[{\"role\":\"user\",\"content\":\"Reply with JSON {\\\"ok\\\":true}\"}]}"
```
4. Never paste key values into chat or into the agent. Names only.

## Common preamble (paste at the top of EVERY prompt)

```
You are the implementation agent for the "story-agent" feature in this repository.
First read: AGENTS.md, CLAUDE.md, architecture.yaml, and every guide AGENTS.md points to under .agent/.
They override anything I write if they conflict: tell me about the conflict, never choose silently.
Product docs: .project/story-agent/ (read 02-agentic-ai-architecture.md fully; 01 and 03 as needed).
Scope: application feature "story-agent" only. Do NOT modify codebase/, architecture.yaml, .agent/, AGENTS.md,
CLAUDE.md, pnpm-workspace.yaml. Do not add a monorepo orchestrator. No new dependency without asking me first and explaining why.
Style: feature-first flat files, pure functions, immutable data, dependencies passed explicitly. No classes, no DI container,
no global state (stateful things are created at the feature's composition point and passed in).
Secrets: never print, log or write key values anywhere. Env vars only; add only variable NAMES to .env.example.
Method: reply first with a short plan (files to create/change, dependencies, how you will validate) and WAIT for my "go".
After "go": implement only this task, run the smallest useful validation command, show me the output,
then summarize in at most 10 lines what changed and anything you were unsure about.
```

---

## Step 1: Recon (no code). If you already ran it, skip to the check below.

Gate: you reviewed Codex's report with me; conflicts with doc 02 are resolved (either docs or plan updated). Then reply to Codex: "Recon accepted. Corrections: <list>. Wait for the next task."

## Step 2: Single-provider client + schemas

```
[PREAMBLE]
Task: model client and contracts. Nothing else. No router, no DB, no routes, no UI, no pipeline.
Context: .env already contains Gemini and Groq keys and models. Find the existing variable NAMES from .env.example or by listing names only (never print values).
Build:
- story-agent.integration.ts: a client for any OpenAI-compatible chat-completions API. It takes a ProviderConfig {name, baseUrl, apiKey, model, timeoutMs} as a PARAMETER (it does not read env itself). Input: messages. Output: { text, usage:{inputTokens,outputTokens}, provider, model }.
  Typed errors that separate: retryable (429, 5xx, timeout, network; include Retry-After if present) vs auth/config (401, 403) vs bad request (400) vs bad response shape.
  No retries here (the router will own failover). Timeout via AbortController.
- Env loader (pure function from an env object to ProviderConfig[]): target scheme in doc 02 section 3 (LLM_PROVIDERS, LLM_<NAME>_BASE_URL/_API_KEY/_MODEL). Propose how to map my EXISTING Gemini/Groq variable names (rename with my approval, or support both) and add Ollama Cloud (base URL https://ollama.com/v1) in the same pattern. Update .env.example with names only.
- zod schemas: RequirementInput, Clarifications, Story (title, intent, epic?, feature?, size S|M|L, acceptanceCriteria[], states[], dependsOn[], openQuestions[], sliceRationale), RunStatus (questions|draft|queued|failed).
- Fake model client for tests. JSON extractor (handles code fences and leading/trailing prose) + zod validation returning ok/error results, never throwing on bad model output.
- Tests: error classification (status code -> error kind) using a fake fetch, env loader, schemas, JSON extraction.
- Manual smoke script (not part of pnpm test): for each configured provider make one tiny call and print provider, model, latency, usage, or the error KIND (never the key).
Validate: typecheck, lint, tests for this unit. Show output.
```
**Gate:** you run the smoke script: Gemini, Groq and Ollama each return a reply or a clear error kind. Tests pass.
**Standup:** "I wrapped each provider behind one OpenAI-compatible client driven by env, and classified errors so failover can decide what to do."

## Step 3: Router: profiles, failover, circuit breaker

```
[PREAMBLE]
Task: provider router. Only this. No pipeline, DB, routes, UI.
Build story-agent.router.ts (flat file, functions only):
- createRouter({ providers, routes, clock, call, cooldownMs }) returns a function route(profile, messages) -> { text, usage, provider, model, attemptedProviders[] } or a typed error.
  Profiles: fast, generate, tools. routes come from env: LLM_ROUTE_FAST, LLM_ROUTE_GENERATE, LLM_ROUTE_TOOLS (ordered provider names). Unknown provider names in a route are config errors reported at startup.
- Failover: retryable error -> try the next provider in that profile's list. Auth/config error (401/403) -> skip that provider, log the error kind (never the key), continue. Bad request (400) -> do not fail over blindly: return the error.
- Circuit breaker (simple cooldown): 2 consecutive retryable failures -> provider is "open" for cooldownMs (use Retry-After if present, else LLM_COOLDOWN_MS). While open, skip it. After cooldown allow ONE trial; success closes it, failure re-opens it.
  State lives in a closure created inside createRouter (no module global). The clock is injected so tests need no real time.
- If every provider in the chain is unavailable -> typed error providers_unavailable. If the profile has no configured provider -> no_provider_configured.
- Router never retries the same provider in a tight loop.
Tests with fake call/clock: 429 on first -> second used; open breaker skips provider until cooldown; trial after cooldown; 401 skipped; all down -> providers_unavailable; route order respected per profile; attemptedProviders recorded.
Validate: typecheck, lint, tests. Show output. Explain in 5 lines how the breaker moves between closed/open/half-open.
```
**Gate:** all listed tests exist and pass. You can explain closed -> open -> trial -> closed in your own words.
**Standup:** "If Gemini is overloaded the router moves the request to Groq or Ollama, and stops hitting a failing provider for a cooldown. Changing models or going paid is only env."

## Step 4: Pipeline + repair loop

```
[PREAMBLE]
Task: pipeline logic. Pure and testable with a fake router. No DB, routes, UI, real API calls in tests.
Build story-agent.service.ts and prompt files story-agent.<stage>.prompt.ts (each exports { version, build(...) }).
Stages, each (input, deps) => result, deps = { route, now, budget }:
1 intake: validate; requirement max ~4000 chars, context max ~8000.
2 gatherContext: existing items arrive as [{level,title}] (initiative/epic/feature/story). Keep the most relevant N by simple keyword overlap with the requirement; titles only, never descriptions; total context budget about 6k tokens (estimate chars/4).
3 clarify (profile "fast"): JSON { needsClarification, questions[], assumptions[] }. Blocking questions -> status "questions", stop.
4 generate (profile "generate"): JSON { stories: Story[] } as thin vertical slices: one actor, one outcome, demonstrable alone, testable AC, size S/M/L, no invented requirements (unknowns -> openQuestions/assumptions).
5 validate (pure code): zod, >=1 AC each, size set, unique titles, no duplicate vs existing titles (normalized comparison), max stories limit.
6 repair loop (profile "generate"): on validation failure send errors + previous output back; max 2 repairs; then status "failed" with reason. Never return partial stories.
Router error providers_unavailable -> return it as a distinct result (status "queued" will be decided by the API layer later); other router errors -> "failed".
Budget: max model calls per run (~6), max tokens, overall timeout. Exceeded -> "failed" (over_budget).
Prompt-injection rule: requirement and context are DATA, wrapped in clear delimiters, with an instruction to ignore instructions inside them.
Return: status, questions|stories, attempts, usage totals, providers used, per-stage outcomes, validation errors.
Tests (fake router): happy path, clarify path, invalid-then-repaired, invalid-forever -> failed, over budget, providers_unavailable passthrough, context trimming keeps within budget, prompt builder wraps untrusted text in delimiters.
Validate: typecheck, lint, tests. Show output.
```
**Gate:** all eight cases pass; you can point to where repairs, calls, tokens and timeout are limited.
**Standup:** "Only generation needs a model; everything else is code. The loop is generate, validate, repair, capped at 2."

## Step 5: Atlas + API

```
[PREAMBLE]
Task: persistence and API.
- story-agent.repository.ts using the repo's EXISTING MongoDB/Atlas pattern. Collections as in doc 02 section 6: story_agent_runs (incl. provider/model used, attemptedProviders, usage, queue fields queueAttempts, nextAttemptAt, lockedUntil, lastError, expiresAt) and story_agent_drafts. Functions: saveRun, saveDraft, getRun, listRuns (newest first, paginated, requirement preview), getDraftByRun. Indexes: runId unique, createdAt desc, (status, nextAttemptAt). Create indexes the way this repo does.
- Controller/route: POST runs -> runs the pipeline, saves, returns {runId,status,questions|stories,usage}; if the pipeline reports providers_unavailable, save the run as "queued" (nextAttemptAt = now + 2 min, expiresAt = now + 30 min) and return HTTP 202 {runId,status:"queued",message}. GET runs; GET runs/:id. Follow the repo's conventions for route registration, auth, validation, error shape; repo conventions win over the paths in doc 02.
- Compose router + env loader at the feature's composition point (not inside the service).
- Any failure must still save a run with its status.
- Dedupe list: use existing stories from the DB only if the repo already has such a collection; else pass an empty list and tell me.
Tests: repository/service tests the way this repo tests DB code (ask me if no pattern exists); route tests with a fake router for 200, 202 queued, failed.
Validate: smallest relevant pnpm command, then pnpm run check. Give me curl examples for POST and GET.
```
**Gate:** curl gives a real draft; the run exists in Atlas; with all keys broken you get 202 queued (not a crash); a failed run is still saved.

## Step 6: UI (two phases, you approve the layout)

```
[PREAMBLE]
Task: web UI for story-agent.
Phase A (no code): read .agent/instructions/ui-composition.md. Propose the page layout as a text sketch, components and every state, and say how it will port into a larger production UI later (reusable components, no page-specific hacks). Ask me what I want changed. WAIT for my approval.
Phase B (after approval): implement
- requirement + optional context textareas, Generate button
- states: empty, loading (up to ~60 s), error, questions, draft, queued
- draft: story cards (title, intent, size badge, AC list, slice rationale, open questions); run info (provider, model, tokens, duration, attempts)
- queued: message "High demand right now. Your request is queued; the result usually arrives in a few minutes", polls GET runs/:id every ~10-15 s, shows the result when ready, stops polling on a final status or when the page closes
- history list and opening a past run
Follow the shadcn/Tailwind and component-hierarchy rules from the guide; call the server the way this web app already does. No new UI dependency without asking.
Validate: typecheck, lint, component tests where the repo has them. Start dev and tell me exactly what to click, including how to see the queued state (e.g. temporarily break provider keys).
```
**Gate:** you click through every state.

## Step 7: Queue + worker

```
[PREAMBLE]
Task: queue worker. Only when Steps 0-6 are green.
Build story-agent.queue.ts (+ repository functions as needed), no Redis, no Docker, no new dependency unless I approve:
- claimNext: atomically claim one run with status "queued", nextAttemptAt <= now and not locked, via a single findOneAndUpdate that sets lockedUntil. Safe with several server instances.
- processOne: run the SAME pipeline function the API uses. Success -> save draft, status draft/questions. providers_unavailable again -> queueAttempts+1, nextAttemptAt = now + interval, unlock. Other failure -> failed.
- Limits: max queue attempts (e.g. 5), max age via expiresAt (e.g. 30 min) -> failed with reason "expired"; max queue length (e.g. 20) -> POST rejected with a clear error.
- Process one run at a time (free tiers allow little concurrency).
- Worker start/stop: an in-process timer (interval from env, default ~2 min) started at the feature's composition point and stopped on shutdown. Explain how it behaves with multiple instances.
Tests with fake clock/router: claim is exclusive, retry reschedules, expiry, queue-full rejection, success path, lock timeout recovery.
Validate: typecheck, lint, tests. Show output and explain how to demo: break keys -> POST -> queued -> restore keys -> result appears.
```
**Gate:** you demo it end to end with the UI polling.

## Step 8: Eval + hardening

```
[PREAMBLE]
Task: evaluation and hardening.
- Eval script (not part of pnpm test) with 5 fixtures: one clear, one ambiguous, one with existing items to avoid duplicating, one long, one prompt-injection attempt ("ignore previous instructions and ..."). Run the real pipeline through the router and print per case: status, valid?, attempts, story count, tokens, time, provider used; then a summary: valid-output rate, average tokens, and a per-provider table (calls, failures, avg latency).
- Add tests: oversized input rejected; injection fixture does not change status or output shape.
- Confirm no secrets in the repo and none in logs. Run pnpm run check and pnpm run scan.
- List the 3 biggest weaknesses and propose prompt or route-order fixes. Apply only after my approval.
```
**Gate:** eval table printed; check and scan clean. Use the per-provider table to reorder `LLM_ROUTE_*` in env (config only).
**Standup:** "AI output isn't deterministic, so I measured it, and I picked provider order from data."

## Step 9: Record + demo

```
[PREAMBLE]
Task: record and wrap-up. Change no code except documentation mismatches.
- Update .project/story-agent/ docs to match what was built: file names, endpoints, collections, env vars (full list), router profiles, known limits (02 becomes final).
- Short usage doc: setup env, run, test, eval, demo script covering: draft, questions, failed/over-budget, repaired, failover (break one provider), queued (break all).
- V2 candidates: tool calling on the "tools" profile, critic pass, accept/reject drafts, task and test generation, shared breaker state in Squid, moving the feature into Squid's routing layer.
- Risks you saw.
```
**Gate:** you run the demo yourself once, all paths.

## Optional Step 10 (only if everything is green)

```
[PREAMBLE]
Task: optional tool-calling context step behind a feature flag, default off.
One read-only tool "list_existing_items" on the "tools" profile, max 3 tool steps, fallback to the plain-code context path if the provider does not support tool calls. Tests with fake router. Explain how it fails.
```

## If the agent goes wrong

- "You changed files outside scope: <paths>. Revert and explain."
- "Explain <file> in 5 lines and where its limits are enforced."
- "Your plan adds <dependency>. Justify it or do it without."
- "Stop. Re-read AGENTS.md section <x> and fix your approach."
- Coding model drifting? Split the step into one file at a time.

# Story Agent: Architecture (draft v0.4)

Owner: Umair. Feature inside the new Squid codebase, designed so it can move into Squid later (Kraken-style routing).
Repo rules win over this doc: `AGENTS.md`, `CLAUDE.md`, `architecture.yaml`, `.agent/*`. Conflicts must be reported, not silently resolved.
Old Squid docs are only a story-format reference and sample data.

## 1. Purpose and scope

Input: requirement (+ optional context). Output: clarifying questions, or draft stories (thin vertical slices, AC, size), saved and shown in a UI. Survives free-tier outages by failing over between providers and, if all are down, queueing the run.

**V1 in scope:** pipeline, repair loop, multi-provider routing with failover and cooldown, Atlas storage, API, UI with history, DB-backed queue + in-process worker.
**Out of scope:** tasks/test generation, accept/reject workflow, tool calling (optional last step), multi-agent, RAG, Redis, Docker, streaming, training anything. Quality bar: working.

## 2. Repo rules that shape it

| Rule | Consequence |
|---|---|
| Feature-first flat server files | `story-agent.route/.controller/.service/.repository/.validation/.model/.integration.ts`; extra flat files for router/queue only if needed (e.g. `story-agent.router.ts`, `story-agent.queue.ts`). No `domain/`. |
| Functional style, explicit dependencies, no classes/DI/global state | Stages `(input, deps) => result`. Router, clock, model client are created at the feature's composition point and passed in. Breaker state lives in a closure created there, never a module global. |
| External systems behind `.integration.ts` | Provider HTTP calls only there. |
| Contracts local until reused | Schemas stay in the feature. |
| App work != foundation work | Never edit `codebase/`, `architecture.yaml`, `.agent/`, `AGENTS.md`, `CLAUDE.md`, `pnpm-workspace.yaml`. |
| New dependency only with reason | Ask first. |

## 3. Decisions (made)

| ID | Decision | Choice |
|---|---|---|
| D1 | Where | Server feature + web page in existing units (recon confirms) |
| D2 | Provider access | Providers = OpenAI-compatible endpoints configured **only by env**. Starting set: Gemini, Groq, Ollama Cloud (backup). Paid migration = add/replace provider env values and routes, no code change |
| D3 | Routing | **Profiles** by task type: `fast`, `generate`, `tools`. Each profile = ordered provider list from env. Code knows profile names only, never model names |
| D4 | Failover | Retryable failure (429, 5xx, timeout, network) -> next provider in the profile's list. Auth/config failure (401/403) -> skip provider, log clearly, continue |
| D5 | Circuit breaker | Simple per-provider cooldown: 2 consecutive retryable failures -> open for cooldown (honor Retry-After if present, else `LLM_COOLDOWN_MS`) -> then allow one trial request -> success closes it. In-memory, injected clock. Per process only (Squid can move it to shared storage later) |
| D6 | Loop | Repair loop: generate -> validate in code -> send errors back -> retry max 2. Bad JSON is a repair problem, not a provider failure |
| D7 | Storage | MongoDB Atlas: `story_agent_runs`, `story_agent_drafts` (existing repo pattern) |
| D8 | Queue | Only when **all providers for the needed profile are unavailable**. Run saved as `queued`; in-process worker retries every ~2 min; no Redis/Docker |
| D9 | Quality bar | Working; eval prints valid-output rate and provider usage |

### Env scheme (target; Codex adapts the already-existing Gemini/Groq names with Umair's approval)

```
LLM_PROVIDERS=gemini,groq,ollama
LLM_<NAME>_BASE_URL / LLM_<NAME>_API_KEY / LLM_<NAME>_MODEL    (for each name)
LLM_ROUTE_FAST=groq,gemini,ollama
LLM_ROUTE_GENERATE=gemini,groq,ollama
LLM_ROUTE_TOOLS=gemini,groq            (optional, last step)
LLM_COOLDOWN_MS=60000   LLM_TIMEOUT_MS=60000
```
Go paid later: add `LLM_PAID_*` and put `paid` first in the routes. Env only.
Starting assignments are guesses (fast inference for `fast`, long-context/structured output for `generate`); the eval step measures them, then you reorder env. [Guessing]

Ollama Cloud notes [Likely, verify]: OpenAI-compatible base URL `https://ollama.com/v1`; API key from ollama.com/settings/keys. Free tier is light usage: session limits reset about every 5 hours, weekly limits every 7 days, one cloud model at a time. Models get retired, so list current names (`/api/tags`) before choosing.

## 4. Context

```
Person -> [Web page] -> [Story Agent API] -> [Router: profile -> provider chain w/ cooldown] -> Gemini | Groq | Ollama
                              |-> MongoDB Atlas: runs, drafts, queue state
                              |-> [Worker, every ~2 min] picks queued runs -> same pipeline
Person <- questions | draft | queued (message) | failed + reason
```

## 5. Pipeline and profiles

| # | Stage | Kind | Profile |
|---|---|---|---|
| 1 | Intake | code | n/a |
| 2 | Gather context | code, **context budget** | n/a |
| 3 | Clarify | model JSON | `fast` |
| 4 | Generate | model JSON | `generate` |
| 5 | Validate | code | n/a |
| 6 | Repair loop (max 2) | model | `generate` |
| 7 | Finalize | code | n/a |

**Context budget:** send existing items as `{level, title}` only (levels may be initiative/epic/feature/story), keep the most relevant N by simple keyword overlap, cap total context at roughly 6k tokens (estimate chars/4), never full descriptions. Requirement max ~4000 chars, context max ~8000.
**Run budget:** max model calls (~6), max tokens, overall timeout. Exceeded -> `failed` (`over_budget`).

## 6. Contracts

**Story:** `title ("A <role> <does> <outcome>") · intent · epic? · feature? · size S|M|L · acceptanceCriteria[] · states[] · dependsOn[] · openQuestions[] · sliceRationale`
Statuses: `questions | draft | queued | failed`.

**Router result:** `{ text, usage, provider, model, attemptedProviders[] }`; errors: `providers_unavailable` (all retryable-failed), `no_provider_configured`.

**API** (repo conventions win over these indicative paths):
- `POST /story-agent/runs` -> 200 `{runId,status,questions|stories,usage}` or **202** `{runId,status:"queued",message}`
- `GET /story-agent/runs` (newest first, paginated)
- `GET /story-agent/runs/:id` (run + draft; UI polls while `queued`)

**Collections:**
- `story_agent_runs`: runId, requirement, context, status, provider/model actually used, attemptedProviders, promptVersions, attempts, usage, durationMs, validationErrors, error, createdAt, plus queue fields `queueAttempts, nextAttemptAt, lockedUntil, lastError, expiresAt`
- `story_agent_drafts`: runId, stories[], reviewStatus (`draft` only), createdAt
- Indexes: runId unique; createdAt desc; (status, nextAttemptAt) for the worker

## 7. Queue and worker (D8)

- Enqueue only on `providers_unavailable`. Never on validation failure.
- Worker: in-process timer every ~2 min (configurable), started at the feature's composition point, stopped on shutdown. Claims one run at a time atomically (`findOneAndUpdate` on `status=queued, nextAttemptAt<=now, not locked`, sets `lockedUntil`), so it is safe with several instances and respects Ollama's one-at-a-time limit.
- Max queue attempts (e.g. 5) and max age (e.g. 30 min) -> `failed` (`expired`). Max queue length (e.g. 20) -> reject new runs with a clear error.
- Worker runs the same pipeline function the API uses.

## 8. UI (V1)

One page: requirement + optional context, Generate; loading state (up to ~60 s); result as question list or story cards (title, intent, size badge, AC, rationale, open questions); run info (provider, model, tokens, duration, attempts); history and past runs; empty/error states; **queued state**: message "High demand right now. Your request is queued; the result usually arrives in a few minutes", auto-polls `GET runs/:id`, shows the result when ready. Layout is proposed by the coding agent and approved by Umair first. Follows `.agent/instructions/ui-composition.md` and uses shadcn/Tailwind so it ports into Squid's production UI.

## 9. Failure branches

| Failure | Behaviour |
|---|---|
| Provider 429/5xx/timeout | Next provider; cooldown after 2 consecutive failures |
| Provider 401/403 | Skip, log as config error, continue |
| All providers down | Run saved `queued`, 202, worker retries |
| Invalid output after 2 repairs | `failed`, nothing partial |
| Over budget/timeout | `failed` (`over_budget`) |
| Ambiguous requirement | `questions` |
| Queue expired / queue full | `failed` (`expired`) / rejected with clear error |

## 10. Security

Requirement/context are data, not instructions (delimited in prompts). Keys only from env; never printed, logged or committed (`pnpm run scan`). Output validated before save/return. No stack traces in errors. Provider names and keys never sent to the browser.

## 11. Risks

1. Free-tier spikes: failover + cooldown + queue.
2. Weak JSON from some models: extract + zod + repair.
3. Ollama Cloud limits: one at a time, session/weekly caps, models retired. Config only, never hard-coded.
4. Breaker state is per process: fine for V1.
5. More moving parts: each step is gated, queue comes after the sync path works.
6. Coding-agent drift: scope guard in every prompt, review the diff.

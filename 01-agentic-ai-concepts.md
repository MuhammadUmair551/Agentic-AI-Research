# Agentic AI: what you must know (and what to ignore for now)

Goal: say in a standup, "I understood X, so I will implement Y this way."
This is not a course. Each concept: one line, where it lives in OUR agent, when you need it.

## The one idea underneath everything

An LLM is a **stateless function**: messages in, text out. It remembers nothing.
"Agentic" = **code you write around that function**: what you put in, what you do with the
output, and whether you call it again.

| Shape | Who decides the next step | Use when |
|---|---|---|
| **Workflow** | Your code (fixed steps A -> B -> C) | Steps are known in advance. Most of our job. |
| **Agent** | The model, in a loop, using tools | Steps depend on what it finds. Use only where needed. |

Requirement -> stories is a **workflow with at most one small agentic step**. Do not build a free-roaming agent.
Only mandatory reading: Anthropic, "Building effective agents" (~20 min).

## Loop in plain words

A loop = repeat something until it is good or a limit is reached. Our loop: model writes stories -> our code validates them -> if invalid, send the errors back to the model -> retry, at most 2 times. The limit is what makes it safe.

## Concepts, in the order you need them

| # | Concept | One line | Where in our agent |
|---|---|---|---|
| 1 | Model call (API) | Send messages, get a reply. Stateless. | `*.integration.ts` (external system) |
| 2 | Tokens | Unit of size and cost; input + output both count. | Per-run budget |
| 3 | Context window | Max tokens the model sees at once. | What we load per stage |
| 4 | System prompt | Standing role, rules, format. | One versioned prompt file per stage |
| 5 | Context engineering | Choosing what goes into the window. The real skill. | Context pack (product notes, sample stories, DoD) |
| 6 | Structured output | Force JSON matching a schema. | zod story schema (`*.validation.ts`) |
| 7 | Tool calling | Model asks to run a function; YOUR code runs it. | Optional stretch step (free models are unreliable at it) |
| 8 | Loop | Repeat until ok or a limit is hit. | Generate -> validate -> feed errors back -> retry, max 2 |
| 9 | Stop conditions | max steps, tokens, cost, timeout. | Run budget |
| 10 | Validation / guardrails | Plain code checking model output. The model WILL be wrong. | Validate stage |
| 11 | Hooks | Code at fixed points (before tool, after output, on error). | Wrapper around each stage: log, budget, validate |
| 12 | Evals | Fixed inputs with known-good outputs, scored automatically. AI output is non-deterministic, so plain unit tests aren't enough. | Golden set + scorer |
| 13 | Critic / repair loop | Second call checks the first against a rubric; retry max 2. | Quality stage |
| 14 | Human in the loop | Agent drafts, a person approves. | Output is `draft` status |
| 15 | Observability | Log each run: prompt version, tokens, cost, steps, outcome. | Run record |
| 16 | Prompt injection | Text in the requirement tries to give the model orders. Treat input as data. | Prompt delimiting + injection test cases |
| 17 | Provider routing (profiles) | Each task type (fast, generate, tools) has its own ordered list of providers. | Env-configured routes; code only knows profile names |
| 18 | Failover / fallback | If a provider fails (429, 5xx, timeout), try the next one in the list. | Router in the model layer |
| 19 | Circuit breaker (cooldown) | After repeated failures stop calling a provider for a while, then try one request to see if it recovered. | In-memory state, injected clock |
| 20 | Queue + background job | If every provider is down, save the run as `queued` in the DB; a worker retries later. | Atlas + in-process timer (no Redis) |

## Free-tier reality (read once)

- "1M tokens" for Gemini is the **context window** (how much fits in one request), not a free quota. The free quota is separate (requests and tokens per minute/day), differs per model and changes. Check your own dashboard. [Likely]
- Free tiers spike and return 429. That is normal, so failover is a requirement, not a nice-to-have.
- Ollama Cloud free tier: light usage with session and weekly limits and one cloud model at a time, so don't send parallel calls to it. Model names get retired; list current ones before choosing.
- Cheaper than any quota trick: **send less**. Existing items go in as titles only, a capped list, never full descriptions.

## Words you hear that you can postpone

- **Skill**: a packaged folder of instructions loaded on demand. Idea: don't paste everything into every prompt. You may later write one for this agent; no system needed now.
- **Graph / LangGraph**: workflow drawn as nodes, edges, shared state. Know the idea for standups; our flow is a straight line with one retry loop, so plain functions are enough. [Likely]
- **Memory**: history (short term) vs saved facts across runs (long term). V1 needs neither.
- **RAG / embeddings / vector DB**: search over many documents. Skip until context overflows.
- **MCP**: a protocol for exposing tools to agents. Skip.
- **Multi-agent**: adds failure modes. Skip until one agent works.
- **Fine-tuning / training**: not our job.

## Standup lines

- Step 1: "An LLM is stateless, so I'm learning context and structured output. Today: one requirement -> one schema-valid story."
- Step 2: "I separated workflow from agent. Only context-gathering is a tool loop; everything else is fixed steps plus validation."
- Step 3: "AI output isn't deterministic, so I'm building an eval set to measure quality instead of guessing."
- Step 4: "The agent only drafts; a person approves every story."

## Understood-gate

You can (1) draw it, (2) say how it fails, (3) say what it costs, (4) say where it sits in our agent. Can't do (2)? You memorised it, you didn't understand it.

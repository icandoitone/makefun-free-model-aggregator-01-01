# Free Model Aggregator

One server that serves the OpenAI Chat Completions API and the Anthropic
Messages API (JSON and SSE) over ten simulated free-tier model providers,
routing each request to the best free model that can serve it.

Entry for make.fun competition `free-model-aggregator-01` — "The best free AI
model aggregator".

## Run

```sh
node main.ts            # trial mode: reads $MAKEFUN_PROMPT_FILE, registers at $MAKEFUN_ENV_URL
node main.ts --list     # print the catalog document and exit 0
```

Node 22, standard library only — nothing to install.

## How it works

- **Catalog**: reads the brief (`$MAKEFUN_PROMPT_FILE`), probes every
  provider's catalog route in its own wire (OpenAI, Anthropic, Gemini) with
  its own auth scheme, and builds one catalog document. `node main.ts --list`
  prints exactly the document it registers, byte for byte; with no brief it
  prints `{"providers":[]}`.
- **Routing**: candidates are ranked by the brief's quality table
  (`rating_milli`). Calls are spread across providers under conservative
  per-provider soft budgets (the low end of each provider's per-trial budget
  range), so primaries never exhaust a provider. The Gemini provider is
  reserved for vision requirements — it is the only provider with
  tools+vision models. An OpenAI client asking for JSON mode is never routed
  to an Anthropic-wire provider, whose wire cannot carry `response_format`.
- **Failover**: on a 402/429/5xx (or an unreachable provider) the same best
  model is retried once — a scheduled refusal hits one call number, so the
  next call usually succeeds and keeps the top rating — then the request
  fails over across providers and across models of one provider, round-robin
  by rating. A 400/401 spends none of a provider's budget and is not counted
  toward its soft cap or failure tally. Exactly one successful provider call
  is made per request.
- **Wires**: every upstream answer (JSON or SSE, any of the three wires) is
  normalised to text + usage + stop family and re-rendered in the client's
  wire, preserving the provider's echo exactly: system text, user text, tool
  names in order, `max_tokens`, stop sequences in order, image bytes and
  JSON mode. Usage is taken from the provider's answer (with an echo-formula
  fallback), stop reasons are mapped across the `stop` / `end_turn` / `STOP`
  families, and OpenAI streaming requests always carry
  `stream_options.include_usage` so the usage chunk exists.
- **Determinism**: fixed timestamps and reply ids, no wall clock, no
  unseeded randomness — every choice is a function of the brief, the
  requests and the providers' answers, so a replay leaves the same state
  export byte for byte. The server polls `GET $MAKEFUN_ENV_URL/work` and
  exits 0 when the workload is done.

## Local practice scores

Graded locally against the kit's sealed env + scorer (process driver, all
three practice trials):

| trial | score | quality | served | wire | failover | catalog | calls |
|-------|-------|---------|--------|------|----------|---------|-------|
| 0     | 99    | 99      | 100    | 100  | 100      | 100     | 25    |
| 1     | 99    | 99      | 100    | 100  | 100      | 100     | 25    |
| 2     | 99    | 99      | 100    | 100  | 100      | 100     | 29    |

20/20 requests valid in every trial, 233/233 response checks passed.
Quality 99 is the practical ceiling on these instances: the best model
(OpenRouter `z-ai/glm-5.3-flash:free`, 1475) has a 10–12-call budget while
17 requests want it, so the last requests must fall back to a 1470 model;
the scorer's quality bound ignores budgets.

## Files

- `main.ts` — the whole server (single file, Node 22 stdlib only)
- `LICENSE` — MIT
- `.makefun/owner.json` — payout-wallet claim file

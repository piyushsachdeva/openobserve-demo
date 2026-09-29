# Local LLM Chat + Per-User Observability — Demo Guide

Stack: **Ollama** (on the VM) → **LiteLLM Proxy** (adds tracing) → **Open WebUI** (multi-user chat, with a ReelDrop-aware custom model), traces land in **OpenObserve**.


## Prerequisites

- Docker installed locally. If you don't have it:
  ```bash
  brew install --cask docker
  open -a Docker
  ```
- Ollama already running and reachable on the VM (`187.77.187.85:11434` in this demo), with both models pulled:
  ```bash
  curl -X POST http://187.77.187.85:11434/api/pull -d '{"model": "qwen2.5:0.5b"}'
  curl -X POST http://187.77.187.85:11434/api/pull -d '{"model": "qwen2.5:1.5b"}'
  ```
  `qwen2.5:0.5b` is the "look how small this is" model for the install/platform tour. `qwen2.5:1.5b` is what the ReelDrop support model actually uses — see the gotcha in Step 3 for why size matters here.

## Step 1 — Start OpenObserve

Run it as a standalone binary instead — download from https://openobserve.ai/downloads, then:

```bash
chmod +x openobserve
ZO_ROOT_USER_EMAIL="piyush@techtutorialswithpiyush.com" ZO_ROOT_USER_PASSWORD="Demo@Observe123" ./openobserve
```

## Step 2 — Get your OpenObserve auth token

```bash
echo -n 'root@example.com:Complexpass#123' | base64
```

Save the output — you'll need it as `Basic <token>`.

## Step 3 — Start LiteLLM Proxy (this is what adds tracing)

LiteLLM is an open-source AI gateway and proxy server that gives developers a single, OpenAI-compatible API to call over 100 large language model (LLM) providers

Docs: https://docs.litellm.ai/docs/observability/opentelemetry_integration and https://docs.litellm.ai/docs/providers/ollama

`litellm-config.yaml` (already in this repo):

```yaml
model_list:
  - model_name: qwen2.5:0.5b
    litellm_params:
      model: ollama_chat/qwen2.5:0.5b
      api_base: http://187.77.187.85:11434
  - model_name: qwen2.5:1.5b
    litellm_params:
      model: ollama_chat/qwen2.5:1.5b
      api_base: http://187.77.187.85:11434
      num_ctx: 8192
      temperature: 0

litellm_settings:
  callbacks: ["otel"]

general_settings:
  user_header_mappings:
    - header_name: X-OpenWebUI-User-Id
      litellm_user_role: internal_user
    - header_name: X-OpenWebUI-User-Email
      litellm_user_role: customer
```

The `user_header_mappings` block is what makes traces per-user: it tells LiteLLM to read the Open WebUI user's id/email off request headers and attach them to every trace.

**Gotcha, found the hard way:** Ollama loads a model with a **4096-token context window by default**, regardless of what the model actually supports (`qwen2.5:1.5b` supports up to 32k). The ReelDrop support system prompt (full product docs, ~4,300 tokens) was silently overflowing that default window, and the model would respond with confident, well-formatted, **completely fabricated** answers — wrong prices, invented features, no error of any kind. `num_ctx: 8192` on the `qwen2.5:1.5b` entry above is what fixes it. Verify the running context window any time with:
```bash
curl http://187.77.187.85:11434/api/ps
```
Look for `"context_length"` in the response — if it's `4096`, the fix didn't take (usually because the model was already loaded before you restarted the proxy; unload it first with `curl -X POST http://187.77.187.85:11434/api/generate -d '{"model": "qwen2.5:1.5b", "keep_alive": 0}'` and let the next request reload it).

**Second gotcha, found only after testing through the real Open WebUI chat (not curl):** setting "Temperature: Custom: 0.1" in Open WebUI's own model editor **does nothing** — traced requests confirmed Open WebUI never sends a `temperature` field to the API at all, so Ollama silently used its own high default regardless of what the UI showed. This is why curl tests against the proxy (which *did* include `temperature` in the payload) looked perfectly reliable while the actual chat kept flip-flopping between a correct answer and a fabricated one on the identical question. The fix has to live server-side — `temperature: 0` in the `litellm_params` above, same as `num_ctx` — so it applies regardless of what Open WebUI does or doesn't send.

**Third gotcha (current images):** `litellm:main-latest` now refuses to boot unless a master key is set, and it rejects weak/publicly-known values like `sk-1234` outright (`UnsafeMasterKeyError`). Generate one once and reuse it for both containers:

```bash
echo "LITELLM_MASTER_KEY=sk-$(openssl rand -hex 32)" > .env && chmod 600 .env
source .env
```

Two related traps: the `-v` mount must be run from the directory that actually holds `litellm-config.yaml` (this one) — from anywhere else Docker silently creates an empty *directory* at the source path and the proxy dies with `IsADirectoryError: Is a directory: '/app/config.yaml'`. And the OTel header needs the literal `Basic ` prefix before the base64 token, or traces never reach OpenObserve.

Run it (after `source .env`):

```bash
docker run -d --name litellm-proxy \
  -p 4000:4000 \
  -v $PWD/litellm-config.yaml:/app/config.yaml \
  -e LITELLM_MASTER_KEY="$LITELLM_MASTER_KEY" \
  -e OLLAMA_API_BASE=http://187.77.187.85:11434 \
  -e OTEL_EXPORTER=otlp_http \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://host.docker.internal:5080/api/default \
  -e OTEL_EXPORTER_OTLP_PROTOCOL=http/json \
  -e OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <token from Step 2>" \
  -e OTEL_SERVICE_NAME=litellm-proxy \
  ghcr.io/berriai/litellm:main-latest \
  --config /app/config.yaml --port 4000
```

(On Linux, replace `host.docker.internal` with your machine's LAN IP.)

## Step 4 — Start Open WebUI, pointed at LiteLLM Proxy

```bash
docker run -d --name open-webui \
  -p 3001:8080 \
  -e OPENAI_API_BASE_URL=http://host.docker.internal:4000/v1 \
  -e OPENAI_API_KEY="$LITELLM_MASTER_KEY" \
  -e ENABLE_FORWARD_USER_INFO_HEADERS=True \
  -v open_webui_data:/app/backend/data \
  ghcr.io/open-webui/open-webui:main
```

`ENABLE_FORWARD_USER_INFO_HEADERS=True` is required — without it, Open WebUI never sends the user's id/email, and every trace stays anonymous.

## Step 5 — Give it ReelDrop's knowledge (the "customer support agent" part)

The goal: a chat model that only answers from ReelDrop's actual docs/pricing/features, so it can act as a support agent for the demo.

**What worked:** paste the docs directly into the model's **System Prompt**, verbatim, as one block. At ~17KB / ~4,300 tokens this comfortably fits `qwen2.5:1.5b`'s 8192-token context (once fixed per the Step 3 gotcha) with room left for the question and answer.

1. Fetch and clean the source pages (already done — see `reeldrop-knowledge/*.md` in this repo: `pricing.md`, `features.md`, `docs-scheduling-and-account.md`, `docs-dm-automation.md`, `docs-mcp-claude.md`).
2. Open WebUI → **Workspace → Models → Create**.
   - Name: `ReelDrop Support`
   - Base Model: `qwen2.5:1.5b`
   - System Prompt: an instruction line ("Answer ONLY using the documentation below...") followed by the full contents of all 5 files concatenated.
   - Temperature: leave on Default — it's enforced server-side (see the second gotcha in Step 3), the model editor's own setting doesn't reach the API.
   - **Capabilities → uncheck "Builtin Tools".** Left on (the default), every real chat request silently attaches ~35 tool/function schemas — memory, calendar, notes, automations, knowledge search — none of which this demo uses. A model this small can't hold that plus the ReelDrop docs plus a straight answer at the same time; it degrades into fabricated answers ("up to 50 accounts", "free") indistinguishable from the context-window bug except that curl tests never reproduce it, because curl never sends those tool schemas either. Confirmed by inspecting the actual traced request: `llm_request_functions_0` through `llm_request_functions_34` were present with Builtin Tools on, gone with it off.
   - Do **not** attach a Knowledge collection to this model (see above — it doesn't help and adds risk of confusing the small model further).
3. Save. Select the model in a **new** chat (capability changes don't apply retroactively to already-open chats) and ask something the docs actually cover, e.g. *"How many Instagram accounts can I connect on the Creator plan, and what does it cost?"* — correct answer: **up to 5 accounts, $29/month or $149/year**. Verified twice in a row through the actual browser chat, on two different questions, after all three fixes above.

If you want to demo a *stale knowledge base* failure mode instead of a working support agent, that's a separate, deliberate scenario — see the git history for the `openobserve-demo` repo's earlier support-widget prototype (not part of this stack) for how that was built with a genuinely conflicting pair of documents.

## Verify

1. Open http://localhost:3001, sign in, chat with `qwen2.5:0.5b` for the "look how fast/small this is" beat, then switch to `ReelDrop Support` for the actual demo questions.
2. Open **Traces** and select stream `default` — it does not auto-select, and the blank
   picker looks identical to "no data". Deep link that skips it:
   `http://localhost:5080/web/traces?stream=default&period=15m&org_identifier=default&tab=traces`
3. Click a trace, then the `litellm_request` span — model name, prompt/completion token
   counts, latency, the end user's email, and the full prompt and completion. (Cost is
   present as a field but is always `0` for local models — see the monitoring section.)
4. Open **AI → Insights** for the LLM dashboard: total tokens, trace count, P95 latency,
   error rate.
5. Open a second browser session (different Open WebUI account) and confirm its requests show up as separate traces — this is the "per-user session" observability piece.

## Monitoring customer interactions in OpenObserve

This is the actual point of the demo: not just that the bot answers correctly, but that
every interaction is fully observable — tokens, speed, cache behaviour, per user.

Everything below was run against this stack and reflects what actually renders. Where a
panel is empty, that is stated rather than glossed over.

### Before you start: the two things that make the UI look broken

**1. The trace stream does not auto-select.** Traces opens on "Pick a stream to start
exploring" with a blank picker, which reads exactly like "no data arrived". Open the
dropdown at the top of the left panel and choose `default`. Mouse clicks on the option
can fail to register — click the dropdown, press **↓**, then **Enter**. Better still,
skip the picker entirely with a deep link:

```
http://localhost:5080/web/traces?stream=default&period=15m&org_identifier=default&tab=traces
```

**2. The Streams page reports `0` documents** for `default` long after ingestion is
working, because those stats only refresh when OpenObserve flushes WAL to disk. Never
judge ingestion by that page. To check ingestion for real:

```bash
curl -s -u 'itzmario133@gmail.com:Demo@Observe123' -H 'Content-Type: application/json' \
  -X POST 'http://localhost:5080/api/default/_search?type=traces' \
  -d '{"query":{"sql":"SELECT count(*) FROM \"default\"","start_time":1750000000000000,"end_time":1800000000000000,"from":0,"size":1}}'
```

### Demo beat 1 — the AI Observability dashboard (open here)

**AI → Insights** is the purpose-built LLM dashboard and the strongest opening visual.
Deep link (set the range wide enough that your demo traffic is inside it):

```
http://localhost:5080/web/ai/llm-insights?org_identifier=default&period=6h&type=stream&stream=default
```

Five hero tiles across the top: **Total Cost**, **Total Tokens**, **Total Traces**,
**P95 Latency**, **Error Rate**. On a demo run of a dozen chat turns this showed
15.3K tokens / 28 traces / 1.6min P95 / 0.0% errors.

Two caveats worth knowing before you present it, so a blank panel doesn't catch you out:

- **Total Cost will read $0.00, always.** See "The cost tile is a zero" below.
- **The per-model panels render "No Data"** — Cost trend, Token trend, Span trend,
  Latency by model, Spans by model, Tokens by model. The attributes they need *are*
  present and correct on the spans (`gen_ai_response_model` is populated with
  `qwen2.5:0.5b` / `qwen2.5:1.5b`), so this is a dashboard-side mismatch, not missing
  instrumentation. **Requests over time** and **Errors over time** do populate.

If you want per-model breakdowns live in the demo, drive them from the SQL in beat 4
rather than these panels.

### Demo beat 2 — one request, end to end

Traces → stream `default` → click any trace. The waterfall has seven span types; these
are the two worth opening:

| `operation_name` | What it carries |
|---|---|
| `litellm_request` | The OTel Gen-AI semantic attributes — model, tokens, user, full prompt and completion |
| `raw_gen_ai_request` | Ollama's own native timings — generation speed, prefill, prompt-cache hits |

The others (`Received Proxy Server Request`, `self`, `router`, `auth`,
`proxy_pre_call`) are LiteLLM's internal plumbing and are mostly useful for showing
where proxy overhead sits versus model time.

On a `litellm_request` span, open **Attributes**:

- `gen_ai_request_model` / `gen_ai_response_model` — which model served the turn
- `gen_ai_usage_input_tokens`, `gen_ai_usage_output_tokens`, `gen_ai_usage_total_tokens`
- `metadata_user_api_key_end_user_id` — the end user's email
  (note: `metadata_user_api_key_user_email` exists but is empty — use `end_user_id`)
- `gen_ai_input_messages` / `gen_ai_output_messages` — the **full prompt and completion**,
  structured as OTel Gen-AI message parts

That last pair is the highest-impact thing to show. You can read the entire system
prompt the ReelDrop model was given, including the retrieved documentation, and the
exact answer it produced — from the trace alone, with no logging added to the app. It is
also how you'd debug a wrong answer: you can see whether the model was given the right
docs or hallucinated past them.

### Demo beat 3 — generation speed and the prefill cliff

The `raw_gen_ai_request` span carries Ollama's native timings. **These fields are
ingested as strings**, so arithmetic on them needs an explicit `CAST` — without it the
query fails with "Search SQL execute error" and returns nothing.

Durations are in **nanoseconds**. Tokens/sec = `eval_count / (eval_duration / 1e9)`.

```sql
SELECT llm_ollama_chat_model AS model,
       CAST(llm_ollama_chat_eval_count AS DOUBLE) AS output_tokens,
       ROUND(CAST(llm_ollama_chat_eval_count AS DOUBLE)
             / (CAST(llm_ollama_chat_eval_duration AS DOUBLE) / 1e9), 2) AS tokens_per_second,
       ROUND(CAST(llm_ollama_chat_prompt_eval_duration AS DOUBLE) / 1e9, 2) AS prefill_seconds
FROM "default"
WHERE llm_ollama_chat_eval_duration IS NOT NULL
  AND llm_ollama_chat_eval_duration != '0'
ORDER BY _timestamp DESC
```

Filtering on the field rather than `operation_name` is deliberate — it picks up the
timing spans without you having to know which operation name carries them.

The aggregate is the better slide, because it quantifies the model trade-off the demo
opens with:

```sql
SELECT llm_ollama_chat_model AS model,
       count(*) AS calls,
       ROUND(AVG(CAST(llm_ollama_chat_eval_count AS DOUBLE)
                 / (CAST(llm_ollama_chat_eval_duration AS DOUBLE) / 1e9)), 1) AS avg_tokens_per_sec,
       ROUND(AVG(CAST(llm_ollama_chat_prompt_eval_duration AS DOUBLE) / 1e9), 2) AS avg_prefill_sec,
       ROUND(MAX(CAST(llm_ollama_chat_prompt_eval_duration AS DOUBLE) / 1e9), 2) AS worst_prefill_sec
FROM "default"
WHERE llm_ollama_chat_eval_duration IS NOT NULL
  AND llm_ollama_chat_eval_duration != '0'
GROUP BY model
```

Measured on this stack:

| model | calls | avg tokens/sec | avg prefill | worst prefill |
|---|---|---|---|---|
| `qwen2.5:0.5b` | 7 | 44.1 | 0.79s | 1.50s |
| `qwen2.5:1.5b` | 4 | 32.2 | 12.62s | 23.71s |

The headline is not the 44 vs 32 tokens/sec. It is that the bigger model's **prefill**
is 16× worse — and prefill is the part the user experiences as "it's frozen". Generation
speed is barely different; time-to-first-token is what the observability actually
exposes. This is a point you cannot make from a latency number alone.

### Demo beat 4 — why the second question is fast (prompt caching)

Same model, same prompt size, wildly different latency — and the trace explains it:

```sql
SELECT llm_ollama_chat_model AS model,
       CAST(llm_ollama_chat_prompt_eval_count AS DOUBLE) AS prompt_tokens,
       CAST(llm_ollama_chat_prompt_eval_cached_count AS DOUBLE) AS cached_tokens,
       ROUND(100.0 * CAST(llm_ollama_chat_prompt_eval_cached_count AS DOUBLE)
             / CAST(llm_ollama_chat_prompt_eval_count AS DOUBLE), 1) AS cache_hit_pct,
       ROUND(CAST(llm_ollama_chat_prompt_eval_duration AS DOUBLE) / 1e9, 2) AS prefill_sec
FROM "default"
WHERE llm_ollama_chat_prompt_eval_cached_count IS NOT NULL
ORDER BY _timestamp DESC
```

Two rows from an actual run, both `qwen2.5:0.5b`, both 288 prompt tokens:

| cached | cache hit | prefill |
|---|---|---|
| 287 / 288 | 99.7% | **0.03s** |
| 0 / 288 | 0.0% | **1.50s** |

A 50× swing in time-to-first-token, fully explained by one attribute. Ask the ReelDrop
model two questions in a row during the demo and show this table — the first is slow,
the second is fast, and the trace says exactly why. This is the single best argument in
the demo for tracing an LLM app rather than just timing it.

### Demo beat 5 — per-user attribution, and the traffic nobody counted

Dashboards → New dashboard → Add panel → chart type **Table** → Stream Type `traces`,
Stream `default` → **Custom** (next to SQL/Builder) → paste:

```sql
SELECT metadata_user_api_key_end_user_id AS user_email,
       gen_ai_request_model AS model,
       count(*) AS requests,
       sum(gen_ai_usage_input_tokens) AS input_tokens,
       sum(gen_ai_usage_output_tokens) AS output_tokens,
       sum(gen_ai_usage_total_tokens) AS total_tokens
FROM "default"
WHERE gen_ai_request_model IS NOT NULL
GROUP BY user_email, model
```

**Do not skip the field mapping, and note that it starts wrong.** A new panel ships with
placeholder axes — **First Column** = `x_axis_1` (`histogram(_timestamp)`) and
**Other Columns** = `y_axis_1` (`count(_timestamp)`). Your custom SQL returns
`user_email`, `model`, `requests`, … and *none* of those placeholder columns, so the
panel renders blank cells and looks exactly like "no data arrived".

The tell that it is a mapping problem and not an ingestion or time-range problem: the
table footer still reads **"1-3 of 3"**. The rows came back; the columns are pointing at
a field that isn't in the result. A genuinely empty result says "No Data" instead.

So: click the **×** on `x_axis_1` and on `y_axis_1` first, then drag from **Query Fields**
(the left panel already lists your SQL's real columns) into **First Column**
(`user_email`) and **Other Columns** (`model`, `requests`, `input_tokens`,
`output_tokens`, `total_tokens`). Then **Apply → Save**.

Also widen the time range before judging the result — the dashboard default is
**Past 15 Minutes**, and demo traffic from even half an hour ago falls outside it.
Verified on this stack: the same query returns 0 rows at 15 minutes and 3 rows at
60 minutes.

(Dropping `sum(gen_ai_usage_cost)` from this query is deliberate — see below. A column of
zeros undercuts the panel.)

Now the finding that makes this panel interesting. Some rows come back with a **null**
user — and they are not noise:

```sql
SELECT CASE WHEN metadata_user_api_key_end_user_id IS NULL
            THEN 'background (no user attributed)'
            ELSE metadata_user_api_key_end_user_id END AS attributed_to,
       count(*) AS calls,
       sum(gen_ai_usage_total_tokens) AS tokens
FROM "default"
WHERE operation_name = 'litellm_request'
GROUP BY attributed_to
```

| attributed to | calls | tokens |
|---|---|---|
| `piyush.sachdeva055@gmail.com` | 12 | 11,166 |
| background (no user attributed) | 2 | 4,235 |

**Two background calls burned 27% of all tokens.** These are Open WebUI's own automatic
LLM calls — chat-title generation, tag generation, follow-up suggestions — fired without
the user headers that carry attribution. Inspect one and you can see it directly:

```sql
SELECT substr(gen_ai_output_messages, 1, 120) AS output_preview,
       gen_ai_usage_input_tokens
FROM "default"
WHERE operation_name = 'litellm_request'
  AND metadata_user_api_key_end_user_id IS NULL
```

This is the strongest closing beat in the demo. The point of AI observability is not
confirming the numbers you expected — it is finding the LLM traffic your product is
generating that nobody budgeted for, and which no per-user dashboard would have shown
you. Two invisible calls, 2,050 input tokens each, on every single chat.

### The cost tile is a zero — say so up front

`gen_ai_usage_cost` is `0` on every span, so **Total Cost** reads `$0.00` and any
`sum(gen_ai_usage_cost)` column is a row of zeros.

LiteLLM has no price book for local `ollama_chat` models, and — verified on this stack —
adding `input_cost_per_token` / `output_cost_per_token` does **not** fix it, under either
`litellm_params` or `model_info`. The proxy restarts cleanly and still emits zero.

Handle it one of two ways:

- **Recommended:** frame the demo around tokens, latency, and cache behaviour, all of
  which are real here, and say plainly that cost is zero because the model is local —
  which is itself the point of running it locally.
- If cost has to be non-zero, point one model in `litellm-config.yaml` at a hosted
  provider LiteLLM already prices (any OpenAI or Anthropic model), and demo cost on that
  model only.

Do not promise a cost number you have not seen render.

### AI → Sessions is empty, and why

**AI → Sessions** ("Browse and inspect agent sessions turn by turn") shows
"No LLM sessions yet". It groups by `gen_ai_conversation_id`, and LiteLLM is not emitting
that attribute in this setup — the field exists in the schema but is null on every span.
Wiring Open WebUI's chat id through to it has not been verified here, so treat Sessions
as out of scope for the demo rather than something to click into and discover live.

### Two-user demo

Open a second browser session with a different Open WebUI account and re-run the
per-user table from beat 5. The two emails appear as separate rows with their own token
totals. Worth doing — the per-user split is more convincing as a live result than as a
single-row table.

## Known gaps 
- SLOs are missing from this OpenObserve build's UI (see Step 1).
- `gen_ai_usage_cost` is always `0` for local Ollama models, and custom per-token pricing
  does not change it — verified under both `litellm_params` and `model_info`. The
  **Total Cost** tile reads `$0.00`.
- **AI → Sessions** is empty: it groups by `gen_ai_conversation_id`, which LiteLLM does
  not emit here.
- The per-model panels on **AI → Insights** (Token trend, Latency by model, Tokens by
  model, Spans by model, Cost trend, Span trend) render "No Data" even though
  `gen_ai_response_model` is correctly populated on the spans. Use the SQL in the
  monitoring section for per-model views.
- Ollama's native timing fields (`llm_ollama_chat_*`) ingest as **strings**. Arithmetic
  on them without `CAST(... AS DOUBLE)` fails with "Search SQL execute error" and
  returns zero rows rather than an obvious error.
- The `ReelDrop Support` model's accuracy on questions the docs don't cover is honest but blunt — it says it doesn't know rather than guessing (by design, from the system prompt).
- Running on the VM's CPU, prompt prefill on the first request after a model reload can take 20–30 seconds even after the context-window fix — subsequent requests are fast because Ollama caches the prompt prefix (`llm_ollama_chat_prompt_eval_cached_count` in the trace shows this).
- Don't trust a curl test against the LiteLLM proxy as proof the chat model works end-to-end — it isn't the same request Open WebUI actually sends (see the temperature and Builtin Tools gotchas above). The only real verification is asking the question in the actual browser chat, in a genuinely new chat, more than once.

Reference docs used in this guide:
- OpenObserve quickstart: https://openobserve.ai/docs/quickstart/
- OpenObserve AI Gateways (LiteLLM Proxy): https://openobserve.ai/docs/integration/ai/gateways/litellm-proxy/
- OpenObserve No-Code AI (Open WebUI): https://openobserve.ai/docs/integration/ai/no-code/openwebui/
- LiteLLM OpenTelemetry integration: https://docs.litellm.ai/docs/observability/opentelemetry_integration
- LiteLLM Ollama provider: https://docs.litellm.ai/docs/providers/ollama
- Ollama context length / `num_ctx`: https://github.com/ollama/ollama/blob/main/docs/faq.md

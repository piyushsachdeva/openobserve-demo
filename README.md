# Local LLM Chat with Per-User Observability

A self-hosted stack that runs a local LLM chat app and traces every request — tokens, latency, prompt cache behaviour, and the end user behind each call.

**Ollama** (local models) → **LiteLLM Proxy** (OpenAI-compatible gateway, emits OpenTelemetry traces) → **Open WebUI** (multi-user chat UI), with traces stored and queried in **OpenObserve**.

The repo also includes a sample knowledge base (`reeldrop-knowledge/`) used to build a documentation-grounded support chatbot, so you can trace a realistic RAG-style workload instead of empty prompts.

## Placeholders

Replace these throughout before running:

| Placeholder | Meaning |
|---|---|
| `<OLLAMA_HOST>` | Host/IP where Ollama is reachable, e.g. `localhost` or your VM's IP |
| `<YOUR_EMAIL>` | OpenObserve root user email |
| `<YOUR_PASSWORD>` | OpenObserve root password (must satisfy its policy: upper, lower, digit, special char) |
| `<BASE64_TOKEN>` | Base64 of `<YOUR_EMAIL>:<YOUR_PASSWORD>` (see Step 2) |

## Prerequisites

- Docker
  ```bash
  # macOS
  brew install --cask docker && open -a Docker
  ```
- Ollama running and reachable at `http://<OLLAMA_HOST>:11434`, with both models pulled:
  ```bash
  curl -X POST http://<OLLAMA_HOST>:11434/api/pull -d '{"model": "qwen2.5:0.5b"}'
  curl -X POST http://<OLLAMA_HOST>:11434/api/pull -d '{"model": "qwen2.5:1.5b"}'
  ```
  `qwen2.5:0.5b` is the small/fast model. `qwen2.5:1.5b` backs the support chatbot — see the context-window note in Step 3 for why size matters.

## Alternative: Docker Compose

`docker-compose.yml` runs all three services (OpenObserve, LiteLLM Proxy, Open WebUI) together. Copy `.env.example` to `.env`, fill in the values, set `<OLLAMA_HOST>` in `docker-compose.yml` and `litellm-config.yaml`, then:

```bash
docker compose up -d
```

Note the base64 token in `.env` depends on the OpenObserve credentials in the same file, so set those first. The step-by-step path below is the same stack run manually — use it if you want to see each piece start, or if you already run OpenObserve as a binary.

## Step 1 — Start OpenObserve

Download the binary from https://openobserve.ai/downloads, then:

```bash
chmod +x openobserve
ZO_ROOT_USER_EMAIL="<YOUR_EMAIL>" ZO_ROOT_USER_PASSWORD="<YOUR_PASSWORD>" ./openobserve
```

UI at http://localhost:5080.

## Step 2 — Get your OpenObserve auth token

```bash
echo -n '<YOUR_EMAIL>:<YOUR_PASSWORD>' | base64
```

Use the output as `<BASE64_TOKEN>`. The OTel header needs the literal `Basic ` prefix in front of it, or traces never arrive.

## Step 3 — Start LiteLLM Proxy

LiteLLM is an open-source AI gateway that exposes 100+ LLM providers behind one OpenAI-compatible API — and emits OpenTelemetry traces for every call, which is what makes this stack observable.

`litellm-config.yaml` (in this repo) — set `api_base` to your Ollama host:

```yaml
model_list:
  - model_name: qwen2.5:0.5b
    litellm_params:
      model: ollama_chat/qwen2.5:0.5b
      api_base: http://<OLLAMA_HOST>:11434
  - model_name: qwen2.5:1.5b
    litellm_params:
      model: ollama_chat/qwen2.5:1.5b
      api_base: http://<OLLAMA_HOST>:11434
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

`user_header_mappings` is what makes traces per-user: LiteLLM reads the Open WebUI user's id/email off the request headers and attaches them to every trace.

Generate a master key (current LiteLLM images refuse to boot without one, and reject weak values like `sk-1234` with `UnsafeMasterKeyError`):

```bash
echo "LITELLM_MASTER_KEY=sk-$(openssl rand -hex 32)" > .env && chmod 600 .env
source .env
```

Run the proxy from this directory (the `-v` mount must resolve against the directory holding `litellm-config.yaml`; run it elsewhere and Docker creates an empty *directory* at the source path, and the proxy dies with `IsADirectoryError: Is a directory: '/app/config.yaml'`):

```bash
docker run -d --name litellm-proxy \
  -p 4000:4000 \
  -v $PWD/litellm-config.yaml:/app/config.yaml \
  -e LITELLM_MASTER_KEY="$LITELLM_MASTER_KEY" \
  -e OLLAMA_API_BASE=http://<OLLAMA_HOST>:11434 \
  -e OTEL_EXPORTER=otlp_http \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://host.docker.internal:5080/api/default \
  -e OTEL_EXPORTER_OTLP_PROTOCOL=http/json \
  -e OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <BASE64_TOKEN>" \
  -e OTEL_SERVICE_NAME=litellm-proxy \
  ghcr.io/berriai/litellm:main-latest \
  --config /app/config.yaml --port 4000
```

On Linux, replace `host.docker.internal` with your machine's LAN IP.

### Context window: set `num_ctx` explicitly

Ollama loads a model with a **4096-token context window by default**, regardless of what the model supports (`qwen2.5:1.5b` supports up to 32k). A large system prompt (the knowledge base here is ~4,300 tokens) silently overflows that window, and the model returns confident, well-formatted, **fabricated** answers with no error at all. `num_ctx: 8192` in the config above fixes it.

Verify the running context window:

```bash
curl http://<OLLAMA_HOST>:11434/api/ps
```

If `context_length` is still `4096`, the model was already loaded before the proxy restarted. Unload it and let the next request reload it:

```bash
curl -X POST http://<OLLAMA_HOST>:11434/api/generate \
  -d '{"model": "qwen2.5:1.5b", "keep_alive": 0}'
```

### Temperature must be set server-side

Open WebUI's per-model temperature setting does **not** reach the API — traced requests confirm it never sends a `temperature` field, so Ollama uses its own default no matter what the UI shows. Set `temperature: 0` in `litellm_params` (as above) so it applies regardless.

## Step 4 — Start Open WebUI

```bash
docker run -d --name open-webui \
  -p 3001:8080 \
  -e OPENAI_API_BASE_URL=http://host.docker.internal:4000/v1 \
  -e OPENAI_API_KEY="$LITELLM_MASTER_KEY" \
  -e ENABLE_FORWARD_USER_INFO_HEADERS=True \
  -v open_webui_data:/app/backend/data \
  ghcr.io/open-webui/open-webui:main
```

`ENABLE_FORWARD_USER_INFO_HEADERS=True` is required — without it Open WebUI never sends the user's id/email and every trace stays anonymous.

Open http://localhost:3001 and create an account.

## Step 5 — Create the documentation-grounded chatbot

Goal: a model that answers only from the supplied docs. The knowledge base in `reeldrop-knowledge/` (`pricing.md`, `features.md`, `docs-scheduling-and-account.md`, `docs-dm-automation.md`, `docs-mcp-claude.md`) is sample content — swap in your own.

What worked: paste the docs directly into the model's **System Prompt**, verbatim, as one block (~17KB / ~4,300 tokens fits the 8192-token window with room for the question and answer). A Knowledge collection was not needed and did not help with a model this small.

1. Open WebUI → **Workspace → Models → Create**.
2. Name it (e.g. `Docs Support`), Base Model `qwen2.5:1.5b`.
3. System Prompt: an instruction line (`Answer ONLY using the documentation below...`) followed by the concatenated contents of all knowledge files.
4. Temperature: leave on Default — it's enforced server-side (see Step 3).
5. **Capabilities → uncheck "Builtin Tools".** Left enabled, every chat request silently attaches ~35 tool/function schemas (memory, calendar, notes, automations, knowledge search). A small model cannot hold those plus the docs plus a straight answer, and degrades into fabricated output. Confirmed from traces: `llm_request_functions_0` … `llm_request_functions_34` present with Builtin Tools on, gone with it off.
6. Save, then select the model in a **new** chat — capability changes do not apply to already-open chats.

A `curl` test against the proxy is **not** proof the chat works end-to-end: curl doesn't send the tool schemas and does send `temperature`, so it can pass while the real chat fabricates answers. Verify in the browser chat, in a new chat, more than once.

## Verify

1. Chat with `qwen2.5:0.5b`, then with your docs model, and ask a question the docs cover.
2. Open **Traces** and select stream `default` — it does **not** auto-select, and the blank picker looks identical to "no data". Deep link that skips the picker:
   ```
   http://localhost:5080/web/traces?stream=default&period=15m&org_identifier=default&tab=traces
   ```
   If mouse clicks on the dropdown option don't register, click the dropdown, press **↓**, then **Enter**.
3. Click a trace → the `litellm_request` span: model, token counts, latency, end-user email, full prompt and completion.
4. **AI → Insights** for the LLM dashboard: total tokens, trace count, P95 latency, error rate.
5. Sign in as a second Open WebUI user and confirm its requests appear as separate traces.

**The Streams page reports `0` documents** for `default` long after ingestion works — those stats only refresh when OpenObserve flushes WAL to disk. Don't judge ingestion by that page; check it directly:

```bash
curl -s -u '<YOUR_EMAIL>:<YOUR_PASSWORD>' -H 'Content-Type: application/json' \
  -X POST 'http://localhost:5080/api/default/_search?type=traces' \
  -d '{"query":{"sql":"SELECT count(*) FROM \"default\"","start_time":1750000000000000,"end_time":1800000000000000,"from":0,"size":1}}'
```

## What the traces show

### The AI Insights dashboard

```
http://localhost:5080/web/ai/llm-insights?org_identifier=default&period=6h&type=stream&stream=default
```

Hero tiles: Total Cost, Total Tokens, Total Traces, P95 Latency, Error Rate. **Requests over time** and **Errors over time** populate; the per-model panels (Cost trend, Token trend, Span trend, Latency/Spans/Tokens by model) render "No Data" even though `gen_ai_response_model` is correctly populated on the spans. Use the SQL below for per-model views.

### One request, end to end

A trace has seven span types. Two carry the useful data:

| `operation_name` | What it carries |
|---|---|
| `litellm_request` | OTel Gen-AI attributes — model, tokens, user, full prompt and completion |
| `raw_gen_ai_request` | Ollama's native timings — generation speed, prefill, prompt-cache hits |

The rest (`Received Proxy Server Request`, `self`, `router`, `auth`, `proxy_pre_call`) are LiteLLM internals, useful for seeing proxy overhead versus model time.

Key attributes on a `litellm_request` span:

- `gen_ai_request_model` / `gen_ai_response_model` — which model served the turn
- `gen_ai_usage_input_tokens`, `gen_ai_usage_output_tokens`, `gen_ai_usage_total_tokens`
- `metadata_user_api_key_end_user_id` — the end user's email (`metadata_user_api_key_user_email` exists but is empty)
- `gen_ai_input_messages` / `gen_ai_output_messages` — the **full prompt and completion**, as OTel Gen-AI message parts

That last pair lets you read the entire system prompt the model was given and the exact answer it produced, from the trace alone, with no logging added to the app — which is also how you debug a wrong answer.

### Generation speed and prefill

Ollama's native timing fields ingest as **strings**, so arithmetic needs an explicit `CAST` — without it the query fails with "Search SQL execute error" and returns no rows. Durations are in **nanoseconds**.

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

Example output from a CPU-only run:

| model | calls | avg tokens/sec | avg prefill | worst prefill |
|---|---|---|---|---|
| `qwen2.5:0.5b` | 7 | 44.1 | 0.79s | 1.50s |
| `qwen2.5:1.5b` | 4 | 32.2 | 12.62s | 23.71s |

Generation speed differs little between the two models; **prefill** — the part users experience as "it's frozen" — is 16× worse on the larger one. That's visible from traces and invisible in a single latency number.

### Prompt caching

Same model, same prompt size, very different latency:

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

Two rows from one run, both `qwen2.5:0.5b`, both 288 prompt tokens:

| cached | cache hit | prefill |
|---|---|---|
| 287 / 288 | 99.7% | **0.03s** |
| 0 / 288 | 0.0% | **1.50s** |

A 50× swing in time-to-first-token, explained by one attribute. Ask two questions in a row and the second is fast for exactly this reason.

### Per-user attribution

Dashboards → New dashboard → Add panel → chart type **Table** → Stream Type `traces`, Stream `default` → **Custom** (next to SQL/Builder) → paste:

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

**You must fix the field mapping, and it starts wrong.** A new panel ships with placeholder axes — **First Column** = `x_axis_1` (`histogram(_timestamp)`) and **Other Columns** = `y_axis_1` (`count(_timestamp)`). Custom SQL returns none of those columns, so the panel renders blank cells that look exactly like "no data".

The tell that it's a mapping problem, not ingestion or time range: the table footer still reads "1-3 of 3" — the rows came back, the columns point at fields that aren't in the result. A genuinely empty result says "No Data".

So: click the **×** on `x_axis_1` and `y_axis_1`, then drag your SQL's real columns from **Query Fields** into **First Column** (`user_email`) and **Other Columns** (`model`, `requests`, `input_tokens`, `output_tokens`, `total_tokens`). Then **Apply → Save**.

Also widen the time range — the dashboard default is **Past 15 Minutes**, so traffic from half an hour ago falls outside it.

### Unattributed background traffic

Some rows come back with a **null** user, and they aren't noise:

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
| `<YOUR_EMAIL>` | 12 | 11,166 |
| background (no user attributed) | 2 | 4,235 |

Two background calls accounted for 27% of all tokens. These are Open WebUI's own automatic LLM calls — chat-title generation, tag generation, follow-up suggestions — fired without the user headers that carry attribution. Inspect one:

```sql
SELECT substr(gen_ai_output_messages, 1, 120) AS output_preview,
       gen_ai_usage_input_tokens
FROM "default"
WHERE operation_name = 'litellm_request'
  AND metadata_user_api_key_end_user_id IS NULL
```

This is the kind of thing per-user dashboards alone never surface: LLM traffic the product generates on its own, roughly 2,050 input tokens per chat, that nobody budgeted for.

## Known limitations

- **Cost is always `$0.00`.** `gen_ai_usage_cost` is `0` on every span because LiteLLM has no price book for local `ollama_chat` models. Adding `input_cost_per_token` / `output_cost_per_token` does **not** fix it, under either `litellm_params` or `model_info`. To show real cost, point one model in `litellm-config.yaml` at a hosted provider LiteLLM prices (any OpenAI or Anthropic model).
- **AI → Sessions is empty.** It groups by `gen_ai_conversation_id`, which LiteLLM does not emit in this setup — the field exists in the schema but is null on every span.
- **Per-model panels on AI → Insights render "No Data"** despite `gen_ai_response_model` being populated. Use the SQL above instead.
- **Ollama timing fields are strings** — `CAST(... AS DOUBLE)` is mandatory.
- **SLOs are missing** from this OpenObserve build's UI.
- **First request after a model reload is slow** — 20–30s prefill on CPU even with the context-window fix. Subsequent requests are fast because Ollama caches the prompt prefix.
- The docs-grounded model says "I don't know" for questions the docs don't cover, by design from the system prompt.

## References

- [OpenObserve quickstart](https://openobserve.ai/docs/quickstart/)
- [OpenObserve AI Gateways — LiteLLM Proxy](https://openobserve.ai/docs/integration/ai/gateways/litellm-proxy/)
- [OpenObserve No-Code AI — Open WebUI](https://openobserve.ai/docs/integration/ai/no-code/openwebui/)
- [LiteLLM OpenTelemetry integration](https://docs.litellm.ai/docs/observability/opentelemetry_integration)
- [LiteLLM Ollama provider](https://docs.litellm.ai/docs/providers/ollama)
- [Ollama context length / `num_ctx`](https://github.com/ollama/ollama/blob/main/docs/faq.md)

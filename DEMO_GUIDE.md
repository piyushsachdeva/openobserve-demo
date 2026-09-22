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
echo -n 'piyush@techtutorialswithpiyush.com:Demo@Observe123' | base64
```

Save the output — you'll need it as `Basic <token>`.

## Step 3 — Start LiteLLM Proxy (this is what adds tracing)

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

Run it:

```bash
docker run -d --name litellm-proxy \
  -p 4000:4000 \
  -v $PWD/litellm-config.yaml:/app/config.yaml \
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
  -e OPENAI_API_KEY=sk-1234 \
  -e ENABLE_FORWARD_USER_INFO_HEADERS=True \
  -v open_webui_data:/app/backend/data \
  ghcr.io/open-webui/open-webui:main
```

`ENABLE_FORWARD_USER_INFO_HEADERS=True` is required — without it, Open WebUI never sends the user's id/email, and every trace stays anonymous.

## Step 5 — Give it ReelDrop's knowledge (the "customer support agent" part)

The goal: a chat model that only answers from ReelDrop's actual docs/pricing/features, so it can act as a support agent for the demo.

**What did NOT work:** attaching a Workspace → Knowledge collection to the model and letting Open WebUI's RAG pipeline retrieve context automatically. A model this small (0.5B–1.5B) doesn't reliably invoke Open WebUI's native-function-calling retrieval tool, so the knowledge base was silently never consulted — the model answered from its own pretraining instead (fabricated ReelDrop prices).

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
2. Open http://localhost:5080/web/, go to **Traces**, filter by `service_name = litellm-proxy`.
3. Click a span — you'll see the model name, prompt/completion token counts, latency, and cost per chat turn.
4. Open a second browser session (different Open WebUI account) and confirm its requests show up as separate traces — this is the "per-user session" observability piece.

## Monitoring customer interactions in OpenObserve

This is the actual point of the demo: not just that the bot answers correctly, but that every interaction is fully observable — tokens, cost, latency, per user.

**Per-request detail:** Traces → stream `default` → click any span with `operation_name = litellm_request` → **Attributes**. Key fields:
- `metadata_user_api_key_end_user_id` — which user/email made this request
- `gen_ai_usage_input_tokens`, `gen_ai_usage_output_tokens`, `gen_ai_usage_total_tokens` — token counts
- `gen_ai_usage_cost` — dollar cost, auto-computed, zero pricing config needed
- `gen_ai_input_messages` / `gen_ai_output_messages` — the full prompt and completion, for debugging exactly what the model saw and said

**Tokens per second (generation speed):** the child span `raw_gen_ai_request` carries Ollama's own native timing fields:
- `llm_ollama_chat_eval_count` — output tokens generated
- `llm_ollama_chat_eval_duration` — time spent generating, in **nanoseconds**
- `llm_ollama_chat_prompt_eval_count` / `llm_ollama_chat_prompt_eval_duration` — input tokens and prefill time
- Tokens/sec = `eval_count / (eval_duration / 1e9)`

SQL for a quick tokens/sec view:
```sql
SELECT llm_ollama_chat_model as model,
       llm_ollama_chat_eval_count as output_tokens,
       llm_ollama_chat_eval_duration / 1e9 as generation_seconds,
       llm_ollama_chat_eval_count / (llm_ollama_chat_eval_duration / 1e9) as tokens_per_second
FROM "default"
WHERE operation_name = 'raw_gen_ai_request'
ORDER BY _timestamp DESC
```

**Aggregated per-user table:** Dashboards → New dashboard → Add panel → chart type **Table** → Stream Type `traces`, Stream `default` → **Custom** (next to SQL/Builder) → paste:

```sql
SELECT metadata_user_api_key_end_user_id as user_email,
       gen_ai_request_model as model,
       count(*) as requests,
       sum(gen_ai_usage_input_tokens) as input_tokens,
       sum(gen_ai_usage_output_tokens) as output_tokens,
       sum(gen_ai_usage_total_tokens) as total_tokens,
       sum(gen_ai_usage_cost) as total_cost
FROM "default"
WHERE gen_ai_request_model IS NOT NULL
GROUP BY user_email, model
```

Drag the resulting fields into **First Column** / **Other Columns**, then **Apply → Save**. This is the "who is using how much, and what is it costing" view — the core payoff of the demo.

## Known gaps 
- SLOs are missing from this OpenObserve build's UI (see Step 1).
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

# Local LLM Chat + Per-User Observability — Demo Guide

Stack: **Ollama** (on the VM) → **LiteLLM Proxy** (adds tracing) → **Open WebUI** (multi-user chat), traces land in **OpenObserve**.

## Fastest way

```bash
docker compose up -d
```

That's it — skip to "Verify" below. The rest of this guide is the manual, step-by-step version in case you want to deploy each piece by hand.

## Prerequisites

- Docker installed locally. If you don't have it:
  ```bash
  brew install --cask docker
  open -a Docker
  ```
  (First launch needs your Mac password once, to finish linking the CLI.)
- Ollama already running and reachable, with a model pulled (this demo uses `qwen2.5:0.5b` on `187.77.187.85:11434`)

## Step 1 — Start OpenObserve

Docs: https://openobserve.ai/docs/quickstart/

```bash
docker run -d --name openobserve \
  -v $PWD/openobserve_data:/data \
  -p 5080:5080 \
  -e ZO_ROOT_USER_EMAIL="itzmario133@gmail.com" \
  -e ZO_ROOT_USER_PASSWORD="Demo@Observe123" \
  public.ecr.aws/zinclabs/openobserve:latest
```

Open http://localhost:5080/web/login and confirm you can log in.

**No Docker?** Run it as a standalone binary instead — download from https://openobserve.ai/downloads, then:

```bash
chmod +x openobserve
ZO_ROOT_USER_EMAIL="itzmario133@gmail.com" ZO_ROOT_USER_PASSWORD="Demo@Observe123" ./openobserve
```

Two gotchas, verified by testing this directly:
- **Password policy:** the binary rejects weak passwords (needs upper+lower+digit+special char) and panics on startup if it doesn't pass — `Demo@Observe123` above already satisfies this.
- **Changing ports:** if you need a non-default HTTP port (e.g. to run alongside the Docker version), also set `ZO_GRPC_PORT` — it defaults to `5081` independently of `ZO_HTTP_PORT`, and leaving it unset causes an internal port collision (`Address already in use`) when you move HTTP onto `5081`.

## Step 2 — Get your OpenObserve auth token

Docs: https://openobserve.ai/docs/integration/ai/gateways/litellm-proxy/

```bash
echo -n 'itzmario133@gmail.com:Demo@Observe123' | base64
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

litellm_settings:
  callbacks: ["otel"]

general_settings:
  user_header_mappings:
    - header_name: X-OpenWebUI-User-Id
      litellm_user_role: internal_user
    - header_name: X-OpenWebUI-User-Email
      litellm_user_role: customer
```

The `user_header_mappings` block is what makes traces per-user: it tells LiteLLM to read the Open WebUI user's id/email off request headers and attach them to every trace. Docs: https://docs.litellm.ai/docs/tutorials/openweb_ui

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

Docs: https://openobserve.ai/docs/integration/ai/no-code/openwebui/ and https://docs.litellm.ai/docs/tutorials/openweb_ui

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

(Port 3001 avoids clashing with anything else already using 3000 on your machine — change it if you'd rather use a different port.)

## Verify

1. Open http://localhost:3001, sign up, chat with `qwen2.5:0.5b`.
2. Open http://localhost:5080/web/, go to **Traces**, filter by `service_name = litellm-proxy`.
3. Click a span — you'll see the model name, prompt/completion token counts, latency, and cost per chat turn.
4. Open a second browser session (different Open WebUI account) and confirm its requests show up as separate traces — this is the "per-user session" observability piece.

## Viewing per-user token usage in OpenObserve

**Per-request detail:** Traces → stream `default` → click any span → **Attributes** tab. You'll see `metadata_user_api_key_end_user_id` (the user's email), plus `gen_ai_usage_input_tokens`, `gen_ai_usage_output_tokens`, `gen_ai_usage_total_tokens`, and `gen_ai_usage_cost` for that single chat turn.

**Aggregated per-user table:** Dashboards → New dashboard → Add panel → chart type **Table** → Stream Type `traces`, Stream `default` → click **Custom** (next to SQL/Builder) to unlock manual SQL → paste:

```sql
SELECT metadata_user_api_key_end_user_id as user_email,
       gen_ai_request_model as model,
       count(*) as requests,
       sum(gen_ai_usage_input_tokens) as input_tokens,
       sum(gen_ai_usage_output_tokens) as output_tokens,
       sum(gen_ai_usage_total_tokens) as total_tokens
FROM "default"
WHERE gen_ai_request_model IS NOT NULL
GROUP BY user_email, model
```

Then drag the resulting fields (`user_email`, `model`, `requests`, `input_tokens`, `output_tokens`, `total_tokens`) from the left field list into **First Column** / **Other Columns**, and click **Apply** → **Save**. This gives a live, per-user breakdown of token usage and request counts across every model — the core "who is using how much" view for the demo.

Reference docs used in this guide:
- OpenObserve quickstart: https://openobserve.ai/docs/quickstart/
- OpenObserve AI Gateways (LiteLLM Proxy): https://openobserve.ai/docs/integration/ai/gateways/litellm-proxy/
- OpenObserve No-Code AI (Open WebUI): https://openobserve.ai/docs/integration/ai/no-code/openwebui/
- LiteLLM OpenTelemetry integration: https://docs.litellm.ai/docs/observability/opentelemetry_integration
- LiteLLM Ollama provider: https://docs.litellm.ai/docs/providers/ollama
- OpenObserve dashboards & custom SQL panels: https://openobserve.ai/docs/

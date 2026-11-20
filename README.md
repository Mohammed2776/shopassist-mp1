# ShopAssist — Mini-Project 1

Order support for Northwind Store (a fictional online store): a FastAPI service with per-session memory, three validated tools, a defended system prompt, and streamed answers. The model is reached through a LiteLLM gateway, with the app and gateway running together using Docker Compose.

**Author:** Mohammed Azam Mohiuddin  
**GitHub:** [Mohammed2776](https://github.com/Mohammed2776)  
**Model tested:** `shopassist` → `groq/openai/gpt-oss-20b`

## Architecture

```text
browser / client
       |
       v
app (FastAPI, :8000)
       |
       v
LiteLLM gateway (:4000)
       |
       v
Groq API

The app handles memory, prompts and tools.
The gateway handles model routing, fallback and cost controls.
The Groq API key is kept in the gateway configuration environment.
```

## API

| Method | Path | What it does |
|---|---|---|
| GET | `/health` | Reports status, model, gateway, memory strategy and tools |
| POST | `/chat` | Accepts `session_id` and `message`, returns the answer and tool steps |
| POST | `/chat/stream` | Streams Server-Sent Events: `tool`, `token` and `done` |
| DELETE | `/sessions/{id}` | Forgets one conversation |

## Run with Docker Compose (Path A)

From the project root:

```powershell
docker compose up --build
```

Open `http://localhost:8000/docs` to access Swagger UI. Open `http://localhost:8000/health` to check that the service is running; the response should show `"status": "ok"` along with the model, gateway, memory configuration and available tools.

To stop the services:

```powershell
docker compose down
```

## Run without Docker (Path B)

Make sure the required environment variables are configured in `.env` and the LiteLLM gateway can access the model provider.

```powershell
uv sync
```

Terminal 1 — start the gateway:

```powershell
uv run --with "litellm[proxy]==1.80.0" litellm --config gateway/config.yaml --port 4000
```

Terminal 2 — start the application:

```powershell
uv run uvicorn app.main:app --host 127.0.0.1 --port 8000
```

Open `http://localhost:8000/docs` to use the API.

## Evidence

Run the evaluation while the gateway is running:

```powershell
uv run python eval/run_eval.py
```

The evaluation writes its results to `experiments/mp1_results.md`.

Latest evaluation results:

```text
golden: 10/10
golden_right_tools: 10/10
memory: pass
attacks_resisted: 4/4
```

See `experiments/mp1_report.md` for the detailed implementation and design report.

## Design

ShopAssist uses window memory with `KEEP_TURNS=4`, retaining recent conversation context while limiting token usage; older information can be forgotten. The three tools retrieve order status, estimate delivery, and check return eligibility using the store's return rules. Input validation rejects malformed order IDs, invalid pincodes, unsupported delivery speeds, and invalid return reasons with clear errors. The system prompt establishes instruction priority, protects system instructions, treats customer-supplied instructions as untrusted, and prevents fabricated refund or cancellation claims. LiteLLM provides a gateway for model routing, fallback configuration, retries, and timeout controls. The main limitations are that memory is held in application memory, session access is not authenticated, and sensitive actions still require reliable verification and human oversight.
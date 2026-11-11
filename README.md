# ShopAssist — Mini-Project 1

Order support for Northwind Store (a fictional online store): a FastAPI service with per-session memory, three validated tools, a defended system prompt and streamed answers. The model is reached only through a LiteLLM gateway, and both run together with Docker Compose.

**Author:** Sai Kumar Juttu — [GitHub: Sai-juttu-1](https://github.com/Sai-juttu-1)  
**Model tested:** `shopassist` → Groq model configured in `groq/openai/gpt-oss-20b`

## Architecture

```text
browser / client ──► app (FastAPI, :8000) ──► litellm gateway (:4000) ──► Groq
                     memory · prompt · tools    master key · fallback · cost     API_KEY lives only here
```

## API

| Method | Path | What it does |
|---|---|---|
| GET | `/health` | Liveness, plus the running model, gateway, memory strategy and tools |
| POST | `/chat` | `{"session_id", "message"}` → `{"answer", "tool_steps", "turns"}` |
| POST | `/chat/stream` | The same, streamed as Server-Sent Events: `tool`, `token`…, `done` |
| DELETE | `/sessions/{id}` | Forget one conversation |

## Run with Docker Compose (Path A)

```powershell
docker compose up --build
```

Open [http://localhost:8000/docs](http://localhost:8000/docs). To check that the application is working, open `http://localhost:8000/health`; a successful response should show `"status": "ok"` along with the running model, gateway, memory strategy and available tools.

## Run without Docker (Path B)

```powershell
uv sync

# terminal 1 - the gateway
uv run --with "litellm[proxy]==1.80.0" litellm --config gateway/config.yaml --port 4000

# terminal 2 - the app
uv run uvicorn app.main:app --port 8000
```

## Evidence

```powershell
uv run python eval/run_eval.py
```

The evaluation requires the gateway to be running and writes `experiments/mp1_results.md`.

Latest recorded results:

- Golden: **7/10**
- Golden right tools: **9/10**
- Memory: **fail**
- Attacks resisted: **3/4**

Discussion: `experiments/mp1_report.md`.

## Design

The application uses window memory with `KEEP_TURNS=4`, retaining the latest four conversation turns so that useful context is available without keeping the entire conversation indefinitely. The three tools check order status, estimate delivery time using a pincode, and check return eligibility using the order ID and return reason. Tool arguments are validated, and invalid or missing information should be reported to the customer rather than guessed. The system prompt defines ShopAssist's role, allowed facts, tool-use rules, and security protections against instruction injection, unauthorized actions, and attempts to reveal internal instructions. LiteLLM provides a central gateway between FastAPI and the Groq model, keeping provider credentials behind the gateway and supporting configuration such as model routing, fallback, and cost tracking. A major limitation is that the evaluation still shows a memory failure and one successful attack, so conversation recall and resistance to malicious customer instructions need further improvement.
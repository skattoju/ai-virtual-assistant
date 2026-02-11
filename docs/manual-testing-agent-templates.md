# Manual Testing: Agent Templates

Use this guide to bring up the app and manually test agent templates.

## Summary: Changes to get template agents working in local dev

These changes were made so agents deployed from templates work with the containerized dev stack (Ollama + LlamaStack) without editing template YAMLs.

| Area | Change |
|------|--------|
| **Backend build** | Removed `nmap-ncat` install from `deploy/local/Containerfile.backend.dev` (dnf failed on UBI9). Replaced DB wait in `deploy/local/scripts/start-backend-dev.sh` with a Python socket check so no extra packages are needed. |
| **Default model (config)** | Added `DEFAULT_INFERENCE_MODEL` in `backend/app/config.py` (from env). Used when initializing from a template or when chatting in local dev so template agents use an Ollama model instead of production model names. |
| **Template init** | In `backend/app/api/v1/agent_templates.py`: when `LOCAL_DEV_ENV_MODE=true` and `DEFAULT_INFERENCE_MODEL` is set, new agents from templates get that model instead of the template’s production model (unless the client sends an explicit `model_name`). |
| **Chat model** | In `backend/app/services/chat.py`: in local dev, use `DEFAULT_INFERENCE_MODEL` for the LlamaStack request so agents already created with the wrong model still work. |
| **Chat model fallback** | In `backend/app/services/chat.py`: in local dev, call `client.models.list()` before streaming; if the requested model isn’t in the list, use the first available model so we always send a valid LlamaStack model identifier. |
| **Model identifier** | LlamaStack expects full identifiers (e.g. `ollama/llama3.2:1b`), not bare Ollama tags (`llama3.2:1b`). Compose and docs default `DEFAULT_INFERENCE_MODEL` to `ollama/llama3.2:1b`. |
| **Ollama / LlamaStack** | `deploy/local/compose.yaml`: Ollama pulls `llama3.2:1b` (valid Ollama tag). Backend env gets `DEFAULT_INFERENCE_MODEL=ollama/llama3.2:1b`. `deploy/local/llamastack-run.yaml`: registers `model_id: llama3.2:1b` (LlamaStack exposes it as `ollama/llama3.2:1b`). |
| **Tools in dev** | Dev LlamaStack has `tool_groups: []`, so web_search isn’t registered. In `backend/app/services/chat.py`, when in local dev we omit tools in the request so the model can still respond (no web search); production keeps tools. |

No need to recreate agents after these changes: chat uses the default model in local dev, and the fallback picks an available model if the configured one isn’t found.

## 1. Bring up the app (containerized)

From the project root:

```bash
cd deploy/local
make compose-up
```

Or run the start script directly (creates `.env` from `.env.example` if missing):

```bash
cd deploy/local
./scripts/start-dev.sh
```

**Optional:** Start without MinIO (faster) if you don't need attachments:

```bash
cd deploy/local
ENABLE_ATTACHMENTS=false ./scripts/start-dev.sh
```

Wait until all services are healthy (DB, Ollama, LlamaStack, backend, frontend). The backend loads agent templates from `backend/agent_templates/*.yaml` into the database on startup.

**Template model wiring:** Templates use production model names (e.g. `meta-llama/Llama-3.1-8B-Instruct`). In local dev, the compose stack sets `DEFAULT_INFERENCE_MODEL=ollama/llama3.2:1b` so template-initialized agents use the Ollama model. Chat also uses this default when in local dev mode, so template agents work without changing YAML.

**Why tools are off in dev:** The dev LlamaStack config (`deploy/local/llamastack-run.yaml`) has `tool_groups: []` — no tools (e.g. web_search) are registered with LlamaStack. Production install configures tool groups (including Tavily web search) during `make install`. So in local dev the backend omits tools when calling LlamaStack so the model can still respond; agents answer without web search. To use tools in dev you would need to register a web search tool group in `llamastack-run.yaml` and pass a Tavily API key to the LlamaStack container (see LlamaStack docs for the exact config).

## 2. Service URLs

| Service      | URL                      |
|-------------|--------------------------|
| Frontend    | http://localhost:5173    |
| Backend API | http://localhost:8000    |
| API Docs    | http://localhost:8000/docs |
| ReDoc       | http://localhost:8000/redoc |

## 3. Test agent templates via API

With the backend running, use the interactive docs or `curl`:

- **List available templates**
  `GET /api/v1/agent_templates/`
  Example: `curl -s http://localhost:8000/api/v1/agent_templates/`

- **List suites**
  `GET /api/v1/agent_templates/suites`
  Example: `curl -s http://localhost:8000/api/v1/agent_templates/suites`

- **Suites by category**
  `GET /api/v1/agent_templates/suites/categories`
  Example: `curl -s http://localhost:8000/api/v1/agent_templates/suites/categories`

- **Template details** (by template id/name)
  `GET /api/v1/agent_templates/{template_id}`
  Example: `curl -s http://localhost:8000/api/v1/agent_templates/core_banking`

- **Initialize one agent from a template**
  `POST /api/v1/agent_templates/initialize`
  Body: `{"template_name": "core_banking", "include_knowledge_base": false}`
  Use Swagger at http://localhost:8000/docs to call this with a JSON body.

- **List virtual agents** (after initializing)
  `GET /api/v1/virtual_agents/`
  Example: `curl -s http://localhost:8000/api/v1/virtual_agents/`

## 4. Test via frontend

1. Open http://localhost:5173
2. In local dev mode (`LOCAL_DEV_ENV_MODE=true`), auth is bypassed and a dev user is used.
3. Use the UI to browse agent templates, deploy an agent from a template, and open a chat with that agent.

## 5. Stop services

```bash
cd deploy/local
./scripts/stop-dev.sh
```

Or:

```bash
cd deploy/local
make compose-down
```

## Troubleshooting

- **Backend won’t start:** Ensure PostgreSQL, Ollama, and LlamaStack are up. Check `podman compose -f deploy/local/compose.yaml ps` and backend logs.
- **Templates empty:** Backend loads from `backend/agent_templates/`. Ensure YAML files exist and that the backend started without errors (see logs for “Auto-populating templates” or “Templates already populated”).
- **Reset DB and re-load templates:** Remove the Postgres volume and bring compose up again (see CONTRIBUTING.md “Reset the database”).

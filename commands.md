# commands.md

The pipeline is split across four nested repos. Each has its own venv +
`requirements.txt`; run its commands from inside that repo. Each repo loads its own
`.env` first, falling back to an umbrella `.env` at this project root (one shared
`.env` with `OPENAI_API_KEY` + `DATABASE_URL` is the simplest setup).

```bash
# per-repo environment (run inside each sub-repo)
python3 -m venv .venv
source .venv/bin/activate
deactivate
pip install -r requirements.txt
```

All Docker services also attach to a shared external bridge network, `webnet`, so
they share one host-wide network with the project's other containers (LLM
frontend, LiteLLM, Ollama). Create it once before any `docker compose` run:

```bash
docker network create webnet        # one-time; no-op if it already exists
```

```bash
# 1. vector-db-rag-context-pipeline — Postgres + pgvector (Docker)
cd vector-db-rag-context-pipeline
docker compose up -d
docker compose down        # stop (data persists in the pgdata volume)
docker compose down -v     # stop and wipe the database

# 2. indexing-rag-context-pipeline — build the index (once, or after the PDF changes)
cd indexing-rag-context-pipeline
python build_index.py

# 3. engine-rag-context-pipeline — REPL + retrieval eval
cd engine-rag-context-pipeline
python ask.py                              # interactive REPL
python eval/run_eval.py                    # hit-rate@k / MRR (no API key needed)
python eval/run_eval.py --k 10 --show-misses
docker compose run --rm engine                            # or run the REPL in Docker (run-on-demand, no `up`)
docker compose run --rm engine python eval/run_eval.py    # eval in Docker (no key needed)

# 4. backend-rag-context-pipeline — HTTP API (docs at http://localhost:8000/docs)
cd backend-rag-context-pipeline
uvicorn api.main:app --reload
uvicorn api.main:app --host 0.0.0.0 --port 8000     # bind explicitly
docker compose up --build                            # or run it in Docker (joins the vector-db + shared webnet networks; reaches LiteLLM over webnet)
curl localhost:8000/health
# more curl examples: backend-rag-context-pipeline/curl-commands.md
```

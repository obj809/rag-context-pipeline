# RAG Context Pipeline

A minimal Retrieval-Augmented Generation (RAG) pipeline over a single PDF. Loads a
PDF, splits it into chunks, embeds each chunk locally, stores the vectors in
Postgres + pgvector, and answers questions by retrieving the most relevant chunks
and sending them to an OpenAI model — with answers that cite the page each fact
came from.

This umbrella project has been split into **four independent repos**, one per
concern, each nested here as its own git repo (with its own remote). The umbrella
itself now holds only this documentation; all code lives in the sub-repos.

## Repositories

| Repo | Owns | Entry point |
|---|---|---|
| [`vector-db-rag-context-pipeline/`](https://github.com/obj809/vector-db-rag-context-pipeline) | Postgres + pgvector (Docker) | `docker compose up -d` |
| [`indexing-rag-context-pipeline/`](https://github.com/obj809/indexing-rag-context-pipeline) | PDF → chunks → embeddings → `chunks` table; the source PDF | `python build_index.py` |
| [`engine-rag-context-pipeline/`](https://github.com/obj809/engine-rag-context-pipeline) | query engine (retriever + LCEL chain), the REPL, and the retrieval eval | `python ask.py` / `python eval/run_eval.py` |
| [`backend-rag-context-pipeline/`](https://github.com/obj809/backend-rag-context-pipeline) | HTTP API (FastAPI) over the engine | `uvicorn api.main:app` |

Each sub-repo has its own README, `requirements.txt`, and `.env.example`.

## How it works

```mermaid
%%{init: {"flowchart": {"nodeSpacing": 110}}}%%
flowchart LR
    subgraph indexing["indexing-rag-context-pipeline<br/>(run once per document)"]
        pdf[/"data/net-zero-report.pdf"/] --> build["build_index.py<br/>load → chunk → embed<br/>(BGE, local)"]
    end

    subgraph vectordb["vector-db-rag-context-pipeline<br/>(Docker)"]
        chunks[("Postgres + pgvector<br/>chunks table<br/>text · embedding · page · embedding_model")]
    end

    subgraph engine["engine-rag-context-pipeline"]
        retriever["retriever.py<br/>SQL cosine top-k"] --> chain["chain.py<br/>LCEL answer chain"]
        repl["ask.py<br/>(REPL)"] --> chain
        eval["eval/run_eval.py<br/>hit-rate@k · MRR"] --> retriever
    end

    subgraph backend["backend-rag-context-pipeline<br/>(FastAPI, Docker)"]
        api["api/main.py<br/>POST /ask · GET /health"]
    end

    user([user / client])
    openai["OpenAI<br/>gpt-5.4-nano"]

    build -- "writes rows<br/>(drop + recreate)" --> chunks
    chunks -- "top-k chunks" --> retriever
    chain -- "question + chunks" --> openai
    openai -- "page-cited answer" --> chain
    api -. "imports retriever + chain<br/>(sys.path bridge)" .-> chain
    user -- "HTTP" --> api
    user -- "terminal" --> repl
```

The dashed edge is a **build/import-time** relationship, not a network call: the
backend imports the engine's leaf modules (and its Docker image copies them in),
so at runtime only the database and OpenAI are remote.

The repos communicate **only** through the Postgres `chunks` table. Indexing writes
the chunks, their embeddings, the source `page`, and the `embedding_model` name once
per document; the engine retrieves the top-k via a SQL cosine-distance search
(`ORDER BY embedding <=> query LIMIT k`) and composes the answer as a LangChain LCEL
chain (`retriever | prompt | llm | parser`). The backend imports the engine's leaf
modules to expose the same chain over HTTP. The `embedding_model` column is the glue
that stops the query side embedding with a different model than the index was built
with.

The engine wraps raw pgvector SQL in a LangChain `BaseRetriever` rather than using
LangChain's `PGVector` vectorstore, so the `chunks` schema stays under the project's
control. Embeddings are local `sentence-transformers` (`BAAI/bge-small-en-v1.5`, no
API key); only `gpt-5.4-nano` is called through LangChain (`ChatOpenAI`).

**Known limitation — charts and figures.** PyMuPDF4LLM extracts text, tables, and
headings well, but discards data labels embedded in **charts/figures** as picture
content. This report is chart-heavy, so some figures' numbers (e.g. the
methane-by-sector pie chart) are not in the index; recovering them would require
vision-based extraction. Affected eval questions are flagged with
`"chart_dependent": true` in `engine-rag-context-pipeline/eval/dataset.jsonl`.

## Full local setup

Prerequisites: Python 3.9+, Docker, and an OpenAI API key.

Each sub-repo loads its own `.env` first, falling back to an umbrella `.env` at this
project root — so the quickest path is to create one shared `.env` here and let all
repos read it:

```bash
echo "OPENAI_API_KEY=sk-..." > .env
echo "DATABASE_URL=postgresql://rag:rag@localhost:5432/rag" >> .env
```

Then, from each repo (each with its own venv + `pip install -r requirements.txt`):

```bash
# 1. Database
cd vector-db-rag-context-pipeline && docker compose up -d && cd -

# 2. Build the index (first run downloads the ~130MB embedding model)
cd indexing-rag-context-pipeline && python build_index.py && cd -

# 3. Ask questions — REPL, or the retrieval eval
cd engine-rag-context-pipeline && python ask.py            # interactive REPL
#                                  python eval/run_eval.py  # hit-rate@k / MRR (no API key)

# 4. Or serve the HTTP API (interactive docs at http://localhost:8000/docs)
cd backend-rag-context-pipeline && uvicorn api.main:app --reload
```

To index a different PDF, drop it in `indexing-rag-context-pipeline/data/` and update
`PDF_PATH` in that repo's `build_index.py`.

## Stack

| Layer | Choice | Repo |
|---|---|---|
| PDF extraction | `pymupdf4llm` (per-page Markdown) | indexing |
| Chunking | `RecursiveCharacterTextSplitter` (`langchain-text-splitters`) | indexing |
| Embeddings | `sentence-transformers` + `BAAI/bge-small-en-v1.5` (local) | indexing + engine |
| Vector store | Postgres + `pgvector` (Dockerized), via `psycopg` | vector-db |
| Retrieval + orchestration | LangChain LCEL — `langchain-core` | engine |
| Answer generation | OpenAI (`gpt-5.4-nano`) via `langchain-openai` `ChatOpenAI` | engine |
| HTTP API | `fastapi` + `uvicorn`, pooled via `psycopg-pool` | backend |
| Env loading | `python-dotenv` | all |

## Tuning

The knobs worth experimenting with live in each repo's README:

| Constant | Repo / file | Default |
|---|---|---|
| `CHUNK_SIZE`, `CHUNK_OVERLAP` | `indexing-rag-context-pipeline/build_index.py` | 1200 / 150 chars |
| `EMBEDDING_MODEL` | `indexing-rag-context-pipeline/build_index.py` | `BAAI/bge-small-en-v1.5` |
| `TOP_K`, `OPENAI_MODEL` | `engine-rag-context-pipeline/ask.py` | 6 / `gpt-5.4-nano` |
| `QUERY_PREFIX` | `engine-rag-context-pipeline/retriever.py` | BGE instruction prefix |

Changing `EMBEDDING_MODEL` requires re-running `build_index.py` — the rebuild drops
and recreates the `chunks` table with the new `VECTOR(dim)` and records the new model
name, so the engine picks it up automatically.

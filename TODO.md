# To-do

- [x] **Streaming — done via `POST /chat`.** The backend now streams answers as raw `text/plain` fragments (`StreamingResponse` + `chain.stream` in `backend-rag-context-pipeline/api/main.py`), built to the contract in `integration-plan.md`. `/ask` deliberately stays blocking JSON for its existing consumers.

- [ ] **Answer-quality scoring is a deliberate next increment.** The eval (`engine-rag-context-pipeline/eval/run_eval.py`) currently scores retrieval only (hit-rate@k / MRR). Build on the ground-truth `answer` field already in `eval/dataset.jsonl` to score generated answers against the reference.

- [ ] **Make container startup fully offline (`HF_HUB_OFFLINE=1`).** The eval run inside the engine container warned `You are sending unauthenticated requests to the HF Hub` — sentence-transformers still pings HuggingFace at startup to check for model updates, even though the BGE weights are baked into the image. That adds a network round-trip and, worse, a network dependency: on a flaky connection or if HF is down, startup stalls or fails. Fix: set `HF_HUB_OFFLINE=1` in the compose `environment:` block of `engine-rag-context-pipeline/docker-compose.yml` (and `backend-rag-context-pipeline/docker-compose.yml`, which has the same baked-model setup).

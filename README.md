# Hardware-Aware Multi-LLM Orchestration

Hardware-Aware Multi-LLM Orchestration is a local-first assistant that routes
each request to a specialist language model selected for the task and the
available hardware. It combines deterministic routing, embedding-based
classification, optional dataset retrieval, short-term conversation context,
persistent user preferences, response validation, and a lightweight browser
interface.

The project is designed for privacy-conscious development: requests are sent
to a local Ollama server rather than a hosted inference API.

## What it provides

- **Task routing:** coding, mathematics, writing, general questions, and image
  analysis.
- **Hardware profiles:** low, mid, and high profiles control model selection,
  validation depth, and model keep-alive settings.
- **Difficulty-aware agents:** easy, medium, and hard requests select different
  model tiers.
- **Context continuity:** the current process keeps the most recent
  conversation turns available to specialists.
- **Preference memory:** explicit user preferences can be stored in ChromaDB
  and applied across relevant future requests.
- **Response validation:** Python code can be executed in an isolated
  subprocess with a timeout, while non-code responses can be reviewed by a
  validator model.
- **Image support:** image paths can be detected, resized, and passed to a
  local vision model for extraction before normal routing.
- **Browser UI:** a small dependency-free frontend exposes the assistant through
  a local HTTP server.

## Architecture

```text
User request
    |
    +--> image detection and optional vision extraction
    |
    +--> embedding classifier and dataset k-nearest-neighbour router
    |
    +--> difficulty estimation
    |
    +--> tiered specialist agent through Ollama
    |
    +--> code execution or response validation
    |
    +--> temporary chat history and optional preference memory
    |
    +--> final response
```

### Main components

- [`config.py`](./config.py) — hardware profiles, model assignments, behavior
  flags, and optional dataset locations.
- [`core/agents.py`](./core/agents.py) — Ollama client, specialist prompts,
  vision support, and model registry.
- [`core/orchestrator.py`](./core/orchestrator.py) — end-to-end routing,
  context assembly, preference handling, validation, and escalation.
- [`core/memory.py`](./core/memory.py) — ChromaDB-backed preference memory.
- [`core/datasets.py`](./core/datasets.py) — optional local and Hugging Face
  dataset loading and semantic routing.
- [`core/code_validator.py`](./core/code_validator.py) — fenced Python
  extraction and time-limited subprocess validation.
- [`frontend_server.py`](./frontend_server.py) — local HTTP API and static
  frontend server.
- [`frontend/`](./frontend/) — browser chat interface.

## Requirements

- Python 3.10 or newer
- Ollama running locally at `http://localhost:11434`
- Sufficient memory for the selected hardware profile and local models

The default model assignments are:

| Purpose | Default model family |
| --- | --- |
| Coordinator | `qwen3:4b` |
| Coding | `qwen2.5-coder` |
| Mathematics | `deepseek-r1` |
| Writing | `gemma3` |
| General questions | `llama3.2` / `qwen3` |
| Vision | `qwen2.5vl:3b` |
| Validation | `phi4-mini` and `deepseek-r1` |

Model tags can be changed in [`config.py`](./config.py) to match the models
installed on the local Ollama instance.

## Installation

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

Pull the models required by the selected profile, for example:

```bash
ollama pull qwen3:4b
ollama pull qwen2.5-coder:7b
ollama pull deepseek-r1:7b
ollama pull gemma3:4b
ollama pull llama3.2:latest
ollama pull phi4-mini
ollama pull qwen2.5vl:3b
```

The first run also downloads the `all-MiniLM-L6-v2` embedding model through
Sentence Transformers.

## Run the terminal assistant

From the project root:

```bash
python -m core.orchestrator
```

The terminal interface accepts ordinary questions and image paths in the form
`image /path/to/file.png`. Use `diff` to inspect the last routing decision and
`exit` or `quit` to stop.

## Run the browser interface

```bash
python frontend_server.py
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000) in a browser. The server
also exposes `GET /api/health` and `POST /api/chat`.

Example request:

```bash
curl -X POST http://127.0.0.1:8000/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"message":"Write a Python function that reverses a list."}'
```

## Configuration

Set `HARDWARE_PROFILE` in [`config.py`](./config.py):

- `low` — embedding-first routing and lighter validation.
- `mid` — balanced routing and tiered specialists; this is the default.
- `high` — deeper validation and longer model keep-alive periods.

Optional local routing datasets can be placed at:

```text
data/Math Questions.json
data/Leetcode Questions.csv
```

They are not required. If they are absent, the application continues with
embedding and rule-based routing. The optional Hugging Face retrieval path is
loaded lazily and is also skipped when unavailable.

ChromaDB data is created at runtime and is intentionally excluded from source
control. Ordinary questions remain in temporary process memory; only explicit
preference updates are persisted.

## Validation

The codebase includes an offline routing check in
[`_test_routing3.py`](_test_routing3.py). It requires the embedding dependency
and the optional datasets for a complete routing run. Python syntax can be
checked without starting Ollama:

```bash
python -m compileall config.py core frontend_server.py
```

## Security and privacy

- Inference is local by default.
- The browser server binds to `127.0.0.1`.
- Generated ChromaDB stores and local datasets should not be committed.
- Generated Python responses are executed only when they contain Python fenced
  blocks, with a ten-second subprocess timeout.
- Do not place credentials or private datasets in this repository.

## Ownership

**Owner:** Rashwin Krishna


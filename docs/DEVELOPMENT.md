# Code, pipelines and tests

For deployment commands use [the deployment guide](DEPLOYMENT.md); for component
and storage roles use [the architecture](../ARCHITECTURE.md).

## Code map

| Entry point | Responsibility |
|---|---|
| [frontend/src/main.jsx](../frontend/src/main.jsx) | React selectors, question form, chat history, citations, tokens and scrollable SLM exchange panes |
| [frontend/src/style.css](../frontend/src/style.css) | Responsive visual theme |
| [assistant/app.py](../assistant/app.py) | FastAPI API, session cookie, CSRF/same-origin checks, security headers and static UI serving |
| [assistant/service.py](../assistant/service.py) | Scenario/model resolution, bounded history, retrieval and grounded conversation |
| [assistant/clients.py](../assistant/clients.py) | MiniLM/Qdrant/model clients, TLS, response validation and credential-free model exchanges |
| [assistant/embeddings.py](../assistant/embeddings.py) | Offline MiniLM/FastEmbed service |
| [assistant/indexer.py](../assistant/indexer.py) | PDF extraction/chunking, corpus fingerprint and vector snapshots |
| [assistant/scenarios.py](../assistant/scenarios.py) | Definition/ZIP validation, scenario catalog and atomic publication |
| [assistant/config.py](../assistant/config.py) | Pinned embedding identity, dimensions/chunk settings and backend model configuration |
| [tools/scenario.py](../tools/scenario.py) | Create/package scenarios, upload through kubectl and run/wait for the indexing Job |
| [tools/deploy_assistant.py](../tools/deploy_assistant.py) | Read-only preflight and explicitly enabled, namespaced deployment |
| [tools/build_acr.ps1](../tools/build_acr.ps1) | Remote app/embedding builds and Qdrant mirroring |
| [chat.py](../chat.py) | Separate standard-library model chat; no PDF retrieval |

## Pipeline 1: ingest and index a content release

This runs **when publishing documents**, not on every question:

1. `tools.scenario package` bundles the definition, prompt and full PDF set.
2. `tools.scenario deploy` validates it, checks ownership/capacity, takes a
   publication lock, uploads to the document PVC and creates an indexing Job.
3. On Azure Local, `pypdf` extracts text page by page. Python normalizes whitespace
   and makes 900-character windows with 120-character overlap.
4. The separate MiniLM service creates one 384-dimensional vector per chunk.
5. The indexer stores vectors/text/source metadata in a new Qdrant snapshot.
6. Publication makes the scenario's prompt, revision and immutable collection
   visible together through one catalog record.

Unchanged PDF fingerprints reuse the existing snapshot; changed/removed PDFs
trigger a complete new snapshot for that scenario. Prompt/definition-only changes
do not need re-embedding. Failure keeps the previously published scenario usable.
Old snapshots remain available for recovery; this is not incremental per-chunk
indexing or automatic retention.

See [the chunking example](../ARCHITECTURE.md#how-pdf-chunking-works-in-this-application)
and [publication/recovery details](SCENARIOS.md).

## Pipeline 2: answer a question

1. React sends a same-origin `POST /api/chat` with question, scenario, selected
   model and CSRF token. FastAPI validates the request and browser session.
2. The service resolves the scenario's published snapshot and takes a
   non-queuing lock: one generation request at a time.
3. It uses bounded history to resolve follow-ups, embeds the retrieval question
   and searches Qdrant for up to four passages above the configured score threshold.
4. With no evidence, it returns an explicit unsupported answer **without calling
   Qwen**. Otherwise it constructs instructions + history + PDF excerpts + question.
5. The selected model client sends an authenticated, TLS-verified, non-streaming
   completion request to its local Foundry Service.
6. React displays the answer, document/page sources, model-reported token usage
   and actual request body / untrimmed response content and finish reason.

Successful turns enter bounded in-memory history keyed by browser session,
scenario, revision and model. Failed turns are not saved. Sessions expire after
one hour; switching scenario/model resets displayed history. Exchanges are held
in browser memory, not persisted in Qdrant or server history. Headers/keys/endpoint
URLs are excluded, but prompt/history/PDF text is visible to that browser.

The app retrieves evidence, not external systems: Qwen cannot browse referenced
documents, run tools, approve claims or update records.
See [the API contract](SCENARIOS.md#api-and-verification).

## Dependencies and builds

- [requirements.txt](../requirements.txt): direct runtime dependencies.
- [requirements.lock.txt](../requirements.lock.txt): resolved container/runtime dependencies.
- [requirements-dev.txt](../requirements-dev.txt): local test/fixture dependencies.
- [frontend/package-lock.json](../frontend/package-lock.json): pinned frontend dependency graph.

Use the isolated Python 3.12 environment from [deployment step 1](DEPLOYMENT.md#1-prepare-isolated-application-dependencies).
Do not use Azure CLI's Python. The [Dockerfile](../Dockerfile) builds React into
the FastAPI image; no separate UI server or CDN is needed in production.

## Testing

Tests are part of the repository, not dependent on a running cluster:

| Suite | Checks |
|---|---|
| [tests/test_assistant.py](../tests/test_assistant.py) | Generated PDF extraction, in-memory Qdrant indexing/persistence, failed-index recovery, chat/follow-ups, session isolation, clients, usage and exchanges |
| [tests/test_scenarios.py](../tests/test_scenarios.py) | Package validation, per-scenario isolation, publication and failed-update behavior |
| [tests/test_deployment.py](../tests/test_deployment.py) | Ownership, capacity and preserved placement guards |
| [tests/test_manifests.py](../tests/test_manifests.py) | Requests match guards, namespaced ownership, restricted pods, read-only indexer documents and credential scope |
| [test_chat.py](../test_chat.py) | Original standard-library client's conversation and request/error handling |

Run from the repository root after installing development dependencies:

```powershell
.\.venv\Scripts\python.exe -B -m unittest -q `
    tests.test_assistant tests.test_scenarios tests.test_deployment `
    tests.test_manifests test_chat
if ($LASTEXITCODE -ne 0) { throw "Tests failed." }
```

For a frontend change, with supported Node 22 on PATH:

```powershell
Push-Location frontend
try {
    npm.cmd ci
    if ($LASTEXITCODE -ne 0) { throw "Frontend dependency installation failed." }
    npm.cmd run build
    if ($LASTEXITCODE -ne 0) { throw "Frontend build failed." }
} finally {
    Pop-Location
}
```

Local tests use fixtures/mocks and in-memory or temporary vector storage; they
do not prove deployment health. Validate the ACR-built image in-cluster through
the browser: a known answer, source citations, follow-up, reset, unsupported
question, model exchange panes and persistent indexes after an approved restart.
Do not equate a Ready pod or token count with answer correctness.

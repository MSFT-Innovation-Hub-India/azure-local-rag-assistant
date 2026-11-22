# Architecture: PDF assistant and Qwen on Azure Local

## Deployment overview

[Product-icon platform view](./docs/images/platform-architecture.svg) separates
the client, Azure cloud and on-premises installation. Microsoft product icons
come from the [Azure architecture icon set](https://learn.microsoft.com/azure/architecture/icons/)
under its diagram/documentation terms; Kubernetes and Istio icons come from
[CNCF artwork](https://github.com/cncf/artwork). Generic client, datacenter and
certificate illustrations are not product logos.

[Connected platform and application overview](./docs/images/connected-architecture.svg)
shows the workstation, Azure services and Azure Local multi-rack boundary
together. It separates the outbound-established Arc session from local AI
processing and provisioning; the component diagram below expands the workloads.

![PDF assistant and Qwen deployment overview](./architecture.svg)

[Scalable SVG](./architecture.svg) / [PNG](./architecture.png).
This is a logical component view, not a physical host or node-placement map.

```text
Browser -> local Arc proxy / Cluster Connect / kubectl port-forward
        -> FastAPI + React on Azure Local
        -> MiniLM question embedding -> Qdrant passages -> selected Qwen answer

Content release -> document PVC -> Python indexing Job
                -> MiniLM chunk embeddings -> new Qdrant snapshot -> publication
```

FastAPI serves the built UI and same-origin API. No browser model/database keys,
separate production UI server or public database/model endpoint is required.
The backend calls the selected model's internal HTTPS Service, not the developer
port-forward or an Azure-hosted inference API.

**Connected sovereignty boundary:** parsing, embeddings, retrieval, persistent
documents/vectors and Qwen inference run on Azure Local. Entra/Arc provide Azure
identity, management and cloud-mediated browser/upload access; ACR supplies
builds/images. Provisioning downloads and platform monitoring are separate
outbound flows. This demo is not air-gapped or proof that transport never leaves
the site. See [prerequisites](./docs/PREREQUISITES.md) and
[detailed Python model transport](./docs/CHAT.md#detailed-model-transport).

## Connected access architecture

### Cluster administration through Azure Arc

The on-premises cluster establishes an outbound session to Azure Arc; the user's
`az connectedk8s proxy` connects to Azure and directs local kubectl requests onto
that session. Entra authentication and Kubernetes authorization guard access;
no inbound Internet firewall port on the appliance is required for this path.
See [the connection flow, agents and identity controls](./docs/CONNECTIVITY.md#1-cluster-access-an-outbound-established-two-way-channel).

### End-user browser access through port-forwarding

`kubectl port-forward` carries browser HTTP from localhost:8080 through the
existing Arc connection to the app pod's port 8000, and returns the response.
This is connected demo access, not a private production application endpoint.
See [the separate application-access flow and Mermaid diagram](./docs/CONNECTIVITY.md#2-application-access-browser-http-over-kubectl-port-forward)
and [the private production boundary](./docs/CONNECTIVITY.md#3-demo-access-versus-a-private-production-design).

## Existing platform versus added application

| Layer | Components | Responsibility |
|---|---|---|
| Existing Azure Local / Kubernetes | Control plane, worker pools, DNS, CNI and Azure Local CSI | Compute, networking, scheduling and persistent volumes |
| Existing Arc platform | Cluster-connect agents, authentication and extension management | Connected management and kubectl access |
| Existing Foundry platform | Operator, local model store/cache, certificate/trust management, Gateway API/Istio | Provision and expose local model services |
| Existing model workloads | Qwen2.5 0.5B and later 14B CPU serving pods | Generate model completions |
| Added `assistant-demo` application | React/FastAPI, MiniLM embedding service, Qdrant, document helper, indexing Jobs, two PVCs | PDF ingestion, retrieval and conversational answers |

The original `systempool` worker hosted the initial 0.5B model. A second worker
was later approved for the assistant; a higher-configured `slmnodepool` was added
for 14B. These are recorded lab decisions, not changes the application performs.
Existing platform/models must be reused, not reinstalled or resized by a demo.

### Foundry Local and CPU inference

Foundry Local handles the model-serving lifecycle separately from the assistant.
A catalog asset/version and `ModelDeployment` definition specify the model,
runtime, CPU resources and placement. Its operator reconciles these into serving
pods, Services, keys and certificate/routing resources; the local model store
supplies the artifacts at startup. The inference container then executes the
model on the appliance. The app image contains client code, not Qwen weights.

Only CPU resources were available to this walkthrough in the lab cluster.
The deployed Qwen2.5 0.5B and 14B use the CPU ONNX GenAI runtime; the examples
require no GPU. The same model services support the different document scenarios.
The higher-configured worker hosts 14B, but larger model size does not guarantee
better answers and CPU latency is workload-dependent.

## Pods and containers

Application names/containers come from [stack.yaml](./deploy/stack.yaml) and
[index-job.yaml](./deploy/index-job.yaml). Platform names describe the recorded
lab inventory; exact versions/readiness can change.

### Added application: `assistant-demo`

| Pod pattern | Container | Role |
|---|---|---|
| `assistant-web-*` | `app` | FastAPI serves React, handles sessions/scenarios, retrieves evidence and calls Qwen |
| `embeddings-*` | `embeddings` | MiniLM/FastEmbed text-to-vector API; model files baked into the image |
| `qdrant-0` | `qdrant` | Stores/searches vectors, passage text, metadata and scenario catalog |
| `documents-*` | `documents` | Long-running helper mounts the PDF volume for content transfer; not an inference service |
| `pdf-index-*` | `indexer` | One-off PDF parsing/chunking/embedding/publication; exits on completion |

Each application pod has one container and no added sidecars. A successful
indexing Job can show `0/1 Completed`; that does not mean its indexing failed.

### Existing model and platform components

| Pod / component | Containers or process | Role |
|---|---|---|
| `qwen2-5-0-5b-cpu-*`, `qwen2-5-14b-cpu-*` | `inference`, `nginx-sidecar`; native `otel-sidecar`; startup `model-store-pull` | CPU serving, TLS front end, platform telemetry and model-file retrieval |
| `inference-operator-*` | `operator`, `nginx-sidecar`, native `otel-sidecar` | Reconciles model deployments; does not generate tokens |
| `inference-operator-model-store-*` | `registry`, `model-store-auth`, `nginx-sidecar` | Local model artifacts and authenticated store access |
| `inference-internal-gateway-istio-*` | `istio-proxy` | Installed Envoy model gateway; separate from the Arc access proxy |
| `istiod` in `istio-system` | Istio control plane | Configures the model gateway |
| `telemetry-collector-*` | `otel-collector`, native `msi-adapter`, `mdsd`, `mdm` | Platform monitoring, not the PDF/answer data pipeline |
| Certificate/trust controllers | cert-manager / trust-manager processes | Model TLS certificates and CA trust |
| Arc agents in `azure-arc` | Cluster-connect/auth/configuration agents | Outbound-established Arc connectivity and management |
| `kube-system` components | API server, kubelet/runtime, DNS, Cilium and CSI components | Kubernetes operations, service networking and storage |

Native sidecars are declared under `initContainers` with `restartPolicy: Always`
and count toward READY. A one-off startup init container does not. Thus a model
pod can show `3/3` even though it has only two regular `spec.containers`;
READY counts containers, not model replicas.

To inspect the current inventory without changing it:

```powershell
kubectl --context zines-22325-aks get pods -A
```

For one pod, use `kubectl --context zines-22325-aks get pod <pod-name> -n <namespace> -o yaml`,
replacing both placeholders. Read container, init-container and volume names;
do not export Secrets or commit kubeconfigs.

## Volumes and configuration

| Volume / mount | Used by | Purpose |
|---|---|---|
| `pdf-documents` PVC, 2 GiB -> `/documents` | Document helper (read/write), indexer (read-only) | Persistent PDFs and uploaded scenario ZIPs |
| `qdrant-data` PVC, 5 GiB -> `/qdrant/storage` | Qdrant | Persistent vectors/text/catalog; not the source PDF filesystem |
| Secret -> `/run/credentials` | App; indexer gets only the Qdrant key | Backend model/database credentials |
| ConfigMaps -> `/run/trust`, `/run/models` | App | Public model CA and model catalog |
| Bounded `emptyDir` -> `/tmp` | App, embeddings, Qdrant and indexer | Temporary working space, not persistent document storage |
| `/models` in the embedding image | Embeddings | Provisioned MiniLM artifacts; not a separate PVC |

The web pod does **not** mount the source PDF PVC: it retrieves text/metadata
from Qdrant. Azure Local CSI provisions the two filesystem PVCs. The indexer is
colocated with the document helper for the ReadWriteOnce document volume.
Do not delete PVCs for recovery.

## Chunking, embeddings and vector storage

MiniLM is not part of the Qdrant database and does not perform this app's chunking.

| Stage | Component | What it does |
|---|---|---|
| Extract | `pypdf` in [indexer.py](./assistant/indexer.py) | Reads PDF page text; no OCR |
| Chunk | Python `extract_chunks` | Splits normalized page text with overlapping character windows |
| Vectorize | MiniLM through [FastEmbed](./assistant/embeddings.py) | Converts chunks/questions into 384-dimensional vectors |
| Store/search | Qdrant via [backend clients](./assistant/clients.py) | Persists vectors/text/metadata and finds similar passages |
| Answer | Qwen via [the assistant service](./assistant/service.py) | Generates a response from the question, history and retrieved evidence |

### How PDF chunking works in this application

Before MiniLM receives any text, Python:

1. Reads each PDF page with `pypdf`; rejects encrypted, corrupt or entirely
   textless PDFs. Blank pages contribute no chunks.
2. Normalizes whitespace with `" ".join(text.split())`.
3. Takes a window of **up to 900 characters**.
4. Advances **780 characters** (`900 - 120`), retaining **120-character overlap**.
5. Stops when the page is covered; the last chunk can be shorter. Chunks never
   cross pages.
6. Retains relative filename, 1-based page, zero-based chunk number within the
   document, text and document content hash for storage and source citations.

Example: a normalized 2,000-character page:

| Chunk | Character positions (1-based, inclusive) | Length |
|---|---|---|
| First | 1-900 | 900 |
| Second | 781-1680 | 900 |
| Third | 1561-2000 | 440 |

The overlaps are 781-900 and 1561-1680. This is **fixed-size character chunking**,
not semantic/sentence splitting; boundaries can cut a word or sentence.
These are not token counts, and MiniLM may truncate at its own token limit.
Settings are defined in [config.py](./assistant/config.py).

Chunking/embedding run at content publication, not per question. Unchanged
documents reuse their snapshot; updates publish a new complete scenario snapshot
and preserve the previous searchable index on failure.

### Open-source components and project choices

- [all-MiniLM-L6-v2](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2):
  Apache-2.0 English sentence/paragraph embedding model, not a parser/database.
- [FastEmbed](https://github.com/qdrant/fastembed): Apache-2.0 library maintained
  by Qdrant, running the model with ONNX Runtime. The pinned
  [ONNX snapshot](https://huggingface.co/Qdrant/all-MiniLM-L6-v2-onnx) is baked
  into the embedding image and loaded offline.
- [Qdrant](https://github.com/qdrant/qdrant): Apache-2.0 vector database. This
  deployment is local, not Qdrant Cloud.

MiniLM/FastEmbed and Qdrant are this project's lightweight CPU/persistent-search
choices, not mandatory Azure Local or universal RAG defaults. FastEmbed's Qdrant
ownership does not put MiniLM inside the database. The UI model selector chooses
generative Qwen, not the embedding model.

## Shared engine and scenario isolation

All PDF scenarios share the app, embedding API, Qdrant server and Qwen services.
Each scenario supplies its own definition, prompt and document set, with its own
published revision/vector snapshot and separate chat history. No new model or
application Deployment is needed for content-only scenarios.

The separate [telemetry assistant](https://github.com/MSFT-Innovation-Hub-India/azure-local-telemetry-ai-agent)
also reuses the embedding/Qdrant services with isolated SOP collections; it is
not deployed by this repository. Shared services must not be replaced to update
one application. Scenario isolation is not user/document authorization.

Continue with [scenario publication](./docs/SCENARIOS.md),
[code/runtime pipelines](./docs/DEVELOPMENT.md) or
[image/YAML deployment](./docs/DEPLOYMENT.md).

# Deploy the PDF assistant

Follow [prerequisites](PREREQUISITES.md) first. The appliance, Arc-enabled
Kubernetes cluster and Foundry/Qwen model services already exist; this workflow
adds only the assistant application. Commands run from the repository root.

## What is packaged

| Artifact | Contents / purpose |
|---|---|
| [Dockerfile](../Dockerfile), `frontend` stage | Node builds the React UI using the npm lockfile |
| Dockerfile, `app` target | Python/FastAPI, built UI and indexing code; reused by `assistant-web`, `documents` and indexing Jobs |
| Dockerfile, `embeddings` target | MiniLM/FastEmbed API with a pinned ONNX model provisioned into the image for offline runtime |
| Mirrored Qdrant image | Approved ACR copy of the pinned upstream database image; not built from this Dockerfile |
| [stack.yaml](../deploy/stack.yaml) | Namespace, Deployments, StatefulSet, Services, PVCs and NetworkPolicy |
| [index-job.yaml](../deploy/index-job.yaml) | One-off PDF indexing Job; the publisher supplies the actual image, package path, DNS and pull settings |
| [Model manifests](../deploy/) | Separate Foundry `ModelDeployment` resources; follow [Qwen CPU model deployment](PREREQUISITES.md#b8-platform-administrator-only-deploy-the-qwen-cpu-models) with platform-admin approval |
| Scenario ZIP | Definition, domain prompt and PDFs; a content release, **not** an image |

YAML describes the desired Kubernetes resources, not a self-contained appliance
installer. The [deployment tool](../tools/deploy_assistant.py) renders templates,
checks ownership/capacity/storage, adds namespaced configuration and credentials,
and applies through `kubectl` using an explicit context. **Do not directly apply
the placeholder stack or index Job.**

Model packaging is separate: Foundry Local's catalog/model-store artifacts and
CPU runtime are selected by `ModelDeployment` YAML, and the operator creates the
serving workload/endpoint. This lab walkthrough uses CPU-only Qwen 0.5B and 14B;
it does not build model weights into the assistant image or require a GPU.

Azure is not used *only* for ACR: Entra/Arc provide connected identity, management
and access too. ACR Tasks build Linux images remotely and workers pull them.
The app's PDF parsing, embeddings, retrieval, data storage and inference execute
on Azure Local. The [build-context allowlist](../.dockerignore) excludes documents,
credentials, local environments and generated output.

## 1. Prepare isolated application dependencies

Use Python 3.12, not Azure CLI's bundled Python:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements-dev.txt
if ($LASTEXITCODE -ne 0) { throw "Application dependency installation failed." }
.\.venv\Scripts\python.exe -m pip check
if ($LASTEXITCODE -ne 0) { throw "Application dependency validation failed." }
```

Run the [local tests](DEVELOPMENT.md#testing) before publishing code. Node is
optional locally; the production UI build runs in ACR.

## 2. Verify the existing cluster and capacity

With the Arc proxy running, confirm the intended context, Ready workers, model
services and Azure Local CSI storage using
[the runbook's read-only checks](ASSISTANT.md#step-1---read-only-environment-checks).
The assistant reserves 2,496 MiB including one indexer; initial deployment also
requires 1,024 MiB memory headroom on one eligible worker.

Do not reduce requests or change shared workloads if preflight fails. Ask the
lab owner to resolve capacity. A Ready node alone does not establish capacity.

## 3. Build and mirror images in the approved ACR

Follow [approved registry sign-in and the build commands](ASSISTANT.md#step-4---build-all-images-in-an-approved-acr).
[build_acr.ps1](../tools/build_acr.ps1) calls `az acr build` for `app` and
`embeddings`, imports pinned Qdrant, and records immutable image digests in
ignored `.local\acr-images.json`.

Use an existing owner-approved registry and isolated CLI profile when required.
Do not create registries, change networking/identities or use local Docker.
For later app-only upgrades, [rebuild only the application](ASSISTANT.md#subsequent-core-upgrades-rebuild-only-the-application);
the embedding and Qdrant images can remain unchanged.

## 4. Preflight, then deploy the assistant

Follow [registry pull access and guarded deployment](ASSISTANT.md#step-5---registry-pull-access-and-guarded-deployment):

1. Obtain approved pull credentials in memory; never print/save the token.
2. Pass immutable ACR image references to `tools.deploy_assistant`.
3. Run without `--apply` first. Review the result; only proceed if it passes.
4. Repeat with `--apply`, retaining every selected `--model-deployment` argument
   in the same order (first = default).
5. For an **existing installation**, use `--preserve-placement`; the tool refuses
   changed CPU/memory requests. Do not use it for initial deployment.

Only owned `assistant-demo` resources are written. The tool reuses the existing
model Services, reads model keys/CA in memory and creates the assistant's own
namespaced credentials. Existing Foundry/model resources are not redeployed.
Demo pull tokens expire in about three hours; refresh before later pulls.

## 5. Verify rollout and persistent storage

```powershell
kubectl --context zines-22325-aks rollout status deployment/assistant-web -n assistant-demo --timeout=240s
if ($LASTEXITCODE -ne 0) { throw "Assistant rollout failed." }
kubectl --context zines-22325-aks get pods,pvc -n assistant-demo
```

Expect the app, embeddings, documents and Qdrant pods Ready, and `pdf-documents`
and `qdrant-data` PVCs Bound. Verify the other workload rollouts and unchanged
model readiness as described in [the runbook](ASSISTANT.md#step-5---registry-pull-access-and-guarded-deployment).
Do not publish content before the core is healthy.

## 6. Publish content, then try the UI

Use [the first-demo workflow](SCENARIOS.md#first-demo-operations-or-contoso).
It runs parsing, chunking and embeddings in an indexing Job on Azure Local;
it does not put PDFs into ACR.

Then [open the application](ASSISTANT.md#step-7---open-the-deployed-application)
and [ask/inspect sample answers](SCENARIOS.md#4-select-and-demonstrate).
Check citations, model exchanges and an unsupported question, not just pod status.

## Safe recovery

Keep PVCs, model services and other scenarios intact. A failed indexing release
must not replace the previous searchable index. Inspect the named Job and
publication lock using [scenario recovery](SCENARIOS.md#updating-and-recovering-a-scenario).
For a code rollback, restore approved previous image digests through the same
guarded upgrade path. Never delete the namespace, storage or platform to recover.

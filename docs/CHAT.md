# Original standard-library Qwen chat

This separate demo talks directly to the existing 0.5B model. It does not use
PDFs, MiniLM, Qdrant or the React application. Python 3.9+ and kubectl are needed;
there are no application packages to install.

For the PDF assistant, start at [the README](../README.md).

## Run with three terminals

After [workstation and Arc prerequisites](PREREQUISITES.md), keep terminal 1 open:

```powershell
az connectedk8s proxy --resource-group zines-22325-rg --name zines-22325-aks
```

Terminal 2:

```powershell
kubectl --context zines-22325-aks port-forward `
    svc/qwen2-5-0-5b-cpu -n foundry-local-operator 5000:5000
```

Terminal 3, from the repository root:

```powershell
python .\chat.py --insecure
```

The client uses the current kubectl context; confirm it is `zines-22325-aks`.
It reads the model key/name through kubectl without printing or saving the key.
`--insecure` skips TLS certificate verification; use only with the trusted lab
port-forward. It does not make model traffic plaintext. Omit it when Python
trusts the endpoint certificate.

Commands: `/reset`, `/history`, `/exit` (or `/quit`). History is memory-only;
failed turns are not saved. Ctrl+C closes the local client/tunnels, not workloads.

For a slower model: `python .\chat.py --insecure --timeout 300 --max-tokens 512`.
`--timeout` defaults to 180 seconds; `--max-tokens` defaults to 256 per response.
Use `python .\chat.py --help` for options.

## Check follow-up history and test locally

Run `python .\chat.py --insecure --debug`, then ask:

```text
My favorite fruit is mango. Remember this for my next question. Reply with OK.
What is my favorite fruit? Reply with only the fruit name.
```

Expect 2 messages in the first request and 4 in the second. Debug output reports
roles/counts, not keys/message content; `/history` deliberately displays turns.
Model failure to recall a fact does not prove history was dropped. `/reset`
helps when a long conversation exceeds the model's context capacity.

```powershell
python -B -m unittest -v test_chat
```

Tests mock kubectl and inference; they do not contact the cluster or read real keys.
For CLI import failures, use [version-preserving workstation troubleshooting](PREREQUISITES.md#troubleshooting-without-changing-the-lab),
not changes to the cluster or blanket extension upgrades.

## Detailed model transport

[Open the preserved transport diagram](../architecture-detail.svg) or
[download its PNG](../architecture-detail.png).

| Hop | Role |
|---|---|
| Python -> localhost:5000 | Sends messages and a model API key |
| kubectl port-forward -> local Arc proxy | Resolves the model Service to a pod/target port |
| Azure Arc Cluster Connect -> cluster agents/API server | Connected, Entra/RBAC-authorized Kubernetes access |
| Worker kubelet/runtime -> model pod:8443 | Carries the port-forward stream |
| NGINX sidecar -> inference container:5000 | TLS front end -> local ONNX GenAI CPU execution |

Service port 5000 maps to pod port 8443. Port-forward uses the Service to select a
pod and port; forwarded bytes do **not** traverse the Service ClusterIP.
The API server participates in this development tunnel, not normal in-cluster
model inference. Responses return through the same connected Arc path.

Cluster Entra authorization and the model API key are separate authentication
layers. Local inference does not mean transport stays entirely on-premises.

### Installed Istio/Envoy alternative

The platform also has an internal gateway:

```text
In-cluster caller -> inference-internal-gateway-istio:443
  -> HTTPRoute /qwen2-5-0-5b-cpu (prefix rewritten)
  -> model Service:5000 -> model pod:8443 -> inference:5000
```

The chat request path is `/qwen2-5-0-5b-cpu/v1/chat/completions`.
**This Python port-forward demo bypasses that gateway.** Gateway API objects
are configuration, not running proxies. `istiod` in `istio-system` configures
the `istio-proxy` (Envoy) gateway pod in `foundry-local-operator`; the model pod
has NGINX, not an observed Envoy sidecar.

### Model provisioning is not inference

The local catalog identifies a CPU asset/version. The Foundry operator reconciles
`ModelDeployment` and provisions its Service, routing, keys, certificates and
model-store resources. A serving pod's `model-store-pull` startup container
retrieves cached artifacts; the dedicated `inference` container executes Qwen.
The operator and model store do not generate answer tokens.

The original lab diagram recorded Qwen 0.5B asset version 4, one replica,
2 CPU / 6 GiB requested, 4 CPU / 10 GiB limits and approximately 0.80 GiB of model
files. These are historical model settings, not permission to resize the platform
or a measurement of total runtime memory. No physical host topology was established.
See [current application/platform roles](../ARCHITECTURE.md#pods-and-containers)
and [the deployment record](ASSISTANT.md) for the later assistant and 14B additions.

## Lab references

- [Install Foundry Local](https://docs.almr.adaptivecloudlab.com/eng-22325/foundry-local/deploy-foundry-local.html?tabs=powershell)
- [Deploy a CPU model](https://docs.almr.adaptivecloudlab.com/eng-22325/foundry-local/deploy-model.html?tabs=powershell)
- [Run inference](https://docs.almr.adaptivecloudlab.com/eng-22325/foundry-local/run-inference.html?tabs=powershell)

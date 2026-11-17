# AgentOps Unlocked

Companion repo for the **AgentOps Unlocked** video series.

_"I built this on my laptop, I shipped it to production, I trust it in production."_

This repo contains a LangGraph ReAct agent that evolves across the series. We start with a local prototype running on Ollama and end with a production-grade deployment on Kubernetes with sandboxing, identity, and observability.

---

## Prerequisites

- [uv](https://docs.astral.sh/uv/) - Python package manager
- [GNU Make](https://www.gnu.org/software/make/) and a bash-compatible shell (Windows: use [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) or [Git Bash](https://git-scm.com/downloads))
- [Ollama](https://ollama.com/) - local inference (Video 2)
- [Podman](https://podman.io/) or [Docker](https://www.docker.com/) - container builds (Video 3+)
- [Helm](https://helm.sh/) - Kubernetes deployment (Video 3+)
- [oc CLI](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html) - OpenShift deployment (Video 3+)

---

## Series Overview

| # | Video | What You Build | Repo Content |
|---|-------|---------------|--------------|
| 1 | [Is Your AI Agent a Liability?](https://youtu.be/LJHn6a_z9e0) | - | No code (conceptual framing) |
| 2 | [Run Local AI Agents for Free](https://youtu.be/i4A8jN41NNs?si=3A20z_qRYWUibSGb) | Working local agent | Agent code, Ollama/OGX setup, MCP tools |
| 3 | [Deploy AI Agents on Kubernetes](https://youtu.be/6MuAOFfJk2w?si=bZaKmpAwPeAEMi12) | Containerized agent on OpenShift | Dockerfile, Helm chart, deploy targets |
| 4 | [Sandbox Your AI Agent](https://youtu.be/sXXYJQ-3mXs?si=dPjoElvgrTs-fiaM) | Sandboxed agent with OpenShell | OpenShell Helm values |
| 5 | [MCP Gateway Tool Authorization](https://youtu.be/2M3SByYGuZk?si=Yox42PzVyLYhh7Nn) | Identity-based tool authorization | MCP Gateway manifests |
| 6 | [LLM Observability](https://youtu.be/AWyvtTN8zjY?si=fapWK7mdpyKI6a3v) | MLflow tracing | Tracing config |

---

## Video 1 - Is Your AI Agent a Liability?

This is the series opener. It walks through the evolution of the AI stack, from chat completions to autonomous agents, and establishes the production gap. Every subsequent video references this framing.

---

## Video 2 - Run Local AI Agents for Free (Ollama + Qwen + MCP)

Build a working AI agent on your laptop using Ollama for local inference and MCP for tool connectivity.

### Files

| Path | Purpose |
|------|---------|
| `src/react_agent/agent.py` | LangGraph ReAct agent |
| `src/react_agent/tools.py` | Calculator, fetch_page tools, MCP server config |
| `main.py` | FastAPI app with OpenAI-compatible `/chat/completions` endpoint |
| `run_ogx_server.yaml` | OGX local server config (Ollama provider) |
| `playground/` | Browser-based chat UI |

### Setup

```bash
make init                     # create .env from .env.example
make env                      # create venv, install dependencies
make ollama MODEL=qwen3:1.7b  # install Ollama, pull model
```

Edit `.env`:

```ini
BASE_URL=http://localhost:8321/v1
MODEL_ID=ollama/qwen3:1.7b
```

> The `MODEL_ID` must use the `ollama/` prefix and match `ollama list` exactly.

### Run

```bash
make ogx-server   # terminal 1 - keep open
make run-app      # terminal 2 - agent at http://localhost:8000
```

### Test

```bash
curl -X POST http://localhost:8000/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "What is 2+2?"}], "stream": false}'
```

---

## Video 3 - Deploying AI Agents on Kubernetes: Ollama to vLLM

This video uses the same agent, containerized and deployed to OpenShift. Inference moves from Ollama to vLLM for multi-user throughput. **The agent code does not change.**

### Files

| Path | Purpose |
|------|---------|
| `Dockerfile` | UBI9 Python 3.12 container image |
| `deployment/` | Helm chart (`Chart.yaml`, `templates/`) |
| `values.yaml` | Agent-specific Helm overrides |
| `Makefile` | `build`, `push`, `deploy`, `dry-run`, `undeploy` targets |

### Configure

Edit `.env` for your cluster:

```ini
API_KEY=your-api-key
BASE_URL=https://your-vllm-endpoint/v1
MODEL_ID=Qwen3-8B-FP8-dynamic
CONTAINER_IMAGE=quay.io/your-username/langgraph-react-agent:latest
```

### Build and Deploy

```bash
make build        # build container image (podman/docker)
make push         # push to registry
make deploy       # helm install to current namespace
```

Or build in-cluster without podman/docker:

```bash
make build-openshift
```

### Verify

```bash
oc get route langgraph-react-agent -o jsonpath='{.spec.host}'
curl -k https://<route>/health
```

---

## Video 4 - Sandbox Your AI Agent: OpenShell

Video 3 deployed the agent to OpenShift. Video 4 wraps it with sandbox policy—restricting file access, network egress, and system calls. Via [OpenShell](https://github.com/NVIDIA/OpenShell).

### Prerequisites

Install Red Hat Agent Sandbox Operator (cluster-wide, one-time):

```bash
# Create operator namespace and install via OLM
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-agent-sandbox-operator
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: agent-sandbox-operatorgroup
  namespace: openshift-agent-sandbox-operator
spec:
  targetNamespaces:
  - openshift-agent-sandbox-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: agent-sandbox-operator
  namespace: openshift-agent-sandbox-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: agent-sandbox-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Verify operator is running
oc -n openshift-agent-sandbox-operator get pods
```

See [openshell/SETUP.md](openshell/SETUP.md) for OpenShift namespace and SCC setup.

### Deploy

```bash
helm install openshell oci://ghcr.io/nvidia/openshell/helm-chart \
  --version 0.1.0 \
  --namespace openshell \
  -f openshell/values.yaml
```

### Verify

```bash
oc rollout status statefulset/openshell -n openshell
oc port-forward -n openshell svc/openshell 8080:8080 &
openshell gateway add http://127.0.0.1:8080 --local --name openshift
```

### Files

| Path | Purpose |
|------|---------|
| `openshell/SETUP.md` | Prerequisites, detailed walkthrough, troubleshooting |
| `openshell/values.yaml` | Sandbox policy (filesystem, network, process, inference) |

---

## Video 5 - MCP Gateway Tool Authorization

In this video we deploy an MCP Gateway (Kuadrant-based) that sits in front of tool servers and authorizes each tool call against the caller's identity. A "readonly" key can call `github_issue_read` but is denied on `github_issue_write`; a "readwrite" key can call both.

### Files

| Path | Purpose |
|------|---------|
| `deployment/mcp_gateway/kuadrant-operator.yaml` | Kuadrant operator + CatalogSource |
| `deployment/mcp_gateway/kuadrant-cr.yaml` | Activate Kuadrant |
| `deployment/mcp_gateway/gateway.yaml` | Gateway + ReferenceGrant |
| `deployment/mcp_gateway/mcp-gateway-values.yaml` | MCP Gateway Helm values |
| `deployment/mcp_gateway/reference-grant.yaml` | ReferenceGrant for MCPGatewayExtension |
| `deployment/mcp_gateway/agent-authpolicy.yaml` | Tool-level AuthPolicy (readonly vs readwrite) |
| `deployment/mcp_gateway/agent-identities.yaml` | Demo API key secrets |
| `deployment/github-mcp-server.yaml` | GitHub MCP server Deployment + Service |
| `deployment/github-mcp-registration.yaml` | HTTPRoute + MCPServerRegistration |

### Deploy (MCP Gateway)

```bash
# 1. Kuadrant
oc apply -f deployment/mcp_gateway/kuadrant-operator.yaml
# wait for all 4 CSVs to be Succeeded:
oc get csv -n kuadrant-system -w

# 2. Kuadrant CR
oc apply -f deployment/mcp_gateway/kuadrant-cr.yaml
oc wait kuadrant/kuadrant -n kuadrant-system --for=condition=Ready --timeout=300s

# 4. GatewayClass — triggers auto-provisioning of Istio in openshift-ingress
oc apply -f deployment/mcp_gateway/gatewayclass.yaml

# wait for the auto-installed Istio to be ready:
oc get istio -n openshift-ingress -w

# 5. Gateway
oc apply -f deployment/mcp_gateway/gateway.yaml

# wait for the auto-provisioned gateway deployment:
oc get deployment -n openshift-ingress -w

# 6. ReferenceGrant + MCP Gateway
oc apply -f deployment/mcp_gateway/reference-grant.yaml

export MCP_GATEWAY_VERSION=0.7.1
kubectl apply -k "https://github.com/kuadrant/mcp-gateway/config/crd?ref=v${MCP_GATEWAY_VERSION}"
helm upgrade -i mcp-gateway oci://ghcr.io/kuadrant/charts/mcp-gateway \
  --version ${MCP_GATEWAY_VERSION} \
  -f deployment/mcp_gateway/mcp-gateway-values.yaml \
  -n mcp-system --create-namespace

# 7. Register GitHub MCP server
oc create secret generic github-mcp-token \
  --from-literal=GITHUB_TOKEN=<your-token> -n mcp-system
oc apply -f deployment/github-mcp-server.yaml
oc apply -f deployment/github-mcp-registration.yaml

# 8. Authorization policy + demo identities 
oc apply -f deployment/mcp_gateway/agent-authpolicy.yaml
# edit agent-identities.yaml with real API keys first
oc apply -f deployment/mcp_gateway/agent-identities.yaml
```

---

## Video 6 - LLM Observability: Monitor and Trace AI Agents

This video demos MLflow Tracing capturing every prompt, reasoning step, tool invocation, and token cost.

### Files

| Path | Purpose |
|------|---------|
| `src/react_agent/tracing.py` | MLflow tracing setup |
| `.env.example` | MLflow config (local and OpenShift sections) |

### Enable Tracing

Add to `.env`:

```ini
MLFLOW_TRACKING_URI="http://localhost:5000"
MLFLOW_EXPERIMENT_NAME="langgraph-react-agent"
```

Start MLflow:

```bash
uv run --extra tracing mlflow server --port 5000
```

For OpenShift (in-pod with K8s service account auth):

```ini
MLFLOW_TRACKING_URI=https://mlflow.redhat-ods-applications.svc:8443/mlflow
MLFLOW_EXPERIMENT_NAME=langgraph-react-agent
MLFLOW_TRACKING_AUTH=kubernetes-namespaced
MLFLOW_TRACKING_SERVER_CERT_PATH=/var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
```

Tracing is optional. If `MLFLOW_TRACKING_URI` is not set, the agent runs without it.

---

## API Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/chat/completions` | POST | OpenAI-compatible chat (supports `stream: true`) |
| `/health` | GET | Service health check |
| `/` | GET | Playground chat UI (disabled when auth is enabled) |

---

## Tests

```bash
make test                  # unit tests
make test-integration      # deployment integration tests
make test-auth-integration # auth integration tests
```

---

## Resources

- [LangGraph](https://langchain-ai.github.io/langgraph/)
- [LangChain](https://python.langchain.com/)
- [OGX](https://ogx-ai.github.io/docs/)
- [Ollama](https://ollama.com/docs)
- [MCP Gateway](https://github.com/Kuadrant/mcp-gateway)
- [OpenShell](https://github.com/NVIDIA/OpenShell)

## License

See [LICENSE](LICENSE).

# Open WebUI on k3s (DGX Spark)

This guide captures the **working, end-to-end steps** to install **Open WebUI** on a **k3s** cluster running on an **NVIDIA DGX Spark** and connect it to a **vLLM** model endpoint running inside the same cluster.

It includes:

- prerequisites
- installation steps
- verification checkpoints
- common errors encountered + fixes

---

## Target outcome

- Open WebUI running in Kubernetes (k3s)
- Exposed via NodePort (example: **30080**)
- Connected to vLLM using OpenAI-compatible API:
  - Base URL: `http://qwen3-4b-instruct-vllm.llm.svc.cluster.local:8000/v1`
  - Model id shown in UI: `qwen3-4b-instruct`

---

## Prerequisites

### 1) Cluster health

Run:

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
kubectl get storageclass
```

Verify:

- Node(s) are **Ready**
- Core system pods are healthy
- A default StorageClass exists (k3s often uses **local-path**)

---

## Verify vLLM (OpenAI-compatible) before installing Open WebUI

This is the most important prerequisite: **prove vLLM is reachable from inside the cluster**.

### 1) Start a temporary curl pod

```bash
kubectl -n llm run curltest --rm -it --image=curlimages/curl:8.5.0 -- sh
```

### 2) Verify models endpoint

```sh
curl -sS http://qwen3-4b-instruct-vllm.llm.svc.cluster.local:8000/v1/models | head
```

Expected:

- HTTP 200
- JSON output containing:
  - `"id":"qwen3-4b-instruct"`

### 3) Verify chat completions endpoint

```sh
curl -sS http://qwen3-4b-instruct-vllm.llm.svc.cluster.local:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-4b-instruct",
    "messages": [{"role":"user","content":"Say hello in one short sentence."}],
    "temperature": 0.2
  }' | head
```

Expected:

- JSON response with `object: "chat.completion"`
- A response under `choices[0].message.content`

If these succeed, Open WebUI will be able to list the model and chat.

Exit the pod:

```sh
exit
```

---

## Install Helm on Ubuntu (DGX Spark)

### Option A: Install Helm via APT (what worked)

#### 1) Install prerequisites

```bash
sudo apt-get update
sudo apt-get install -y curl gnupg
sudo mkdir -p /usr/share/keyrings
```

#### 2) Add the Helm repository

```bash
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" \
  | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
```

#### 3) Install the missing GPG key (critical step)

```bash
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
```

Verify:

```bash
ls -l /usr/share/keyrings/helm.gpg
gpg --show-keys /usr/share/keyrings/helm.gpg | head
```

#### 4) Update & install helm

```bash
sudo apt-get update
sudo apt-get install -y helm
helm version
which helm
```

### Common Helm install error: NO\_PUBKEY

Symptom during `apt-get update`:

- `NO_PUBKEY 4B196BE9C4313D06`
- Repo “not signed”

Fix:

- Install the Helm repo GPG key (step above) into `/usr/share/keyrings/helm.gpg`

---

## Ensure Helm can reach the k3s cluster

### Symptom

Running Helm shows:

- `Kubernetes cluster unreachable: Get "http://localhost:8080/version": connect: connection refused`

`kubectl cluster-info` still works (API server at `https://127.0.0.1:6443`).

### Fix

Helm is not using the correct kubeconfig.

#### Recommended: copy k3s kubeconfig to your user

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
```

Verify Helm connectivity:

```bash
helm ls -A
```

Expected:

- You should see k3s-installed charts (example: `traefik`, `traefik-crd`).

---

## Install Open WebUI via Helm (into the llm namespace)

### 1) Add Open WebUI chart repo

```bash
helm repo add open-webui https://helm.openwebui.com/
helm repo update
helm search repo open-webui
```

Example result:

- `open-webui/open-webui` (chart 10.1.0, app 0.7.1)

### 2) Confirm target namespace

This guide uses `llm` namespace.

Verify:

```bash
kubectl get ns llm
```

Create if missing:

```bash
kubectl create ns llm
```

### 3) Create values file

Create `open-webui-values.yaml`:

```yaml
ollama:
  enabled: false

pipelines:
  enabled: false

persistence:
  enabled: true
  storageClass: local-path
  size: 10Gi

service:
  type: NodePort
  port: 8080
  nodePort: 30080

extraEnvVars:
  - name: OPENAI_API_BASE_URL
    value: "http://qwen3-4b-instruct-vllm.llm.svc.cluster.local:8000/v1"
  - name: OPENAI_API_KEY
    value: "sk-no-key"
  - name: WEBUI_AUTH
    value: "false"
```

Notes:

- `WEBUI_AUTH=false` disables login.
- `OPENAI_API_KEY` is a dummy key for endpoints that don’t enforce auth.

### 4) Install / upgrade

```bash
helm upgrade --install open-webui open-webui/open-webui \
  -n llm \
  -f open-webui-values.yaml
```

---

## Verify Open WebUI deployment

### 1) Helm status

```bash
helm -n llm status open-webui
helm -n llm list
```

Expected:

- STATUS: deployed

### 2) Kubernetes resources

```bash
kubectl get all -n llm
```

Expected:

- `pod/open-webui-0` Running
- `pod/open-webui-redis-...` Running
- `service/open-webui` NodePort 8080:30080

### 3) Confirm Open WebUI can reach vLLM

From inside the Open WebUI pod:

```bash
kubectl -n llm exec -it pod/open-webui-0 -- sh -lc \
  'echo "OPENAI_API_BASE_URL=$OPENAI_API_BASE_URL"; wget -qO- "$OPENAI_API_BASE_URL/models" | head'
```

Expected:

- prints base URL
- JSON includes `qwen3-4b-instruct`

### 4) Access UI via NodePort

Get node IP:

```bash
kubectl get nodes -o wide
```

Open in browser:

- `http://<NODE_IP>:30080`

Verify in UI:

- Model dropdown shows `qwen3-4b-instruct`
- Chat produces responses

---

## Why NodePort doesn’t show up in `ss -pltn`

NodePort is often implemented by **iptables/nftables NAT rules**, not a userspace process binding to the port.

So `ss -pltn` may not show `30080` even though it works.

### Verify NodePort rules

```bash
sudo iptables -t nat -S | grep -E 'KUBE-NODEPORTS|30080'
```

You should see rules for `llm/open-webui:http` on `--dport 30080`.

### Confirm it’s reachable

From the node:

```bash
curl -I http://127.0.0.1:30080/ | head
curl -I http://<NODE_IP>:30080/ | head
```

From another machine on the same network:

```bash
curl -I http://<NODE_IP>:30080/ | head
# or
nc -vz <NODE_IP> 30080
```

If remote access fails but local works, check firewall (UFW/iptables) and allow 30080/tcp.

---

## Notable gotchas and recommendations

### 1) Duplicate OPENAI env vars (chart defaults + overrides)

The chart may inject defaults (`https://api.openai.com/v1` and a default key) in addition to `extraEnvVars`.

If you want a clean single source of truth, set these in your values file and remove the duplicate env vars:

```yaml
openaiBaseApiUrl: "http://qwen3-4b-instruct-vllm.llm.svc.cluster.local:8000/v1"
openaiApiKey: "sk-no-key"
extraEnvVars:
  - name: WEBUI_AUTH
    value: "false"
```

Then apply:

```bash
helm upgrade open-webui open-webui/open-webui -n llm -f open-webui-values.yaml
kubectl -n llm rollout status statefulset/open-webui
```

### 2) Old / unhealthy vLLM pods

You may see stale pods with states like `ContainerStatusUnknown` or `UnexpectedAdmissionError`. If the active vLLM pod is Running and the service works, Open WebUI will still function. Consider cleanup later to reduce noise.

---

## Quick “known good” command bundle

```bash
# Verify vLLM endpoints
kubectl -n llm run curltest --rm -it --image=curlimages/curl:8.5.0 -- sh -lc \
  'curl -sS http://qwen3-4b-instruct-vllm.llm.svc.cluster.local:8000/v1/models | head'

# Install Open WebUI
helm repo add open-webui https://helm.openwebui.com/
helm repo update
helm upgrade --install open-webui open-webui/open-webui -n llm -f open-webui-values.yaml

# Verify
kubectl get all -n llm
kubectl -n llm exec -it pod/open-webui-0 -- sh -lc 'wget -qO- "$OPENAI_API_BASE_URL/models" | head'
```

---

## Appendix: what each extraEnvVar means

- `OPENAI_API_BASE_URL`

  - Base URL Open WebUI uses to call an OpenAI-compatible server.
  - Open WebUI calls `$OPENAI_API_BASE_URL/models` and `$OPENAI_API_BASE_URL/chat/completions`.

- `OPENAI_API_KEY`

  - Sent as `Authorization: Bearer <key>`.
  - Many in-cluster vLLM deployments don’t enforce auth; use a dummy value.

- `WEBUI_AUTH`

  - When `false`, disables login/auth in the UI.
  - Recommended to keep auth enabled if exposing beyond trusted networks.


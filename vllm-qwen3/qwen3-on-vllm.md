# Deploy Qwen3-4B-Instruct with vLLM on Kubernetes (K3s) — Reference Manual

This document captures the **final working configuration** and the key lessons learned while deploying **Qwen/Qwen3-4B-Instruct-2507** with **vLLM** on a GPU-enabled Kubernetes cluster (K3s).

---

## 1) What you are deploying

- **Model:** `Qwen/Qwen3-4B-Instruct-2507`
- **Serving runtime:** vLLM
- **Container image:** `nvcr.io/nvidia/vllm:25.11-py3`
- **API:** OpenAI-compatible endpoints (e.g., `/v1/chat/completions`)
- **Cluster assumptions:**
  - K3s (or Kubernetes) running
  - NVIDIA GPU available on the node
  - NVIDIA runtime class configured (e.g., `runtimeClassName: nvidia`)
  - StorageClass `local-path` present (K3s default)

---

## 2) Kubernetes object definitions (quick reference)

### PersistentVolumeClaim (PVC)

A **request for storage** by a Pod. The PVC is bound to a PersistentVolume (PV) created by a storage provisioner (e.g., K3s `local-path-provisioner`).

**Why we use it:** persist Hugging Face model downloads + vLLM compile artifacts across restarts.

### Deployment

A controller that **manages a ReplicaSet**, ensuring the desired number of Pods stay running. Handles rollouts and rollbacks.

**Why we use it:** keep exactly one GPU-backed vLLM Pod running with a stable spec.

### Service

A stable **virtual IP + DNS name** that load-balances traffic to matching Pods via label selectors.

**Why we use it:** access vLLM via `qwen3-4b-instruct-vllm.llm.svc.cluster.local:8000` (ClusterIP) from inside the cluster.

---

## 3) Final YAML manifests (as used)

> Apply these in the order shown: PVC → Deployment → Service.

### 3.1 PersistentVolumeClaim: `hf-cache`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hf-cache
  namespace: llm
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 100Gi
  storageClassName: local-path
```

### 3.2 Deployment: `qwen3-4b-instruct-vllm`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qwen3-4b-instruct-vllm
  namespace: llm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: qwen3-4b-instruct-vllm
  template:
    metadata:
      labels:
        app: qwen3-4b-instruct-vllm
    spec:
      runtimeClassName: nvidia
      terminationGracePeriodSeconds: 120
      containers:
      - name: vllm
        image: nvcr.io/nvidia/vllm:25.11-py3
        imagePullPolicy: IfNotPresent
        command: ["vllm", "serve"]
        args:
          - "Qwen/Qwen3-4B-Instruct-2507"
          - "--served-model-name"
          - "qwen3-4b-instruct"
          - "--dtype"
          - "bfloat16"
          - "--max-model-len"
          - "8192"
          - "--gpu-memory-utilization"
          - "0.85"
          - "--host"
          - "0.0.0.0"
          - "--port"
          - "8000"

        ports:
          - name: http
            containerPort: 8000

        resources:
          limits:
            nvidia.com/gpu: 1
          requests:
            cpu: "2"
            memory: "16Gi"
            nvidia.com/gpu: 1

        env:
          - name: HF_HOME
            value: /root/.cache/huggingface
          - name: HF_DATASETS_CACHE
            value: /root/.cache/huggingface/datasets

        volumeMounts:
          - name: cache
            mountPath: /root/.cache
          - name: dshm
            mountPath: /dev/shm

        startupProbe:
          httpGet:
            path: /health
            port: http
          periodSeconds: 3
          failureThreshold: 200

        readinessProbe:
          httpGet:
            path: /health
            port: http
          periodSeconds: 5
          failureThreshold: 3

        livenessProbe:
          httpGet:
            path: /health
            port: http
          periodSeconds: 10
          failureThreshold: 6

      volumes:
        - name: cache
          persistentVolumeClaim:
            claimName: hf-cache
        - name: dshm
          emptyDir:
            medium: Memory
            sizeLimit: 8Gi
```

### 3.3 Service: `qwen3-4b-instruct-vllm`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: qwen3-4b-instruct-vllm
  namespace: llm
spec:
  selector:
    app: qwen3-4b-instruct-vllm
  ports:
    - name: http
      port: 8000
      targetPort: http
  type: ClusterIP
```

---

## 4) Apply + verify steps

### 4.1 Create namespace (if needed)

```bash
kubectl create ns llm
```

### 4.2 Apply manifests

```bash
kubectl apply -f pvc.yaml -n llm
kubectl apply -f deployment.yaml -n llm
kubectl apply -f service.yaml -n llm
```

### 4.3 Verify PVC is bound

```bash
kubectl get pvc -n llm
kubectl describe pvc hf-cache -n llm
```

**Expected:** `STATUS: Bound`

### 4.4 Verify Pod is running

```bash
kubectl get pods -n llm -o wide
kubectl rollout status deploy/qwen3-4b-instruct-vllm -n llm
```

### 4.5 Verify health endpoint

From inside cluster (or from node if you port-forward):

```bash
kubectl port-forward -n llm deploy/qwen3-4b-instruct-vllm 8000:8000
curl -sS http://localhost:8000/health
```

**Expected:** HTTP 200

---

## 5) Send a test inference request

### 5.1 Minimal “hello” test

```bash
curl -sS http://10.42.0.49:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-4b-instruct",
    "messages": [{"role":"user","content":"Reply with exactly: hello there. This is Qwen3. How can i help? "}],
    "max_tokens": 1024,
    "temperature": 0
  }' | jq -r '.choices[0].message.content'
```

**Expected output:**

```
hello there. This is Qwen3. How can i help?
```

### 5.2 Inspect the full JSON response

```bash
curl -sS http://10.42.0.49:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3-4b-instruct",
    "messages": [{"role":"user","content":"Reply with exactly: hello there. This is Qwen3. How can i help? "}],
    "max_tokens": 1024,
    "temperature": 0
  }' | jq
```

Look for:

- `.choices[0].message.content`
- `.usage.prompt_tokens`, `.usage.completion_tokens`, `.usage.total_tokens`

---

## 6) Cache behavior: what to expect + how to verify

### 6.1 What the cache includes

With the PVC mounted at `/root/.cache`, you persist:

- Hugging Face model files (weights/tokenizer/config)
- HF hub metadata
- vLLM torch.compile cache
- flashinfer artifacts (if created)

### 6.2 Verify the cache is actually mounted

Exec into the pod and check mounts:

```bash
kubectl exec -n llm deploy/qwen3-4b-instruct-vllm -- sh -lc 'mount | grep -E "(/root/.cache|hf-cache)"'
```

You should see `/root/.cache` mounted from the PVC volume.

### 6.3 Verify cache contents exist

```bash
kubectl exec -n llm deploy/qwen3-4b-instruct-vllm -- sh -lc \
  'ls -lah /root/.cache; \
   ls -lah /root/.cache/vllm || true; \
   ls -lah /root/.cache/huggingface || true'
```

Typical directories to see:

- `/root/.cache/huggingface/models--Qwen--Qwen3-4B-Instruct-2507`
- `/root/.cache/vllm/torch_compile_cache/`

### 6.4 Log lines that indicate cache usage

**Torch compile cache directory created/used:**

```
Using cache directory: /root/.cache/vllm/torch_compile_cache/... for vLLM's torch.compile
```

**If weights were downloaded (first run):**

```
Time spent downloading weights for Qwen/Qwen3-4B-Instruct-2507: ... seconds
```

On a subsequent run (with warm cache), you should see **less or no download time**.

---

## 7) Key log snippets to keep an eye on

### 7.1 Good signals (expected)

- vLLM starts and exposes API:
  ```
  Starting vLLM API server ... on http://0.0.0.0:8000
  ```
- Health checks succeed:
  ```
  "GET /health HTTP/1.1" 200 OK
  ```
- Model loads successfully:
  ```
  Starting to load model Qwen/Qwen3-4B-Instruct-2507...
  Loading safetensors checkpoint shards: 100% Completed
  ```
- Torch compile cache location:
  ```
  Using cache directory: /root/.cache/vllm/torch_compile_cache/...
  ```

### 7.2 Warnings that are usually OK

- Transformers cache env var deprecation warning:
  ```
  FutureWarning: Using `TRANSFORMERS_CACHE` is deprecated ... Use `HF_HOME` instead.
  ```
- GPU autotune warning:
  ```
  Not enough SMs to use max_autotune_gemm mode
  ```

---

## 8) Troubleshooting guide (including what went wrong before)

### Scenario A — PVC stuck in **Pending**

**Symptom:** `kubectl describe pvc hf-cache -n llm` shows `Status: Pending`

**Typical causes:**

- StorageClass mismatch (PVC requests a class that doesn’t exist)
- Provisioner not running

**What to check:**

```bash
kubectl get storageclass
kubectl -n kube-system get pods | grep -i local-path
kubectl describe pvc hf-cache -n llm
```

**Fix:**

- Use the correct StorageClass name (`local-path` on K3s by default)
- Ensure `local-path-provisioner` is healthy

---

### Scenario B — Volume is not mounted where you think it is

**Symptom:** You exec into the pod and see files writing fine, but it’s actually the container filesystem (overlay), not the PVC.

**What we learned:**

- `df` can be misleading if you only look at `/`.
- Use `mount` or `/proc/mounts` for truth.

**Verify mount (best):**

```bash
kubectl exec -n llm deploy/qwen3-4b-instruct-vllm -- sh -lc 'cat /proc/mounts | grep -E "/root/.cache|/cache"'
```

---

### Scenario C — Rollout stuck: “old replicas pending termination”

**Symptom:**

```
Waiting for deployment ... rollout to finish: 1 old replicas are pending termination...
```

**Typical causes:**

- The new pod can’t schedule (GPU constraint)
- The old pod won’t terminate fast enough
- `maxSurge` creates a second pod but there is only **one GPU**

**Fix options:**

1. **Delete the old pod** to free the single GPU:

```bash
kubectl delete pod -n llm <old-pod-name>
```

2. Change Deployment strategy to avoid creating two pods during rollout:

- Set `strategy.rollingUpdate.maxSurge: 0`
- Set `strategy.rollingUpdate.maxUnavailable: 1`

**YAML snippet (recommended for single-GPU nodes):**

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
```

---

### Scenario D — Pod stuck in Pending (GPU scheduling)

**Symptom:** Pod stays Pending; `kubectl describe pod ...` shows insufficient resources.

**What to check:**

```bash
kubectl describe pod -n llm <pod>
kubectl get nodes -o wide
kubectl describe node <node>
```

**Typical reasons:**

- GPU requested but not available
- NVIDIA device plugin not running
- Wrong runtimeClass

---

### Scenario E — “Not enough SMs to use max\_autotune\_gemm mode”

**Meaning:**

- **SMs = Streaming Multiprocessors**, the GPU compute units.
- PyTorch Inductor decides your GPU doesn’t have enough SMs to justify an expensive tuning mode.

**Impact:**

- Usually **not an error**. It’s a performance-mode fallback.
- Inference still works; your logs show the server came up and requests succeeded.

---

## 9) Operational tips

- Keep `/dev/shm` as `emptyDir: medium: Memory` for better performance (already done).
- First run will be slower due to:
  - HF download
  - torch.compile
  - CUDA graph capture
- Subsequent restarts should be faster when:
  - HF weights are already in `/root/.cache/huggingface`
  - vLLM compile cache is present in `/root/.cache/vllm/torch_compile_cache`

---

## 10) Handy commands (copy/paste)

### View everything in the namespace

```bash
kubectl get all -n llm -o wide
```

### Tail logs

```bash
kubectl logs -n llm deploy/qwen3-4b-instruct-vllm -f
```

### Exec and inspect cache

```bash
kubectl exec -n llm deploy/qwen3-4b-instruct-vllm -- sh -lc 'ls -lah /root/.cache'
```

### Check mounts

```bash
kubectl exec -n llm deploy/qwen3-4b-instruct-vllm -- sh -lc 'mount | grep -E "/root/.cache|/cache"'
```

---

## 11) Notes about the final working setup

- PVC `hf-cache` is mounted at `/root/.cache`.
- HF cache paths are explicitly set with `HF_HOME` and `HF_DATASETS_CACHE`.
- vLLM also writes compile cache under `/root/.cache/vllm/...` which is persisted.
- The Service is ClusterIP; access it from within the cluster or port-forward from your workstation.






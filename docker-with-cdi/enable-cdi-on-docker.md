# DGX Spark GPU Containers: Enable + Verify Both Modes (CDI + `--gpus`)

Use this as a quick instruction manual to get a DGX Spark (or similar Linux host) to a working state where **both**:
- **CDI mode** works: `docker run --device nvidia.com/gpu=...`
- **Legacy Docker GPU mode** works: `docker run --gpus ...`

---

## A) Baseline checks (Docker + GPU)

### 1) Confirm Docker version
```bash
dockerd --version
docker version
```

### 2) Confirm Docker is CDI-aware
```bash
docker info
```
Verify you see a section like:
- `CDI spec directories:`
  - `/etc/cdi`
  - `/var/run/cdi`

### 3) Confirm the legacy GPU path works (`--gpus`)
```bash
docker run --rm --gpus all \
  nvcr.io/nvidia/cuda:13.1.0-devel-ubuntu24.04 \
  nvidia-smi
```

---

## B) Diagnose CDI (when it fails)

### 4) Test CDI mode
```bash
docker run --rm --device nvidia.com/gpu=all \
  nvcr.io/nvidia/cuda:13.1.0-devel-ubuntu24.04 \
  nvidia-smi
```

If you get an error like:
- `CDI device injection failed: unresolvable CDI devices nvidia.com/gpu=all`

…it usually means **Docker supports CDI**, but **no NVIDIA CDI spec exists yet**.

### 5) Check whether NVIDIA CDI devices exist
```bash
nvidia-ctk cdi list
```
- If it says `Found 0 CDI devices`, generate the CDI spec (next section).

---

## C) Enable NVIDIA CDI (the fix)

### 6) Generate NVIDIA CDI spec (persistent)
This writes a CDI spec to `/etc/cdi` (persists across reboots).
```bash
sudo mkdir -p /etc/cdi
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

Optional (debug output if needed):
```bash
sudo nvidia-ctk --debug cdi generate --output=/etc/cdi/nvidia.yaml
```

### 7) Restart Docker
```bash
sudo systemctl restart docker
```

---

## D) Verify CDI is working

### 8) Confirm NVIDIA CDI devices are available
```bash
nvidia-ctk cdi list
```
Expected examples:
- `nvidia.com/gpu=0`
- `nvidia.com/gpu=GPU-<uuid>`
- `nvidia.com/gpu=all`

### 9) Run a CDI container test
```bash
docker run --rm --device nvidia.com/gpu=all \
  nvcr.io/nvidia/cuda:13.1.0-devel-ubuntu24.04 \
  nvidia-smi
```

---

## E) Confirm both modes work

### 10) Legacy mode (`--gpus`) still works
```bash
docker run --rm --gpus all \
  nvcr.io/nvidia/cuda:13.1.0-devel-ubuntu24.04 \
  nvidia-smi
```

---

## F) Useful checks for troubleshooting

### Check where CDI specs are coming from
```bash
sudo find /etc/cdi /var/run/cdi -maxdepth 1 -type f -print 2>/dev/null
sudo grep -R "nvidia.com/gpu" /etc/cdi /var/run/cdi 2>/dev/null
```

### Post-reboot quick sanity check
```bash
nvidia-ctk cdi list

docker run --rm --device nvidia.com/gpu=all \
  nvcr.io/nvidia/cuda:13.1.0-devel-ubuntu24.04 \
  nvidia-smi

docker run --rm --gpus all \
  nvcr.io/nvidia/cuda:13.1.0-devel-ubuntu24.04 \
  nvidia-smi
```

---

## Notes / operational guidance
- Avoid mixing mechanisms in a single run (don’t use `--gpus` and CDI `--device nvidia.com/gpu=...` together).
- Prefer `/etc/cdi` for CDI specs so they persist across reboots; `/var/run/cdi` is runtime-generated and can be cleared on reboot.


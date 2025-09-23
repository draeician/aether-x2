# Vulkan-only llama.cpp on CachyOS (fresh install) using an Arch-based Docker image

This guide treats the machine as a **fresh CachyOS** box and sets up a **Vulkan-only** llama.cpp server (no ROCm) in Docker. We’ll:

* Install Vulkan userspace on the host
* Install & enable Docker
* Build an **Arch Linux** container with RADV + llama.cpp (Vulkan backend)
* Mount host model/log directories: `/srv/aether/models` and `/srv/aether/logs`
* Run and verify the server

> **Workflow style:** After each step, run the provided **verify** command(s). **Do not proceed** until the step succeeds.

---

## 0) Host prerequisites

* You have sudo access.
* You’re on CachyOS (Arch-based). Terminal is fine; no GUI required.

---

## 1) Update system & install host Vulkan tools

```bash
sudo pacman -Syu --noconfirm
sudo pacman -S --needed --noconfirm \
  vulkan-radeon vulkan-tools vulkan-headers \
  shaderc glslang
```

**Verify Vulkan is present (headless):**

```bash
# Optional: avoid Wayland noise in headless shells
export XDG_RUNTIME_DIR=/tmp
export AMD_VULKAN_ICD=RADV
vulkaninfo | head -n 40
```

You should **not** see fatal errors. (It’s OK if it prints that `DISPLAY` is not set.)

---

## 2) Install Docker & docker-compose

```bash
sudo pacman -S --needed --noconfirm docker docker-compose
```

**Enable & start Docker:**

```bash
sudo systemctl enable --now docker
```

**Add your user to needed groups (new login required for groups to apply to your shell):**

```bash
sudo usermod -aG docker,video,render "$USER"
# open a NEW terminal (or log out/in) so group membership applies
```

**Verify:**

```bash
id   # ensure you now see docker, video, render in your groups
docker info | grep -E 'Cgroup|Driver|Runtimes'   # should list Cgroup Driver: systemd
ls -l /dev/dri/renderD*  # should exist and be group=render with read/write
```

> If `/dev/dri/renderD*` is missing or not group `render`, ensure AMDGPU is active and consider a udev rule, e.g.:
>
> ```bash
> sudo tee /etc/udev/rules.d/70-amd-vulkan.rules >/dev/null <<'RULES'
> KERNEL=="renderD*", SUBSYSTEM=="drm", MODE="0660", GROUP="render"
> RULES
> sudo udevadm control --reload && sudo udevadm trigger
> ```

---

## 3) Create host directories for models & logs

```bash
sudo mkdir -p /srv/aether/{models,logs}
sudo chown -R "$USER":"$(id -gn)" /srv/aether
```

**Verify:**

```bash
ls -ld /srv/aether /srv/aether/models /srv/aether/logs
```

Place your `.gguf` models into `/srv/aether/models`.

---

## 4) Create the **Arch-based** Vulkan-only Docker image

Create **`Dockerfile.vk-arch`** in an empty working dir with:

```dockerfile
FROM archlinux:base

# Speed up pacman and make it non-interactive
RUN pacman -Syu --noconfirm && \
    pacman -S --noconfirm \
      git cmake ninja base-devel \
      vulkan-headers vulkan-tools vulkan-icd-loader \
      vulkan-radeon glslang shaderc && \
    pacman -Scc --noconfirm

# Build llama.cpp (Vulkan backend only)
RUN git clone --depth=1 https://github.com/ggerganov/llama.cpp.git /opt/llama.cpp && \
    cd /opt/llama.cpp && cmake -B build-vk -G Ninja -DGGML_VULKAN=ON && \
    cmake --build build-vk -j

# Headless defaults to avoid X/Wayland hangs and stray layers
ENV AMD_VULKAN_ICD=RADV \
    XDG_RUNTIME_DIR=/tmp \
    VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json \
    VK_INSTANCE_LAYERS= \
    VK_LOADER_DISABLE_INSTALLED_EXTENSIONS=1

WORKDIR /opt/llama.cpp
CMD ["bash","-lc","/opt/llama.cpp/build-vk/bin/llama-server --help"]
```

**Build the image:**

```bash
docker build -t llama-vk:arch -f Dockerfile.vk-arch .
```

This compiles llama.cpp (Vulkan backend) inside the image.

**Verify Vulkan visibility inside the image (smoke test):**

```bash
docker run --rm -it \
  --device=/dev/dri \
  --group-add "$(getent group render | cut -d: -f3)" \
  --group-add "$(getent group video  | cut -d: -f3)" \
  llama-vk:arch \
  bash -lc 'ls -l /usr/share/vulkan/icd.d/*radeon* ; env -u DISPLAY -u WAYLAND_DISPLAY vulkaninfo --summary | head -n 60'
```

Expected: a RADV device appears in the summary.

> **Why numeric GIDs?** Docker supplements groups by **GID**. Using `--group-add <gid>` ensures permissions on `/dev/dri` work even if group names don’t exist in the image.

---

## 5) One-shot run of the server (baseline)

Run with conservative settings first:

```bash
docker run --rm -it \
  -p 8999:8999 \
  --device=/dev/dri \
  --group-add $(getent group render | cut -d: -f3) \
  --group-add $(getent group video | cut -d: -f3) \
  -e AMD_VULKAN_ICD=RADV \
  -e XDG_RUNTIME_DIR=/tmp \
  -e VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/radeon_icd.x86_64.json \
  -e VK_INSTANCE_LAYERS= \
  -e VK_LOADER_DISABLE_INSTALLED_EXTENSIONS=1 \
  -v /srv/aether/models:/models:ro \
  -v /srv/aether/logs:/logs \
  llama-vk:arch \
  bash -lc '/opt/llama.cpp/build-vk/bin/llama-server \
    --model /models/<YOUR_MODEL>.gguf \
    --host 0.0.0.0 --port 8999 \
    -c 3072 -t 8 -ngl 12 --parallel 1 \
    --timeout 600 --flash-attn off \
    --log-format json --log-file /logs/server.log'
```

Replace `<YOUR_MODEL>.gguf` with the actual filename in `/srv/aether/models`.

**Verify from host:**

```bash
curl -sS http://127.0.0.1:8999/health
curl -sS -X POST http://127.0.0.1:8999/completion \
  -H 'Content-Type: application/json' \
  -d '{"prompt":"Say hello in 3 words.","n_predict":16}'
```

Both should return clean JSON.

---

## 6) Docker Compose (easy start/stop)

Create **`docker-compose.vk.yml`** next to your Dockerfile:

```yaml
services:
  llama:
    image: llama-vk:arch
    container_name: llama-vk
    restart: unless-stopped
    ports:
      - "8999:8999"
    devices:
      - "/dev/dri:/dev/dri"
    group_add:
      - "${RENDER_GID}"
      - "${VIDEO_GID}"
    environment:
      AMD_VULKAN_ICD: "RADV"
      XDG_RUNTIME_DIR: "/tmp"
      VK_ICD_FILENAMES: "/usr/share/vulkan/icd.d/radeon_icd.x86_64.json"
      VK_INSTANCE_LAYERS: ""
      VK_LOADER_DISABLE_INSTALLED_EXTENSIONS: "1"
    command: >
      bash -lc '/opt/llama.cpp/build-vk/bin/llama-server
      --model /models/${MODEL}
      --host 0.0.0.0 --port 8999
      -c ${CTX:-3072} -t ${THREADS:-8} -ngl ${NGL:-12} --parallel ${PARALLEL:-1}
      --timeout 600 --flash-attn off
      --log-format json --log-file /logs/server.log'
    volumes:
      - "/srv/aether/models:/models:ro"
      - "/srv/aether/logs:/logs"
```

Create an **`.env`** file in the same directory to fill in variables:

```bash
RENDER_GID=$(getent group render | cut -d: -f3)
VIDEO_GID=$(getent group video | cut -d: -f3)
MODEL=<YOUR_MODEL>.gguf
CTX=3072
THREADS=8
NGL=12
PARALLEL=1
```

**Bring it up & watch logs:**

```bash
docker compose -f docker-compose.vk.yml up -d
docker compose -f docker-compose.vk.yml logs -f
```

**Verify from host:**

```bash
curl -sS http://127.0.0.1:8999/health
```

**Stop it:**

```bash
docker compose -f docker-compose.vk.yml down
```

---

## 7) Tuning, one knob at a time

After baseline is stable:

* Increase GPU work first: `NGL=16`
* Then context: `CTX=4096`
* Then concurrency: `PARALLEL=2`

Apply via env to Compose:

```bash
NGL=16 docker compose -f docker-compose.vk.yml up -d --force-recreate
```

If unstable, revert to last stable values.

---

## 8) Troubleshooting

* **Container can’t see the GPU**: ensure `/dev/dri/renderD*` exists on host and you pass it (`devices:`). Confirm `group_add` uses **GIDs**: `getent group render`, `getent group video`.
* **Vulkan hangs or layer errors**: we export `VK_INSTANCE_LAYERS=` and `VK_LOADER_DISABLE_INSTALLED_EXTENSIONS=1` to disable stray layers (e.g., anti-lag). Keep `AMD_VULKAN_ICD=RADV` and `XDG_RUNTIME_DIR=/tmp`.
* **Model not found**: verify the exact filename in `/srv/aether/models` and the Compose `MODEL` variable.
* **Performance regressed after a tweak**: revert the last change; change **one knob at a time**.

---

## 9) Notes

* We intentionally do **not** install ROCm and do **not** map `/dev/kfd`.
* The Arch image keeps Mesa/RADV current (good for brand‑new AMD parts).
* Logs are written to `/srv/aether/logs/server.log` inside the container via the `--log-file` flag.

---

**You’re done.** Models live under `/srv/aether/models`. Logs live under `/srv/aether/logs`. Start the server with Docker or Compose and scale cautiously.

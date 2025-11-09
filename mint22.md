```markdown
# Mint 22 — Installation & Setup Notes

This file collects my Mint 22 setup steps and useful snippets (GPU drivers, Ollama, podman, power settings, and static network configuration). Commands are intended to be run as root (sudo) where shown.

---

## Table of contents

- Prerequisites
- Kernel headers & extra modules
- AMD GPU / ROCm quick steps
- Install Ollama & systemd service
- podman (verify ROCm access)
- Prevent the machine from sleeping
- Static wired/wireless network (nmcli)
- Notes & helpful commands

---

## Prerequisites

Update packages and make sure common tools are installed:

```bash
sudo apt update
sudo apt install -y curl wget software-properties-common
```

Make sure the user is in the video and render groups (needed for GPU access):

```bash
sudo usermod -a -G video,render "$LOGNAME"
```

---

## Kernel headers & extra modules

Install matching kernel headers and extra kernel modules (for drivers like amdgpu/ROCm):

```bash
sudo apt install -y "linux-headers-$(uname -r)" "linux-modules-extra-$(uname -r)"
```

---

## AMD GPU / ROCm

Example commands I've used for AMD GPU / ROCm related packages.

1. Download and install insecure udev rules package (example URL):

```bash
wget https://repo.radeon.com/amdgpu/30.20/ubuntu/pool/main/a/amdgpu-insecure-instinct-udev-rules/amdgpu-insecure-instinct-udev-rules_30.20.0.0-2238411.24.04_all.deb
sudo apt install ./amdgpu-insecure-instinct-udev-rules_30.20.0.0-2238411.24.04_all.deb
```

2. (Optional) Download amdgpu-install package:

```bash
wget https://repo.radeon.com/amdgpu-install/7.1/ubuntu/noble/amdgpu-install_7.1.70100-1_all.deb
sudo apt install ./amdgpu-install_7.1.70100-1_all.deb
```

3. Install Python packaging tools (needed for some installer scripts):

```bash
sudo apt install -y python3-setuptools python3-wheel
```

4. (Optional) Install ROCm helper tools and probe devices:

```bash
sudo apt install -y environment-modules
sudo apt install -y rocm-smi rocminfo
rocminfo
```

---

## Ollama (install & systemd service)

Install Ollama (installer from ollama.com) and pull a model:

```bash
sudo curl -fsSL https://ollama.com/install.sh | sh
ollama pull gpt-oss:20b
```

Create a systemd service to run Ollama as a service. Below are two files you can add to /etc/systemd.

1) Unit: /etc/systemd/system/ollama.service

```ini
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/local/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin"

[Install]
WantedBy=default.target
```

2) Drop-in config: /etc/systemd/system/ollama.service.d/ollama.conf

```ini
# Optional environment overrides for Ollama. Example:
# OLLAMA_MODELS=/media/elysium/models/ollama_models
# OLLAMA_HOST=0.0.0.0:11434
# OLLAMA_ORIGINS=*
# OLLAMA_MAX_LOADED_MODEL=3
# OLLAMA_NUM_PARALLEL=6

[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_ORIGINS=*"
Environment="OLLAMA_MAX_LOADED_MODEL=2"
Environment="OLLAMA_NUM_PARALLEL=4"
```

Notes:
- Create the `ollama` system user/group if it does not exist:
  ```bash
  sudo useradd -r -s /usr/sbin/nologin -M ollama || true
  ```
- After adding files, reload systemd and enable the service:

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable --now ollama
  sudo systemctl status ollama
  ```

---

## podman (verify ROCm GPU access in containers)

Install podman and run a ROCm container to verify device access:

```bash
sudo apt install -y podman
podman --version

podman run -it --rm \
  --device=/dev/kfd \
  --device=/dev/dri \
  --group-add keep-groups \
  docker.io/rocm/rocm-terminal \
  rocminfo
```

If `rocminfo` inside the container lists devices, the container can access the ROCm devices.

---

## Prevent the machine from sleeping

Edit /etc/systemd/logind.conf (use your editor) and set the handlers you want to ignore:

```ini
# /etc/systemd/logind.conf
HandleSuspendKey=ignore
HandleSleepKey=ignore
HandleLidSwitch=ignore
```

After editing:

```bash
sudo systemctl restart systemd-logind
```

Note: On laptops, preventing lid-switch suspend may have implications for power usage and safety. Use with care.

---

## Static wired and wireless network (NetworkManager / nmcli)

Examples for configuring a static IP on NetworkManager connections.

Replace the connection names ("Wired connection 1", "falgout-g") with the actual connection names on your machine.

Wired connection example:

```bash
# Set static IP and subnet
sudo nmcli connection modify "Wired connection 1" ipv4.addresses 192.168.22.53/24

# Set the gateway
sudo nmcli connection modify "Wired connection 1" ipv4.gateway 192.168.22.1

# Set the DNS servers
sudo nmcli connection modify "Wired connection 1" ipv4.dns "192.168.22.1"

# Set DNS search domain
sudo nmcli connection modify "Wired connection 1" ipv4.dns-search "draeician.com"

# Use manual (static) method
sudo nmcli connection modify "Wired connection 1" ipv4.method manual
```

Wireless connection example (replace "falgout-g"):

```bash
sudo nmcli connection modify "falgout-g" ipv4.addresses 192.168.22.53/24
sudo nmcli connection modify "falgout-g" ipv4.gateway 192.168.22.1
sudo nmcli connection modify "falgout-g" ipv4.dns "192.168.22.1"
sudo nmcli connection modify "falgout-g" ipv4.dns-search "draeician.com"
sudo nmcli connection modify "falgout-g" ipv4.method manual
```

Apply changes by restarting NetworkManager:

```bash
sudo systemctl restart NetworkManager
```

Tip: To list connections and find their exact names:

```bash
nmcli connection show
```

---

## Useful commands & notes

- Reload systemd after unit changes:
  ```bash
  sudo systemctl daemon-reload
  ```
- Check a unit's logs:
  ```bash
  sudo journalctl -u ollama -f
  ```
- Verify kernel modules and GPU devices:
  ```bash
  lsmod | grep amdgpu
  rocminfo
  ```

---

If you'd like, I can push this reformatted file back to the repo as a commit, or make additional edits (add more explanations, split into separate files, or include scripts). Which would you prefer?
```

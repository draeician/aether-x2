# installation process for mint 22

sudo apt install -y "linux-headers-$(uname -r)" "linux-modules-extra-$(uname -r)"
sudo usermod -a -G video,render $LOGNAME
sudo apt update
wget https://repo.radeon.com/amdgpu/30.20/ubuntu/pool/main/a/amdgpu-insecure-instinct-udev-rules/amdgpu-insecure-instinct-udev-rules_30.20.0.0-2238411.24.04_all.deb
sudo apt install ./amdgpu-insecure-instinct-udev-rules_30.20.0.0-2238411.24.04_all.deb
sudo  curl -fsSL https://ollama.com/install.sh | sh
ollama pull gpt-oss:20b



wget https://repo.radeon.com/amdgpu-install/7.1/ubuntu/noble/amdgpu-install_7.1.70100-1_all.deb
sudo apt install python3-setuptools python3-wheel
   84  sudo apt install python3-setuptools python3-wheel
   85  sudo apt install environment-modules
   87  apt install rocm-smi rocminfo


```
#!/usr/bin/env bash
mkdir -p "$(dirname -- '/etc/systemd/system/ollama.service')"
cat > '/etc/systemd/system/ollama.service' << 'EOF_CB_0D2B6FB8'
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
EOF_CB_0D2B6FB8

mkdir -p "$(dirname -- '/etc/systemd/system/ollama.service.d/ollama.conf')"
cat > '/etc/systemd/system/ollama.service.d/ollama.conf' << 'EOF_CB_6633DE28'
# Prep: 
# mkdir -p /etc/systemd/system/ollama.service.d
# [Service]
# Conditionally include EnvironmentFile directive
# EnvironmentFile=/etc/default/ollama
#
# systemctl daemon-reload
# systemctl restart ollama
# 
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

# If you want to add your models path, uncomment the line below
# Environment="OLLAMA_MODELS=/media/elysium/models/ollama_models"
EOF_CB_6633DE28
```

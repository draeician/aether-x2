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

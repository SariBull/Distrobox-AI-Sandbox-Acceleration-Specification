# Distrobox AI Sandbox Acceleration Specification

> Zero-Pollution AMD Hardware Acceleration Passthrough for Ubuntu 24.04 LTS Containers.

This specification isolates the local LLM runtime (Ollama) inside an OCI container while passing through bare-metal AMD GPU acceleration (`/dev/kfd`, `/dev/dri`), preventing host package pollution and Python dependency collisions.

---

## Hardware & Environment Mapping

| Pipeline Target | Host OS | Default Shell | Hardware Architecture | Target Device | Overridden GFX |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Pipeline 1** | CachyOS | Fish | AMD RDNA 3.5 (APU) | Ryzen AI Max+ 395 (Radeon 8060S) | `11.0.0` (`iGPU=1`) |
| **Pipeline 2** | Fedora 44 | Bash | AMD RDNA 4 (dGPU) | Ryzen 7 9800X3D + Radeon RX 9070 XT | `12.0.1` |

Strictly execute **only one** pipeline matching your target platform.

---

## 1. CachyOS (Fish Shell) Target: AMD Strix Halo (RDNA 3.5)

Optimized for mobile edge nodes (e.g., ROG Flow Z13) equipped with AMD Ryzen AI Max+ 395. Addresses Fish shell Here-Doc syntax incompatibilities via `printf` pipelines and activates the unified memory iGPU flag.

### Step 1. Core Engine Injection & Sandbox One-Shot Build
```fish
# 1. Inject core engines via package manager
sudo pacman -S --needed podman distrobox --noconfirm

# 2. Allocate an isolated home directory for the AI runtime sandbox
mkdir -p ~/distrobox_homes/ai_core

# 3. Build base image with GPU hardware passthrough
distrobox create --image docker.io/library/ubuntu:24.04 \
  --name ai-core \
  --home ~/distrobox_homes/ai_core \
  --additional-flags "--device /dev/kfd --device /dev/dri" -Y
```
### Step 2. Container Internal Runtime Lock-in (Ubuntu)
```fish
# Enter the sandbox
distrobox enter ai-core
(Execute inside the container)
# Sync packages and inject essential tools
sudo apt update && sudo apt install curl fish -y

# Change the default shell to Fish and switch session
chsh -s /usr/bin/fish
fish

# Lock-in hardware acceleration variables targeting RDNA 3.5 (Radeon 8060S)
set -Ux HSA_OVERRIDE_GFX_VERSION 11.0.0
set -Ux OLLAMA_IGPU_ENABLE 1

# Download and install Ollama binary toolchain
curl -fsSL https://ollama.com/install.sh | sh

# Exit session and return to CachyOS host
exit
exit
```
### Step 3. GNOME Autostart Background Daemonization
After returning to the host (CachyOS), execute the script below to lock in the AI daemon to automatically ignite upon GNOME login.
```fish
# Allocate global execution binary and autostart directories
mkdir -p ~/.local/bin ~/.config/autostart

# Write manual/auto trigger script (printf single pipeline processing)
printf '#!/bin/bash\n/usr/bin/distrobox enter ai-core -- true\npodman exec -d ai-core env HSA_OVERRIDE_GFX_VERSION=11.0.0 OLLAMA_IGPU_ENABLE=1 ollama serve\necho "[System] AI Core Daemon successfully injected and detached."\n' > ~/.local/bin/wake-ai
chmod +x ~/.local/bin/wake-ai

# Write desktop autostart entry 
printf '[Desktop Entry]\nType=Application\nName=Ollama Distrobox Server\nComment=Start Ollama with AMD ROCm in background via Podman Detach\nExec=/bin/bash -c "$HOME/.local/bin/wake-ai"\nTerminal=false\nStartupNotify=false\n' > ~/.config/autostart/ollama-ai.desktop
chmod +x ~/.config/autostart/ollama-ai.desktop
```
## 2. Fedora Workstation (Bash Shell) Target: 9800X3D + RX 9070 XT (RDNA 4)

Optimized for high-end desktop workstations running Fedora 44. Standardizes on Bash and Systemd User Services to ensure seamless integration and avoid Python virtual environment (venv) breakage.

### Step 1. Core Engine Injection & Sandbox One-Shot Build
```bash
#Inject core engines
sudo dnf install podman distrobox -y

# Allocate an isolated home directory
mkdir -p ~/distrobox_homes/ai_core

# Build base image with GPU hardware passthrough
distrobox create --image docker.io/library/ubuntu:24.04 \
  --name ai-core \
  --home ~/distrobox_homes/ai_core \
  --additional-flags "--device /dev/kfd --device /dev/dri" -Y
```
### Step 2. Container Internal Runtime Lock-in (Ubuntu)
```bash
# Enter the sandbox
distrobox enter ai-core
```
(Execute inside the container)
```bash
# Sync packages and inject essential tools
sudo apt update && sudo apt install curl wget git python3-venv python3-pip zstd -y

# Lock-in hardware acceleration variables targeting RDNA 4 (RX 9070 XT)
echo 'export HSA_OVERRIDE_GFX_VERSION=12.0.1' >> ~/.bashrc
source ~/.bashrc

# Download and install Ollama engine
curl -fsSL https://ollama.com/install.sh | sh

# Return to host OS
exit
```
### Step 3. Systemd Linger Background Daemonization
After returning to the host (Fedora), compile a script that strikes the daemon directly with the OCI API and lock it as a Systemd unit.
```bash
# Compile global wrapper script
mkdir -p ~/.local/bin
cat <<'EOF' > ~/.local/bin/wake-ai
#!/bin/bash
/usr/bin/podman restart ai-core
sleep 3
/usr/bin/podman exec -d ai-core env HSA_OVERRIDE_GFX_VERSION=12.0.1 HOME=/home/$USER /usr/local/bin/ollama serve
echo "[System] AI Core Daemon fully operational."
EOF
chmod +x ~/.local/bin/wake-ai

# Write Systemd user service unit file
mkdir -p ~/.config/systemd/user
cat <<'EOF' > ~/.config/systemd/user/ai-core.service
[Unit]
Description=Distrobox AI Daemon Autostart
After=network-online.target

[Service]
Type=oneshot
ExecStart=%h/.local/bin/wake-ai
RemainAfterExit=yes

[Install]
WantedBy=default.target
EOF

# Reload daemon and lock-in background startup with Linger permission
systemctl --user daemon-reload
systemctl --user enable --now ai-core.service
loginctl enable-linger $USER
```
## 3. Dead Layer & Weights Incineration Protocol (Maintenance)
Maintenance commands to completely destroy all containers and caches to 0 bytes in the event of a runtime environment collapse. (Downloaded LLM model weights will also be bulk deleted).
```bash
# Force remove container instances
distrobox rm -f ai-core

# Permanently delete the container's isolated home directory
rm -rf ~/distrobox_homes

# Completely delete stopped container fragments and unused image caches
podman system prune -a -f
```

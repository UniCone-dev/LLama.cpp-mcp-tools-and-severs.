# Local-LLM: Fully Private, Self-Hosted AI Infrastructure

A completely open-source, offline, and private AI environment running locally on your machine. Powered by llama.cpp with support for high-performance backends (Vulkan, CUDA, ROCm, CPU), native Jinja template parsing, and local MCP (Model Context Protocol) tools for function calling and web search.

# Directory Structure Overview

When you extract `Local-LLM.tar.z` or `Local-LLM.zip`, your folder structure will look like this:

```text
Local-LLM/
├── llama.cpp/             # Built by the user (or AI assistant)
├── mcp-tools/             # Contains server.py & python venv
├── mcp-websearch/         # Contains server.py & python venv
├── memory/                # Local persistent memory/context storage
├── models/                # Place your .gguf models here
├── start-qwen.sh          # Core inference script
├── start-tools-mcp.sh     # MCP local tools execution script
└── start-websearch-mcp.sh # MCP web search execution script
```

# Quick Installation Guide

Download `Local-LLM.zip` (or `Local-LLM.tar.z`) and extract it.

**Recommended (Optional):** Move the extracted folder directly to your `$HOME` directory (`~/Local-LLM`). This makes terminal paths and global alias shortcuts seamless.

```bash
# Example extraction to Home directory
tar -I zstd -xvf Local-LLM.tar.z -C ~/

# OR for zip:
unzip Local-LLM.zip -d ~/
```

# Step 2: Install System Dependencies

Select your distribution to install build tools, Python environment, and driver libraries.

## 🔹 Arch Linux

```bash
sudo pacman -S --needed base-devel cmake git python python-pip

# For GPU Acceleration (Choose your card):
# AMD: vulkan-radeon | NVIDIA: nvidia-utils | Intel: vulkan-intel
```

## 🔹 Ubuntu / Debian

```bash
sudo apt update
sudo apt install build-essential cmake git python3 python3-venv python3-pip libvulkan-dev
```

## 🔹 Fedora

```bash
sudo dnf groupinstall "Development Tools"
sudo dnf install cmake git python3 python3-pip vulkan-loader-devel
```

# Step 3: Build llama.cpp (Choose Your Backend)

Navigate to the llama.cpp directory and build according to your hardware setup.

```bash
cd ~/Local-LLM
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp
```

## Option A: Vulkan

**Recommended for cross-platform AMD/Intel/NVIDIA GPU acceleration.**

```bash
cmake -B build -DGGML_VULKAN=ON
cmake --build build --config Release -j$(nproc)
```

## Option B: CUDA

**For NVIDIA GPUs.**

```bash
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j$(nproc)
```

## Option C: ROCm

**For AMD GPUs on Linux.**

```bash
hipcc --version
cmake -B build -DGGML_HIPBLAS=ON
cmake --build build --config Release -j$(nproc)
```

## Option D: CPU Only

**Universal, no GPU needed.**

```bash
cmake -B build
cmake --build build --config Release -j$(nproc)
```

After building, return to the root folder:

```bash
cd ~/Local-LLM
```

# Step 4: Model Setup

Place your downloaded `.gguf` file inside the `models/` directory.

**Note:** Qwen 3.5 / Qwen series models are recommended by default due to superior benchmark performance and tool-calling capabilities, but any GGUF model will work.

```bash
mv /path/to/your-model.gguf ~/Local-LLM/models/
```

# Step 5: Setup Python Virtual Environments (venv) for MCP Tools

Both `mcp-tools` and `mcp-websearch` run inside isolated Python virtual environments.

## Setup mcp-tools

```bash
cd ~/Local-LLM/mcp-tools
python3 -m venv venv
source venv/bin/activate
pip install mcp
deactivate
```

## Setup mcp-websearch

```bash
cd ~/Local-LLM/mcp-websearch
python3 -m venv venv
source venv/bin/activate
pip install mcp requests htbuilder
deactivate

cd ~/Local-LLM
```

# Updated Control Scripts

Make sure all scripts have execution permissions:

```bash
chmod +x ~/Local-LLM/*.sh
```

## start-qwen.sh

This script auto-detects your root directory and automatically selects the `.gguf` model present in `models/`.

```bash
#!/usr/bin/env bash

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Auto-detect .gguf model from models directory
MODEL_FILE=$(find "$SCRIPT_DIR/models" -maxdepth 2 -name "*.gguf" | head -n 1)

if [ -z "$MODEL_FILE" ]; then
    echo "Error: No .gguf model found in $SCRIPT_DIR/models/"
    exit 1
fi

cd "$SCRIPT_DIR/llama.cpp" || exit 1

echo "Model is Starting Pleses Wait ....."

( sleep 3 && xdg-open "http://127.0.0.1:8080" >/dev/null 2>&1 & )

./build/bin/llama-server \
    -m "$MODEL_FILE" \
    -ngl 99 \
    -c 16384 \
    --parallel 1 \
    --cache-type-k q4_0 --cache-type-v q4_0 \
    --host 127.0.0.1 --port 8080 \
    --tools all --jinja
```

## start-tools-mcp.sh

```bash
#!/usr/bin/env bash

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

source "$SCRIPT_DIR/mcp-tools/venv/bin/activate"
cd "$SCRIPT_DIR/mcp-tools"
python server.py
```

## start-websearch-mcp.sh

```bash
#!/usr/bin/env bash

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

source "$SCRIPT_DIR/mcp-websearch/venv/bin/activate"
cd "$SCRIPT_DIR/mcp-websearch"
python server.py
```

# Terminal Shortcuts (Aliases)

To run the servers from anywhere in your terminal without typing full paths, add aliases to your shell configuration file (`~/.bashrc`, `~/.zshrc`, or `~/.config/fish/config.fish`).

```bash
# For Bash / Zsh (Arch, Ubuntu, Fedora, Debian)
echo "alias qwen='~/Local-LLM/start-qwen.sh'" >> ~/.bashrc
echo "alias tools-mcp='~/Local-LLM/start-tools-mcp.sh'" >> ~/.bashrc
echo "alias search-mcp='~/Local-LLM/start-websearch-mcp.sh'" >> ~/.bashrc

source ~/.bashrc
```

Now, simply type in your terminal:

* `qwen` → Starts the LLM Inference Server on http://127.0.0.1:8080
* `tools-mcp` → Starts the Local MCP Server (Endpoint: http://127.0.0.1:8000/mcp)

# AI Assistant Auto-Installation Prompt

If you prefer an AI Agent (like Claude Code, Cursor, ChatGPT Desktop, or a terminal agent) to set up everything automatically for you, simply copy-paste this prompt:

**Prompt for AI Assistant:**

> Please install and set up my local LLM environment inside `~/Local-LLM`. Follow these steps:
>
> 1. Clone llama.cpp into `~/Local-LLM/llama.cpp` and build it with Vulkan support (or CUDA if NVIDIA GPU is present).
> 2. Check `~/Local-LLM/mcp-tools` and `~/Local-LLM/mcp-websearch`, create python venv in both folders, activate them, and install required dependencies (`mcp`, `requests`, `htbuilder`).
> 3. Ensure scripts (`start-qwen.sh`, `start-tools-mcp.sh`, `start-websearch-mcp.sh`) are executable (`chmod +x`).
> 4. Add aliases `qwen` and `tools-mcp` to my `~/.bashrc` or `~/.zshrc`.

# Troubleshooting & Verification

**MCP Connection Errors:** Verify that the Python venv activated correctly and dependencies are installed. Ensure the MCP server URL in your client points to http://127.0.0.1:8000/mcp (or the port defined in your `server.py`).

**Persistent Memory:** All conversation logs, vector DB indexes, or tool memories are safely stored locally under the `memory/` directory.

**Privacy & Security:** All scripts execute strictly on `127.0.0.1`. No data leaves your machine.

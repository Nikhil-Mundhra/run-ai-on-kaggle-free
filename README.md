# Run AI on Kaggle (Free Dual T4 GPUs)

Turn a free Kaggle notebook into a high-performance backend for your local VS Code setup. This script automates the installation of **Ollama**, configures **Dual Tesla T4 GPUs**, and exposes the API via a **Cloudflare Tunnel**.

[![Kaggle](https://img.shields.io/badge/Kaggle-20BEFF?style=for-the-badge&logo=Kaggle&logoColor=white)](https://www.kaggle.com/)
[![Ollama](https://img.shields.io/badge/Ollama-000000?style=for-the-badge&logo=ollama&logoColor=white)](https://ollama.com/)

## ✨ Features
* **Zero Cost:** Uses Kaggle's free tier (30 hours of GPU/week).
* **High Performance:** Full GPU acceleration on 2x NVIDIA Tesla T4 (32GB combined VRAM).
* **Seamless Integration:** Designed specifically for the **Continue** extension in VS Code.
* **Auto-Config:** Automatically generates the JSON block for your VS Code settings.

## 🛠️ Prerequisites
1. **Kaggle Account:** [Sign up here](https://www.kaggle.com/).
2. **Phone Verification:** Required by Kaggle to enable GPU access.
3. **GPU Enabled:** In your Kaggle notebook sidebar, set **Accelerator** to `GPU T4 x2`.

## 🚀 Quick Start

1. Create a new Kaggle Notebook.
2. Ensure **Internet on** is toggled in the settings.
3. Paste the following script into a cell and run it:

```python
import subprocess
import os
import time
import re

print("🚀 Initializing Total Kaggle GPU AI Setup...\n")

def run_cmd(cmd, desc):
    print(f"[*] {desc}...")
    # Using shell=True and letting stdout flow normally so you can see the model download progress
    subprocess.run(cmd, shell=True)

# 1. Clean up any zombie processes from previous runs
run_cmd("pkill -9 ollama", "Cleaning up old Ollama processes")
run_cmd("pkill -9 cloudflared", "Cleaning up old Cloudflare tunnels")

# 2. Install dependencies (zstd, GPU detection tools, and Cloudflared)
run_cmd("apt-get update && apt-get install -y zstd pciutils lshw", "Installing zstd and GPU detection tools")
run_cmd("wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb && dpkg -i cloudflared-linux-amd64.deb", "Installing Cloudflared tunnel")

# 3. Install Ollama
run_cmd("curl -fsSL https://ollama.com/install.sh | sh", "Installing Ollama")

# 4. Configure the Environment to force Kaggle's T4 GPU detection
env = os.environ.copy()
env["OLLAMA_ORIGINS"] = "*"
env["OLLAMA_HOST"] = "0.0.0.0"
env["LD_LIBRARY_PATH"] = "/usr/lib64-nvidia:/usr/local/cuda/lib64:/usr/lib/x86_64-linux-gnu"
env["CUDA_VISIBLE_DEVICES"] = "0,1" 

# 5. Start Ollama Server in the background
print("[*] Starting Ollama server with GPU support...")
ollama_log = open("ollama_serve.log", "w")
subprocess.Popen(["ollama", "serve"], env=env, stdout=ollama_log, stderr=ollama_log)
time.sleep(10) # Crucial wait time for it to scan and detect the T4s

# 6. Pull the Model
run_cmd("ollama pull qwen2.5-coder:7b", "Pulling qwen2.5-coder:7b model")

# 7. Start Cloudflare Tunnel in the background
print("[*] Starting Cloudflare tunnel...")
tunnel_log_path = "tunnel.log"
tunnel_log = open(tunnel_log_path, "w")
subprocess.Popen(["cloudflared", "tunnel", "--url", "http://127.0.0.1:11434"], stdout=tunnel_log, stderr=tunnel_log)
time.sleep(10) # Wait for Cloudflare to assign the public URL

# 8. Extract the URL from the logs
url = None
try:
    with open(tunnel_log_path, "r") as f:
        log_content = f.read()
        match = re.search(r"https://[a-zA-Z0-9-]+\.trycloudflare\.com", log_content)
        if match:
            url = match.group(0)
except Exception as e:
    print(f"[!] Error reading tunnel log: {e}")

# 9. Output Final Results
print("\n" + "="*60)
if url:
    print("✅ SETUP COMPLETE! YOUR API IS LIVE.")
    print("="*60)
    print("\nCopy and paste this EXACT block into your VS Code Continue config.json:\n")
    
    config_json = f"""    {{
      "title": "Qwen 2.5 Coder (Kaggle GPU)",
      "provider": "openai",
      "model": "qwen2.5-coder:7b",
      "apiBase": "{url}/v1",
      "apiKey": "none"
    }}"""
    print(config_json)
    print("\n" + "="*60)
    print("Verify the GPU connection by running this command in a new cell:")
    print("!ollama run qwen2.5-coder:7b \"Write a python hello world\" --verbose")
else:
    print("❌ Failed to generate Cloudflare URL. Check tunnel.log for details.")
    print("="*60)
```

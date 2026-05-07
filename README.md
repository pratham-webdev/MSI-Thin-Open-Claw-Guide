# MSI Thin AI Machine Setup Guide

A complete guide to turning your MSI Thin 15 into a self-hosted AI workstation. The agent stack runs in WSL2 (sandboxed from your personal Windows files), while LM Studio runs natively on Windows as your independent chat interface.

**Stack:** Windows 11 + WSL2 + Ollama + OpenClaw + LM Studio

**Total time:** 45-60 minutes

---

## Your MSI Thin 15 Specs That Matter

| Component | Spec |
|-----------|------|
| GPU | NVIDIA RTX 4050 6GB GDDR6 (CUDA) |
| CPU | Intel Core i5 12th gen |
| RAM | 16GB DDR5 |
| Storage | SSD (512GB or 1TB depending on variant) |
| OS | Windows 11 |

The RTX 4050's 6GB **dedicated VRAM** is the key advantage over shared-memory systems. Models load into VRAM independently of system RAM -- no competition between the GPU and Windows for memory.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                     Windows 11                       │
│                                                      │
│  ┌─────────────────┐    ┌──────────────────────┐    │
│  │   LM Studio     │    │       WSL2            │    │
│  │  (Windows app)  │    │                      │    │
│  │                 │    │  ┌────────────────┐  │    │
│  │  Independent    │    │  │     Ollama     │  │    │
│  │  model store    │    │  │  localhost:    │  │    │
│  │  chat UI        │    │  │    11434       │  │    │
│  │                 │    │  └───────┬────────┘  │    │
│  └─────────────────┘    │          │           │    │
│                          │  ┌───────▼────────┐  │    │
│                          │  │   OpenClaw     │  │    │
│                          │  │  (sandboxed)   │  │    │
│                          │  │  localhost:    │  │    │
│                          │  │    18789       │  │    │
│                          │  └────────────────┘  │    │
│                          └──────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

**Why this split:**

- Ollama and OpenClaw in WSL2 share `localhost` -- no cross-network bridging needed
- OpenClaw is sandboxed -- it can only access the WSL2 filesystem by default, not your Windows personal files, OneDrive, or Documents
- LM Studio on Windows is completely independent -- your personal chat sandbox, unaffected by anything in WSL2
- NVIDIA CUDA passes through from Windows to WSL2 automatically -- no driver installation inside Linux

---

## Where Commands Run

| Code block type | Environment | How to open |
|----------------|-------------|-------------|
| `powershell` | Windows PowerShell (Admin) | Win+X → Terminal (Admin) |
| `bash` | WSL2 / Ubuntu terminal | Start menu → Ubuntu, or `wsl` in PowerShell |
| `ini` / `batch` | Windows config file | Notepad |

---

## Storage Footprint

| Component | Size | Location |
|-----------|------|----------|
| WSL2 + Ubuntu | ~3 GB | Windows C: drive (VHDX) |
| Ollama binary | ~200 MB | Inside WSL2 |
| Qwen3.5 4B (Q4_K_M) | ~2.5 GB | `~/.ollama/models` in WSL2 |
| Qwen2.5 Coder 7B (Q4_K_M) | ~4.2 GB | `~/.ollama/models` in WSL2 |
| nomic-embed-text | ~280 MB | `~/.ollama/models` in WSL2 |
| Node.js + OpenClaw | ~600 MB | Inside WSL2 |
| Working overhead | ~2 GB | Inside WSL2 |
| **WSL2 total** | **~13 GB** | |
| | | |
| LM Studio | ~500 MB | Windows (app) |
| LM Studio models | varies | `C:\Users\<you>\.lmstudio\models\` |
| **LM Studio total** | **500 MB + models** | Windows |

Plan for ~20 GB free before starting (WSL2 stack only, LM Studio models are separate).

- MSI Thin with Windows 11 fully updated
- **NVIDIA driver 550+ installed** -- check via Device Manager or run `nvidia-smi` in PowerShell. Download latest from https://www.nvidia.com/drivers if needed. Don't use Windows Update for this -- it lags behind.
- At least 20 GB free on the system drive
- Internet connection for initial downloads
- About 60 minutes

---

# PART 1: WSL2 Setup

**Environment: Windows PowerShell (Admin) for install, Ubuntu terminal once installed**

## 1.1 Install WSL2

```powershell
# Windows PowerShell (Admin)
wsl --install -d Ubuntu
```

Restart when prompted. After reboot, an Ubuntu terminal opens automatically and asks for a Linux username and password. Pick something memorable -- you'll use it throughout.

Verify it worked:

```powershell
# Windows PowerShell
wsl -l -v
```

You should see Ubuntu listed with VERSION 2.

### Sanity check

Typing `wsl` in PowerShell should drop you directly into a prompt like `yourname@MSI:~$`. If it shows a list of distros to install instead, the setup didn't complete -- re-run `wsl --install -d Ubuntu` and reboot.

## 1.2 Configure WSL2 Memory

Create `C:\Users\<yourname>\.wslconfig` in Notepad:

```powershell
# Windows PowerShell
notepad $env:USERPROFILE\.wslconfig
```

Click Yes when Notepad asks to create the file. Paste:

```ini
# Windows config file (Notepad)
[wsl2]
memory=8GB
processors=6
swap=4GB
localhostForwarding=true
```

The math: 16GB system RAM split evenly -- 8GB for WSL2, 8GB for Windows. The RTX 4050's 6GB VRAM is **dedicated** and has nothing to do with system RAM, so there's no GPU memory competing here. WSL2 only needs about 2-3GB to run the agent stack (Ollama runtime ~500MB, OpenClaw ~200MB, Ubuntu ~300MB, headroom for the rest). The 8GB ceiling leaves Windows a comfortable half of system RAM for Chrome, VS Code, Teams, or whatever you normally have open alongside AI work.

Apply:

```powershell
# Windows PowerShell
wsl --shutdown
wsl
```

## 1.3 Update Ubuntu

In your Ubuntu terminal:

```bash
# WSL2 / Ubuntu terminal
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git build-essential
```

## 1.4 Verify CUDA is Available in WSL2

This is the key step that confirms your RTX 4050 is reachable from WSL2. Once a Windows NVIDIA GPU driver is installed, CUDA becomes available within WSL 2. The CUDA driver installed on Windows host will be stubbed inside WSL 2 as libcuda.so -- do not install any NVIDIA GPU Linux driver within WSL 2.

```bash
# WSL2 / Ubuntu terminal
nvidia-smi
```

You should see your RTX 4050 listed with driver version and CUDA version. If you see it, GPU passthrough is working. Move on.

If `nvidia-smi` isn't found, your Windows NVIDIA driver is too old. Update to 550+ from https://www.nvidia.com/drivers and reboot, then retry.

> **Important:** Never run `sudo apt install nvidia-*` inside WSL2. Installing a Linux NVIDIA driver overwrites the Windows CUDA stub and breaks everything. The Windows driver is all you need.

---

# PART 2: Install Ollama

**Environment: WSL2 / Ubuntu terminal**

## 2.1 Install Ollama

```bash
# WSL2 / Ubuntu terminal
curl -fsSL https://ollama.com/install.sh | sh
```

The installer detects the RTX 4050 through WSL2's CUDA stub automatically. You should see `NVIDIA GPU installed` in the output.

## 2.2 Configure Ollama for the MSI

Add to `~/.bashrc`:

```bash
# WSL2 / Ubuntu terminal
cat >> ~/.bashrc << 'EOF'

# Ollama config for MSI Thin
export OLLAMA_HOST=0.0.0.0:11434
export OLLAMA_MAX_LOADED_MODELS=1
export OLLAMA_KEEP_ALIVE=10m
export OLLAMA_NUM_PARALLEL=1
export OLLAMA_FLASH_ATTENTION=1
EOF

source ~/.bashrc
```

What each setting does:

- `OLLAMA_HOST=0.0.0.0:11434` -- allows LM Studio on Windows to connect via localhost forwarding
- `OLLAMA_MAX_LOADED_MODELS=1` -- only one model in VRAM at a time (6GB is plenty for one model but not two)
- `OLLAMA_KEEP_ALIVE=10m` -- unloads after 10 min idle to free VRAM
- `OLLAMA_NUM_PARALLEL=1` -- single inference at a time
- `OLLAMA_FLASH_ATTENTION=1` -- reduces KV cache memory pressure at long contexts

## 2.3 Start Ollama and Verify GPU

```bash
# WSL2 / Ubuntu terminal
ollama serve &
```

Wait a few seconds, then verify it's running:

```bash
# WSL2 / Ubuntu terminal
curl http://localhost:11434
```

Should return `Ollama is running`.

---

# PART 3: Pull Models

**Environment: WSL2 / Ubuntu terminal**

The MSI's 6GB VRAM is dedicated -- models load entirely into VRAM with no sharing with system RAM. Both recommended models fit cleanly.

## 3.1 Pull the Agent Models

```bash
# WSL2 / Ubuntu terminal
# General agentic tasks, 5.5GB at Q4_K_M, right at the 6GB limit
# Keep context under 8K to avoid VRAM pressure
ollama pull qwen3.5:4b

# Coding sessions, 4.2GB at Q4_K_M, fits with 1.8GB headroom
ollama pull qwen2.5-coder:7b

# Embedding model for OpenClaw memory, 280MB
ollama pull nomic-embed-text
```

## 3.2 Verify GPU Inference

Test that the RTX 4050 is actually doing the work:

```bash
# WSL2 / Ubuntu terminal
ollama run qwen3.5:4b "say hi in one sentence"
```

While it's responding, open another Ubuntu terminal and run:

```bash
# WSL2 / Ubuntu terminal
ollama ps
```

The `PROCESSOR` column should say `100% GPU`. That confirms CUDA is working through WSL2.

If it says `100% CPU`, jump to the Troubleshooting section.

## 3.3 Model Notes for 6GB VRAM

**Qwen3.5 4B** fits comfortably at ~2.5GB -- less than half the 6GB ceiling. This gives you ~3.5GB of headroom for the KV cache, meaning long contexts are handled cleanly without overflow. The hybrid Gated DeltaNet architecture also keeps KV cache growth small even at 262K context. This is your default agentic model.

**Qwen2.5 Coder 7B** at ~4.2GB fits with ~1.8GB headroom. Slightly tighter than the 4B but still fully in VRAM. Purpose-tuned for code -- better at completion, refactoring, and multi-file edits than the general 4B.

Practical rule: use Qwen3.5 4B as your OpenClaw default for agentic tasks and general reasoning. Switch to Qwen2.5 Coder 7B for dedicated coding sessions.

---

# PART 4: Install LM Studio (Windows)

**Environment: Windows**

LM Studio is your personal chat interface. It runs on Windows, manages its own model downloads independently, and connects to Ollama in WSL2 for inference.

## 4.1 Download and Install

Go to https://lmstudio.ai and download the Windows installer. Run the `.exe` and install normally.

## 4.2 Connect LM Studio to WSL2 Ollama

LM Studio can use Ollama as a backend instead of running its own inference engine.

1. Open LM Studio
2. Go to **Settings** (gear icon)
3. Find **Local Server** or **Inference Backend**
4. Set the base URL to `http://localhost:11434`

Because `localhostForwarding=true` is in your `.wslconfig`, `localhost:11434` on Windows transparently reaches Ollama running inside WSL2.

> **Verify the connection:** In LM Studio, check that your Ollama models (Qwen3.5 4B, Qwen2.5 Coder 7B) appear in the model list. If they don't, make sure Ollama is running in WSL2 (`wsl` in PowerShell, then `ollama serve &`).

## 4.3 LM Studio's Independent Model Store

LM Studio can also download and run its own models separately from Ollama -- completely independent. Browse the **Discover** tab to find and download models. These go to `C:\Users\<you>\.lmstudio\models\` on Windows and don't interact with Ollama's WSL2 store at all.

Use LM Studio's own models when:
- You want to test a model before committing to pulling it in Ollama
- You want a model for chat-only use that doesn't need to be in OpenClaw
- You want to compare a LM Studio download vs an Ollama pull

---

# PART 5: Install OpenClaw

**Environment: WSL2 / Ubuntu terminal**

OpenClaw runs inside WSL2, sandboxed from your Windows personal files. It connects to Ollama at `localhost:11434` -- same network, no bridging required.

## 5.1 Install OpenClaw

OpenClaw's one-liner handles Node.js and all dependencies:

```bash
# WSL2 / Ubuntu terminal
curl -fsSL https://openclaw.ai/install-cli.sh | bash
source ~/.bashrc
```

Verify:

```bash
# WSL2 / Ubuntu terminal
openclaw --version
node --version  # should show v22.x.x or later
```

### If the one-liner fails (manual install)

```bash
# WSL2 / Ubuntu terminal
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 22
nvm use 22
nvm alias default 22
npm install -g openclaw@latest
```

## 5.2 Run Onboarding

```bash
# WSL2 / Ubuntu terminal
openclaw onboard --install-daemon
```

Answers for the MSI setup:

| Prompt | Answer |
|--------|--------|
| Continue with installation? | **Yes** |
| Gateway network binding | **Loopback (127.0.0.1)** |
| Workspace path | Default (`~/.openclaw`) |
| Install daemon as background service | **Yes** |
| Model provider | **Custom OpenAI-compatible** |

Model provider details:

- **Provider name:** `ollama-local`
- **Base URL:** `http://localhost:11434/v1`
- **API key:** `ollama` (any string -- Ollama ignores it)
- **Model ID:** `qwen3.5:4b`
- **Context window:** `32768`

**Save the access token printed at the end** -- you need it to log into the OpenClaw Web UI.

## 5.3 Verify Models Are Visible

```bash
# WSL2 / Ubuntu terminal
openclaw models list
```

Both Qwen3.5 4B and Qwen2.5 Coder 7B should appear.

## 5.4 Switch Models

```bash
# WSL2 / Ubuntu terminal
# Switch to coding model
openclaw models set ollama-local/qwen2.5-coder:7b

# Switch back to general agentic default
openclaw models set ollama-local/qwen3.5:4b
```

## 5.5 Open the Web UI

In your Windows browser: **http://127.0.0.1:18789**

Paste the access token from onboarding. If you lost it:

```bash
# WSL2 / Ubuntu terminal
openclaw daemon status
```

## 5.6 Run OpenClaw Doctor

```bash
# WSL2 / Ubuntu terminal
openclaw doctor
```

This validates your Node.js version, Ollama connection, daemon status, and config integrity. Fix anything flagged before continuing.

---

# PART 6: Channel Integrations

**Environment: WSL2 / Ubuntu terminal + web browser + your phone (for WhatsApp)**

Test with Discord first, then graduate to WhatsApp when everything is confirmed working.

## 6.1 Why Test with Discord First

- No phone number at risk
- No QR code scanning
- Easy to wipe and retry
- Logs are clear and easy to read
- Validates the full Ollama → OpenClaw → channel pipeline end-to-end before involving WhatsApp

## 6.2 Discord Setup (Testing Channel)

### Create a Discord Application

1. Go to https://discord.com/developers/applications (use your Windows browser)
2. **New Application** → name it "MSI-AI-Test"
3. Sidebar → **Bot** → **Reset Token** → copy immediately
4. Enable **Message Content Intent** under Privileged Gateway Intents (mandatory -- the bot won't respond without this)
5. Save

### Invite the Bot to a Server

1. Sidebar → **OAuth2 > URL Generator**
2. Scopes: `bot`
3. Bot Permissions: Send Messages, Read Message History, Embed Links, Attach Files
4. Copy the generated URL, open it, authorize to a private test server

### Add the Channel in OpenClaw

```bash
# WSL2 / Ubuntu terminal
openclaw channels add
```

Select **discord** when prompted. OpenClaw installs `@openclaw/discord` automatically. Enter your bot token and server details when asked.

```bash
# WSL2 / Ubuntu terminal
openclaw daemon restart
```

### Test

In Discord: `@MSI-AI-Test hello`

While testing, watch what's happening in real time:

```bash
# WSL2 / Ubuntu terminal
openclaw daemon logs --follow
```

The bot should reply within a few seconds. If it does, the full pipeline is working: Discord → OpenClaw → Ollama → RTX 4050 → response.

## 6.3 WhatsApp Setup (Production Channel)

OpenClaw uses the Baileys library (open-source WhatsApp Web protocol):

- Your phone stays the primary device
- The MSI acts as a linked companion (same as WhatsApp Web)
- One QR scan, no Twilio, no Business API
- Counts as one of WhatsApp's 4 allowed linked devices
- Phone must stay online; 14+ days offline = session unlinks

> **WhatsApp ToS:** Personal use is fine. Bulk/automated outreach gets numbers banned.

### Use a Separate Phone Number

Strongly recommended. A second SIM, eSIM, or Google Voice number isolates your personal WhatsApp from the bot. If the bot number ever gets flagged, your real number is unaffected.

### Add the Channel

```bash
# WSL2 / Ubuntu terminal
openclaw channels add
```

Select **whatsapp**. OpenClaw installs `@openclaw/whatsapp` automatically. Follow the prompts:

| Prompt | Answer |
|--------|--------|
| Account type | **Separate phone just for OpenClaw** |
| DM policy | **Pairing (recommended)** |
| allowFrom | Leave default for now |
| Group policy | **allowlist** |

### Link the Device (QR Scan)

```bash
# WSL2 / Ubuntu terminal
openclaw channels login --channel whatsapp
```

A QR code prints in the terminal. On your phone:

1. WhatsApp → **Settings > Linked Devices > Link a Device**
2. Scan the QR code
3. Wait for "WhatsApp connected"

Credentials save to `~/.openclaw/credentials/whatsapp/`. Treat this folder like your WhatsApp password -- don't share or commit it.

### Restart and Test

```bash
# WSL2 / Ubuntu terminal
openclaw daemon restart
openclaw daemon logs  # look for "WhatsApp connected"
```

Send a message from another phone. If using pairing policy, approve the first sender:

```bash
# WSL2 / Ubuntu terminal
openclaw pairing approve whatsapp <CODE>
```

### Add an Allowlist (Recommended)

Once it works, restrict who can message the bot. Edit `~/.openclaw/openclaw.json`:

```bash
# WSL2 / Ubuntu terminal
nano ~/.openclaw/openclaw.json
```

```json
# WSL2 -- ~/.openclaw/openclaw.json
{
  "channels": {
    "whatsapp": {
      "dmPolicy": "allowlist",
      "allowFrom": ["+15551234567"],
      "groupPolicy": "allowlist",
      "groupAllowFrom": ["+15551234567"]
    }
  }
}
```

Use international format (+ country code, no spaces or dashes). Then:

```bash
# WSL2 / Ubuntu terminal
openclaw daemon restart
```

## 6.4 Running Both Channels

Keep Discord even after WhatsApp works -- it's a useful debug channel when something breaks.

```bash
# WSL2 / Ubuntu terminal
openclaw channels disable discord   # turn off without removing
openclaw channels enable discord
openclaw channels remove discord    # full removal
```

---

# PART 7: Auto-Start on Boot

**Environment: Both Windows and WSL2**

## 7.1 Ollama Auto-Start in WSL2

Enable systemd and create an Ollama service so it starts automatically when WSL2 boots:

```bash
# WSL2 / Ubuntu terminal
sudo nano /etc/wsl.conf
```

Add:

```ini
# Windows config file (Notepad)
[boot]
systemd=true

[user]
default=<your-linux-username>
```

Save (`Ctrl+O`, Enter, `Ctrl+X`). Create the systemd service:

```bash
# WSL2 / Ubuntu terminal
sudo tee /etc/systemd/system/ollama.service << 'EOF'
[Unit]
Description=Ollama Service
After=network.target

[Service]
ExecStart=/usr/local/bin/ollama serve
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
Environment="OLLAMA_KEEP_ALIVE=10m"
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_FLASH_ATTENTION=1"
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama
```

## 7.2 Auto-Start WSL2 on Windows Boot

Create `C:\Users\<yourname>\ai-startup.bat`:

```batch
:: Windows -- save as .bat file
@echo off
wsl -d Ubuntu -- bash -c "sudo systemctl start ollama && openclaw daemon start"
```

Add to Windows startup:

1. `Win+R` → `shell:startup` → Enter
2. Right-click → New → Shortcut → browse to `ai-startup.bat`
3. Name it "AI Startup"

After every reboot, WSL2 starts, Ollama loads, and OpenClaw is ready -- all within about 20 seconds.

## 7.3 Verify Auto-Start

Reboot your MSI, then:

```bash
# WSL2 / Ubuntu terminal
ollama ps
openclaw daemon status
```

Both should show running without you having manually started anything.

---

# Frequently Asked Questions

### What tok/s should I expect on the RTX 4050?

At full VRAM utilization with CUDA:

| Model | Tok/s |
|-------|-------|
| Qwen2.5 Coder 7B Q4_K_M | 50-70 tok/s |
| Qwen3.5 4B Q4_K_M | 80-110 tok/s |

These are significantly faster than the ROG Ally's AMD iGPU. Discrete CUDA with 6GB dedicated VRAM is a different league.

### Why is Qwen3.5 4B fast even at long contexts?

The Qwen3.5 4B uses a hybrid Gated DeltaNet architecture where 75% of layers use linear attention (state space model) rather than standard softmax attention. This means the KV cache grows much more slowly than a standard transformer -- context length has much less impact on memory and speed than it does on larger standard-architecture models. This is one of the main reasons to prefer it as the OpenClaw default over a larger model.

### Can I run other models in LM Studio at the same time as OpenClaw?

No -- not on 6GB VRAM. Only one model can be loaded at a time. If LM Studio loads a model, it takes the VRAM and Ollama's model gets evicted (or vice versa). Choose one at a time. The `OLLAMA_KEEP_ALIVE=10m` setting means Ollama frees VRAM after 10 minutes of idle, so switching between them works fine -- just with a reload delay.

### Is OpenClaw actually sandboxed in WSL2?

By default, yes. OpenClaw running in WSL2 can only access:

- The WSL2 Linux filesystem (`~/.openclaw`, `~/`, etc.)
- Anything you explicitly mount from Windows (`/mnt/c/`, `/mnt/d/`, etc.)

It cannot access your Windows Documents, OneDrive, Desktop, or personal files unless you explicitly mount and configure access. This is the primary reason for the WSL2 split architecture.

### What happens if I need OpenClaw to access a Windows file?

Mount the specific Windows folder inside WSL2:

```bash
# WSL2 / Ubuntu terminal
# Example: give OpenClaw access to a specific project folder
mkdir -p ~/windows-projects
sudo mount --bind /mnt/c/Users/<you>/Projects ~/windows-projects
```

Be deliberate about what you expose. Don't mount your entire C: drive.

### Can I add more models to the Ollama stack later?

Yes. Pull them in WSL2 and they appear automatically in OpenClaw:

```bash
# WSL2 / Ubuntu terminal
ollama pull <model-name>
openclaw models list  # verify it appears
openclaw models set ollama-local/<model-name>
```

### How do I update everything?

```bash
# WSL2 / Ubuntu terminal
# Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Models (re-pulling fetches updates)
ollama pull qwen3.5:4b
ollama pull qwen2.5-coder:7b

# OpenClaw
openclaw update --channel stable
```

LM Studio updates itself via its built-in updater.

---

# Troubleshooting

## GPU / CUDA Issues

### `nvidia-smi` not found or shows no GPU in WSL2

Your Windows NVIDIA driver is too old or not installed. Update to 550+ from https://www.nvidia.com/drivers (not Windows Update -- it lags behind). Reboot Windows, then:

```powershell
# Windows PowerShell
wsl --shutdown
wsl
```

Then retry `nvidia-smi` inside Ubuntu.

### Ollama running on CPU instead of GPU

```bash
# WSL2 / Ubuntu terminal
ollama ps
journalctl -u ollama --no-pager -n 50 | grep -i gpu
```

Common causes:

1. **NVIDIA driver too old** -- update to 550+ on Windows
2. **Linux NVIDIA driver accidentally installed inside WSL2** -- check with `dpkg -l | grep nvidia`. If you see any, remove them: `sudo apt remove --purge nvidia-*`
3. **CUDA stub missing** -- run `ls /usr/lib/wsl/lib/libcuda*`. If nothing is there, your Windows driver isn't exposing the CUDA stub. Reinstall the Windows driver.

### Out of memory during inference

The model is exceeding the 6GB VRAM ceiling. Options:

```bash
# WSL2 / Ubuntu terminal
# Switch to the smaller model
openclaw models set ollama-local/qwen2.5-coder:7b

# Or reduce context window
# In ~/.ollama/Modelfile or via Ollama API num_ctx parameter
```

## WSL2 Issues

### WSL2 eating too much RAM

Verify `.wslconfig` is set to `memory=12GB` (Part 1.2). Apply with `wsl --shutdown`. Open Task Manager on Windows and confirm the `Vmmem` process is capped.

### Can't reach `localhost:11434` from LM Studio on Windows

Make sure `localhostForwarding=true` is in `.wslconfig` and that Ollama has `OLLAMA_HOST=0.0.0.0:11434` set. A common cause is Ollama set to listen on `127.0.0.1` (loopback only inside WSL2) rather than `0.0.0.0` (all interfaces). Verify:

```bash
# WSL2 / Ubuntu terminal
curl http://localhost:11434  # should work
```

Then from Windows PowerShell:

```powershell
# Windows PowerShell
curl http://localhost:11434  # should also work if localhostForwarding is on
```

If WSL2 shows running but Windows can't reach it, restart WSL2:

```powershell
# Windows PowerShell
wsl --shutdown
wsl
```

### WSL2 dies after the laptop goes to sleep

Known WSL2 quirk on laptops. Either:

- Set Windows to never sleep when plugged in
- Add to `~/.bashrc`: `alias wakewsl="wsl --shutdown && wsl"` and run `wakewsl` after waking

## Ollama Issues

### Slow first response, then fast

Normal. The first prompt loads the model into VRAM (5-15 seconds on the RTX 4050). Subsequent prompts are fast. `OLLAMA_KEEP_ALIVE=10m` keeps the model warm for 10 minutes after last use.

### "Another instance of Ollama is running"

The systemd service and a manually started Ollama are conflicting:

```bash
# WSL2 / Ubuntu terminal
sudo systemctl stop ollama
pkill ollama
sudo systemctl start ollama
```

## OpenClaw Issues

### `openclaw: command not found`

The install didn't add OpenClaw to PATH. Source your profile:

```bash
# WSL2 / Ubuntu terminal
source ~/.bashrc
```

If still not found, check where it was installed:

```bash
# WSL2 / Ubuntu terminal
which openclaw || ls ~/.nvm/versions/node/*/bin/openclaw 2>/dev/null
```

### OpenClaw can't reach Ollama

```bash
# WSL2 / Ubuntu terminal
curl http://localhost:11434/v1/models
```

If empty, Ollama isn't running. Start it:

```bash
# WSL2 / Ubuntu terminal
sudo systemctl start ollama
```

If it returns models but OpenClaw still fails, check your config has `"baseUrl": "http://localhost:11434/v1"` (the `/v1` suffix is required -- a common typo).

### Tool calls fail silently

Reasoning models inject `<think>` tokens that break OpenClaw's tool-call parser. Qwen3.5 4B and Qwen2.5 Coder 7B both have reasoning modes that can be triggered. If tool calls are failing, switch to Qwen2.5 Coder 7B which has more predictable tool-call formatting:

```bash
# WSL2 / Ubuntu terminal
openclaw models set ollama-local/qwen2.5-coder:7b
```

### Daemon won't start

```bash
# WSL2 / Ubuntu terminal
openclaw daemon logs
openclaw doctor --repair
```

Common causes: invalid JSON in `~/.openclaw/openclaw.json`, port 18789 in use, or Node.js version too old (needs 22+).

## Channel Issues

### Discord: bot online but no replies

Almost always **Message Content Intent** not enabled. Re-check the Developer Portal → Bot → Privileged Gateway Intents, toggle on, save. Then:

```bash
# WSL2 / Ubuntu terminal
openclaw daemon restart
```

### WhatsApp: QR code expired

QR codes expire every 60 seconds. Re-run:

```bash
# WSL2 / Ubuntu terminal
openclaw channels login --channel whatsapp
```

### WhatsApp: keeps disconnecting

- Phone offline >14 days → session expired, re-pair
- Hit the 4-linked-device limit → remove unused devices in WhatsApp Settings → Linked Devices
- Re-pair from scratch:

```bash
# WSL2 / Ubuntu terminal
rm -rf ~/.openclaw/credentials/whatsapp/
openclaw channels login --channel whatsapp
```

---

## When to Ask for Help

- **Ollama on NVIDIA WSL2:** r/LocalLLaMA, Ollama GitHub issues
- **OpenClaw:** OpenClaw Discord
- **WSL2 CUDA:** Microsoft WSL GitHub repo

Capture this before posting:

```bash
# WSL2 / Ubuntu terminal
echo "=== System ===" && uname -a
echo "=== GPU ===" && nvidia-smi
echo "=== Ollama ===" && ollama --version && ollama ps
echo "=== OpenClaw ===" && openclaw --version && openclaw doctor
echo "=== Node ===" && node --version
echo "=== Memory ===" && free -h
```

---

**You now have a sandboxed local AI agent on your MSI Thin.**

OpenClaw can't touch your personal Windows files unless you explicitly allow it. Ollama runs your models on the RTX 4050 at full CUDA speed. LM Studio gives you a polished chat interface on the Windows side that's completely independent. And Discord or WhatsApp gives you mobile access to the whole thing from anywhere.

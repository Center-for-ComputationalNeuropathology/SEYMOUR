# SEYMOUR
# 🧠 OpenClaw — Farrell Lab AI Agent Infrastructure

> Local AI agent system running on Mac Studio hardware, powering **Seymour** and **Hermes** — two purpose-built research agents for the Farrell Lab at the Icahn School of Medicine at Mount Sinai.

---

## Overview

This repository contains the configuration, agent identities, skills, and deployment infrastructure for a locally-hosted AI agent system built around [OpenClaw](https://github.com/openclaw) with a [Qwen3-27B](https://huggingface.co/Qwen/Qwen3-27B) model backend served via `llama-server`.

The system is designed to support computational neuroscience and neuropathology research workflows — including HPC job management, bioinformatics pipelines, data analysis, and lab automation — accessible via Slack from anywhere.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│              Mac Studio ("seymouracstudio")          │
│                                                     │
│  ┌─────────────┐     ┌──────────────────────────┐  │
│  │ llama-server │────▶│        OpenClaw          │  │
│  │  Qwen3-27B  │     │  ┌──────┐  ┌──────────┐  │  │
│  └─────────────┘     │  │Seymour│  │  Hermes  │  │  │
│                      │  └──────┘  └──────────┘  │  │
│                      └──────────┬───────────────┘  │
│                                 │                   │
│                      ┌──────────▼───────────────┐  │
│                      │   Slack Integration       │  │
│                      └──────────┬───────────────┘  │
│                                 │                   │
│                      ┌──────────▼───────────────┐  │
│                      │   Cloudflare Tunnel       │  │
│                      └──────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │  Remote Users      │
                    │  (Slack / Browser) │
                    └────────────────────┘
```

| Component | Role |
|---|---|
| `llama-server` | Serves Qwen3-27B locally via OpenAI-compatible API |
| OpenClaw | Agent runtime — orchestrates tools, memory, and agent identities |
| Seymour | Primary research agent; bioinformatics and HPC focus |
| Hermes | Secondary agent; different identity and task scope |
| Slack | Chat interface for human-agent interaction |
| Cloudflare Tunnel | Secure remote access to dashboard and services |
| LaunchAgent | macOS persistence — agents restart on boot |
| claude-guardrails | Safety layer with configurable deny rules |

---

## Agents

### 🤖 Seymour

Seymour is the primary lab agent, optimized for computational biology and HPC workflows. His identity, tone, and capabilities are defined in `agents/seymour/SOUL.md`.

**Core capabilities:**
- Submit and monitor LSF jobs on Minerva (Icahn HPC cluster)
- Run bioinformatics pipelines (CellRanger, GATK, PLINK2, bcftools, etc.)
- Assist with R and Python scripting for genomics and transcriptomics
- Manage file I/O and data organization on HPC storage
- Summarize job logs and flag errors

### 🪽 Hermes

Hermes is a second agent with a distinct identity and complementary task profile. His configuration lives in `agents/hermes/SOUL.md`.

**Core capabilities:**
- Literature and knowledge synthesis
- General lab communication and writing support
- Task routing and delegation
- Workflow documentation

### SOUL.md

Each agent has a `SOUL.md` file defining its identity, personality, tone, and behavioral boundaries. This is loaded at startup and shapes how the agent interacts with users.

```
agents/
├── seymour/
│   └── SOUL.md
└── hermes/
    └── SOUL.md
```

---

## Infrastructure

### Hardware

| Item | Spec |
|---|---|
| Machine | Mac Studio ("seymouracstudio") |
| Model | Apple Silicon |
| Role | Always-on local inference + agent host |

### Model Serving

Qwen3-27B is served locally using `llama-server` with an OpenAI-compatible endpoint. OpenClaw connects to this endpoint to power both agents.

```bash
# Example: start llama-server
llama-server \
  --model /path/to/qwen3-27b.gguf \
  --host 0.0.0.0 \
  --port 8080 \
  --ctx-size 8192
```

### Slack Integration

Both agents are reachable via dedicated Slack channels. Messages are routed to the appropriate agent by OpenClaw based on channel or mention.

### Cloudflare Tunnel

The OpenClaw dashboard and agent API are exposed remotely via a Cloudflare tunnel — no port forwarding required.

```bash
# Start tunnel
cloudflared tunnel run <tunnel-name>
```

### LaunchAgent (macOS Persistence)

Agents and the model server are registered as macOS LaunchAgents and start automatically on boot.

```
~/Library/LaunchAgents/
├── com.farrelllab.openclaw.plist
├── com.farrelllab.llama-server.plist
└── com.farrelllab.cloudflared.plist
```

---

## Setup

### Prerequisites

- macOS (Apple Silicon recommended)
- `llama.cpp` with `llama-server` binary
- Qwen3-27B GGUF model weights
- [OpenClaw](https://github.com/openclaw) installed
- Slack workspace with bot token
- Cloudflare account and `cloudflared` CLI

### Installation

**1. Clone this repo**
```bash
git clone https://github.com/your-org/openclaw-farrelllab.git
cd openclaw-farrelllab
```

**2. Start the model server**
```bash
llama-server --model /path/to/qwen3-27b.gguf --port 8080
```

**3. Configure OpenClaw**
```bash
cp config/openclaw.example.yaml config/openclaw.yaml
# Edit openclaw.yaml: set model endpoint, Slack token, agent paths
```

**4. Load agent identities**

Ensure `agents/seymour/SOUL.md` and `agents/hermes/SOUL.md` are present and properly formatted.

**5. Start OpenClaw**
```bash
openclaw start --config config/openclaw.yaml
```

**6. Start Cloudflare tunnel**
```bash
cloudflared tunnel run farrelllab
```

**7. Register LaunchAgents**
```bash
cp launchagents/*.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.farrelllab.openclaw.plist
```

---

## Configuration

### `config/openclaw.yaml`

Main configuration file. Key sections:

```yaml
model:
  endpoint: http://localhost:8080/v1
  name: qwen3-27b

agents:
  - name: seymour
    soul: agents/seymour/SOUL.md
    skills: skills/bioinformatics/
  - name: hermes
    soul: agents/hermes/SOUL.md

slack:
  bot_token: ${SLACK_BOT_TOKEN}
  channels:
    seymour: C0XXXXXXXXX
    hermes: C0YYYYYYYYY

guardrails:
  config: config/guardrails.yaml
```

### Bioinformatics Skills Pack

Located in `skills/bioinformatics/`, this pack gives Seymour knowledge of common tools and patterns:

- LSF job submission (`bsub`, `bjobs`, `bkill`)
- CellRanger, CellBender, Vireo, cellsnp-lite
- GATK, BWA-MEM, bcftools, PLINK2, REGENIE, SAIGE
- R/Python scripting conventions for genomics

### Guardrails

`config/guardrails.yaml` defines deny rules — operations the agents will refuse regardless of instruction.

```yaml
deny:
  - pattern: "rm -rf /"
    reason: "Destructive filesystem operation"
  - pattern: "drop table"
    reason: "Destructive database operation"
  # Add additional rules as needed
```

---

## Usage

### Talking to the Agents via Slack

Send a message in the agent's Slack channel or mention them directly:

```
@seymour check the status of my CellRanger job on Minerva
@hermes summarize the methods section of this paper
@seymour submit a PLINK2 PCA job with these parameters: ...
```

### Example Workflows

**Monitor an HPC job:**
```
@seymour bjobs -l 12345678
```

**Run a bcftools merge:**
```
@seymour merge the VCFs in /sc/arion/projects/myproject/vcfs/ and filter to MAF > 0.01
```

**Get a pipeline summary:**
```
@seymour summarize the last CellBender run log for KWO-11
```

### Session Management

OpenClaw maintains session context in `sessions.json`. If context overflows (very long conversations), sessions are cleared automatically to free memory. You can also manually clear:

```bash
openclaw sessions clear --agent seymour
```

---

## Known Issues & Gotchas

| Issue | Description | Workaround |
|---|---|---|
| **macOS TCC permissions** | `/usr/libexec/sshd-session` may be blocked from accessing files | Grant Full Disk Access to `sshd-session` in System Settings → Privacy & Security |
| **sessions.json overflow** | Long conversations cause context to overflow and sessions to auto-clear | Break long tasks into smaller sessions; context limit is model-dependent |
| **Duplicate gateway processes** | OpenClaw gateway may spawn duplicate processes on restart | Run `pkill -f openclaw-gateway` before restarting |
| **Cloudflare tunnel reconnects** | Tunnel may drop and reconnect under poor network conditions | Check `cloudflared` logs; LaunchAgent will restart it automatically |
| **Qwen3 context window** | Very long prompts or tool outputs may exceed context | Increase `--ctx-size` in llama-server flags (within RAM limits) |

---

## Repository Structure

```
.
├── agents/
│   ├── seymour/
│   │   └── SOUL.md
│   └── hermes/
│       └── SOUL.md
├── config/
│   ├── openclaw.example.yaml
│   └── guardrails.yaml
├── skills/
│   └── bioinformatics/
│       ├── lsf.md
│       ├── cellranger.md
│       ├── gatk.md
│       └── ...
├── launchagents/
│   ├── com.farrelllab.openclaw.plist
│   ├── com.farrelllab.llama-server.plist
│   └── com.farrelllab.cloudflared.plist
├── scripts/
│   └── (utility scripts)
└── README.md
```

---

## Roadmap

- [ ] Add YOLO/WSI analysis skill for Seymour (neuropathology image inference)
- [ ] Minerva SSH tool integration for direct job submission
- [ ] Agent memory persistence across sessions
- [ ] Multi-agent task delegation (Seymour ↔ Hermes hand-off)
- [ ] Web dashboard for session monitoring
- [ ] Additional HPC skills: SAIGE, REGENIE, PLINK2 GWAS workflows

---

## Lab & Attribution

**Farrell Lab** — Computational Neuropathology & AI/Human Health  
Icahn School of Medicine at Mount Sinai  
PI: Kurt Farrell, Assistant Professor (Pathology, Neuroscience, AI/Human Health)

This infrastructure supports research in tauopathies, TDP-43 proteinopathies, single-cell genomics, GWAS, and deep learning applied to neuropathology.

---

## License

MIT License — see [LICENSE](LICENSE) for details.

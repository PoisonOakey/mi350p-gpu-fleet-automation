# AMD Instinct™ MI350P DCGPU Fleet Provisioning Automation

> A case study on the highly concurrent, config-driven fleet installer I built for managing drivers and runtimes across 25+ enterprise bare-metal AMD Instinct™ MI350P Data Center GPU servers in Penang & Austin.
> **No proprietary code is included.** This documents the architecture, skills, and outcomes.

---

## What I Built

A cocurrent, config-driven fleet installer that deploys and manages software stacks across 25+ bare-metal GPU servers via SSH — from a single Windows workstation.

**One command** deploys the full stack (GPU runtime, kernel drivers, diagnostics, benchmarks) to the entire fleet with automatic reboot handling.

>  [!IMPORTANT]
>  _**Why custom, not Ansible?**_
>
> Bare-metal hosts, no agents, no cloud APIs — SSH was the only access. Purpose-built for fast version rotations where the team needed a single command, not a playbook.

---

## The Problem

- 25+ GPU servers needed identical software stacks (runtime, drivers, firmware, diagnostic tools)
- BKC (Best Known Configuration) versions changed every 1-2 weeks
- Manual installation = 1-2 hours per host, error-prone, inconsistent
- Driver installs require kernel module ```unload -> reboot -> reinstall -> reboot``` — across every host

---

## Installation Dependencies

Compenents must be installed in order - later stages depend on earlier ones.

<img width="381" height="485" alt="Untitled Diagram drawio (3)" src="https://github.com/user-attachments/assets/8f7a1d2e-3031-4114-9228-a7f629fa6f93" />

<br>
<br>

>[!WARNING]
>Reinstalling Stage 1 wipes Stage 3-4 - must reinstall them after.

---

## What I Did

| Area | Detail |
|---|---|
| **Language** | Python 3 |
| **Remote execution** | Paramiko (SSH) — no agents on target hosts |
| **Config management** | YAML config for all versions, URLs, and host inventory |
| **Parallelism** | ThreadPoolExecutor — up to 12 hosts simultaneously |
| **Reboot orchestration** | SSH polling loop with backoff for driver installs requiring kernel reloads |
| **Logging** | Per-host log files with dual output (console + file) |
| **Deployment model** | Push-based — script runs from operator's machine, pushes installs outward |

---

## Architecture

```text
┌───────────┐    ┌────────────────┐    ┌─────────────────┐
│ CLI args  │───>│ Load configs   │───>│ Thread pool     │
│ --tasks   │    │ YAML -> dict   │    │ 1 thread/host   │
│ --config  │    │                │    │                 │
│ --hosts   │    │ versions.yml   │    │ SSH -> run bash │
└───────────┘    │ hosts.yml      │    │ per host        │
                 └────────────────┘    └─────────────────┘
```

| Layer | Role | Description |
|---|---|---|
| **1. CLI Parser** | Entry point | Parses `--tasks`, `--config`, `--hosts` arguments |
| **2. Config Loader** | Read YAML | Loads version numbers and host inventory from YAML files |
| **3. Translation Layer** | YAML -> URLs | Converts human-readable versions into fully resolved download URLs and filenames |
| **4. Thread Pool** | Parallelism | Spawns one thread per host (up to 12), queues the rest |
| **5. SSH Executor** | Remote execution | Single function that runs any command over SSH, captures stdout/stderr/exit code |
| **6. Task Router** | Dispatch | Maps each `--tasks` argument to the right install script(s) |
| **7. Reboot Handler** | Orchestration | Closes SSH, polls every 30s until host is back, reconnects |
| **8. Per-Host Logger** | Observability | Dual output - console (live) + file (permanent record per host) |

---

## Project Structure

```text
project/
├── fleet.py               # Single-file installer (all logic here)
├── config/
│   ├── versions.yml       # Tool versions, URLs, build numbers
│   └── hosts.yml          # Target host inventory + credentials
├── docs/
│   ├── Architecture.md    # How the tool works (layers, flow, decisions)
│   ├── Workflow_Guide.md  # Install order, dependencies, gotchas
│   ├── Future_Plan.md     # Planned improvements
│   └── Changelog.md       # Version history per release
├── archive/               # Previous script versions (reference)
├── logs/                  # Per-host logs (generated at runtime)
├── requirements.txt       # Python dependencies (paramiko, pyyaml)
└── README.md
```

---

## Outcomes

| Metric | Before | After |
|---|---|---|
| Time to deploy full stack per host | 1-2 hours (manual) | ~15 min (automated) |
| Fleet-wide BKC update | Full day+ | Single command, walk away |
| Version drift across fleet | Common | Eliminated (config-driven) |
| Version update process | Edit Python code | Edit YAML config |
| Documentation | None | Architecture guide, workflow guide, changelog |

> [!NOTE]
> Given more time: idempotent installs (skip if already at target version), dry-run mode, and retry logic for transient failures.

---

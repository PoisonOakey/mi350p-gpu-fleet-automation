# AMD Instinct™ MI350P DCGPU Fleet Provisioning Automation

> A case study on the highly concurrent, config-driven fleet installer I built for managing drivers and runtimes across 20+ enterprise bare-metal AMD Instinct™ MI350P Data Center GPU servers in Penang & Austin.
> 
> **No proprietary code is included.** This documents the architecture, skills, and outcomes.

---

## :wrench: What I Built

A cocurrent, config-driven fleet installer that deploys and manages software stacks across 20+ bare-metal GPU servers via SSH — from a single Windows workstation.

**One command** deploys the full stack (GPU runtime, kernel drivers, diagnostics, benchmarks) to the entire fleet with automatic reboot handling.

>  [!IMPORTANT]
>  _**Why custom, not Ansible?**_
>
> Bare-metal hosts, no agents, no cloud APIs — SSH was the only access. Purpose-built for fast version rotations where the team needed a single command, not a playbook.

---

## :anger: The Problem

- 20+ GPU servers needed identical software stacks (runtime, drivers, firmware, diagnostic tools)
- BKC (Best Known Configuration) versions changed every 1-2 weeks
- Manual installation = 1-2 hours per host, error-prone, inconsistent
- Driver installs require kernel module ```unload -> reboot -> reinstall -> reboot``` — across every host

---

## :open_file_folder: Installation Dependencies

Compenents must be installed in order - later stages depend on earlier ones.

<img width="507" height="647" alt="lovestory drawio" src="https://github.com/user-attachments/assets/6935e923-a0a0-42b4-9a22-f5db2b8a2e99" />

<br>
<br>

>[!WARNING]
>Reinstalling Stage 1 wipes Stage 3-4 - must reinstall them after.

---

## :thought_balloon: What I Did

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

## :house_with_garden: Architecture

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

## :evergreen_tree: Project Structure

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

## :computer: Outcomes

| Metric | Before | After |
|---|---|---|
| Time to deploy full stack per host | 1-2 hours (manual) | ~15 min (automated) |
| Fleet-wide BKC update | Full day+ | Single command, walk away |
| Version drift across fleet | Common | Eliminated (config-driven) |
| Version update process | Edit Python code | Edit YAML config |
| Documentation | None | Architecture guide, workflow guide, changelog |

> [!NOTE]
> ### Additional features implemented:
> - **Idempotent installs** — checks current version on each host before installing, skips if already at target (`--force` to override)
> - **BKC Gatekeeper** — side-by-side Expected vs Actual version comparison table for fleet compliance auditing
> - **Improved logging** — ✅/❌ pass/fail markers per step with summary block at end of each host run
>   
> _**Still planned**_: dry-run mode, retry logic for transient failures, full suite idempotency.


---

# Container Engine (Tiers 1 & 2)

## Overview

Tiers 1 (Free / Community) and 2 (Starter) run each agent workspace in a **multi-tenant container** on a shared host kernel instead of a dedicated microVM. This keeps unit economics viable for free and low-cost users. The trade-offs are explicit: no Docker-in-Docker, no full desktop, and a weaker isolation boundary than Tier 3.

The engine is one of two runtimes behind the dual-engine orchestrator (see [`high-level-architecture.md`](./high-level-architecture.md)). Tier 3 and Tier 4 use the [microVM engine](./microvm-engine.md).

---

## Runtime Layout

```
┌─────────────────────────────────────────────────────────────┐
│                    BARE-METAL HOST (Linux)                  │
│                                                             │
│   Host Node Daemon ──► Container Runtime (Docker / gVisor)  │
│                                                             │
│   ┌──────────────────────────┐   ┌──────────────────────┐   │
│   │ Tier 1/2 Workspace Pod   │   │ Tier 1/2 Workspace   │   │
│   │ - Agent process          │   │ (another tenant)     │   │
│   │ - Node / Python toolchain│   │                      │   │
│   │ - Headless Chromium      │   │                      │   │
│   │ - /workspace (ephemeral  │   │                      │   │
│   │   or persistent volume)  │   │                      │   │
│   └──────────────────────────┘   └──────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

Each pod is an isolated container with:
- An agent process and the project toolchain (Node, Python, etc.)
- **Headless Chromium** driven by Puppeteer for scripted browser checks (no interactive desktop)
- A `/workspace` volume, ephemeral on Tier 1 and persistent on Tier 2

---

## Resource Limits

| Resource | Tier 1 | Tier 2 |
|---|---|---|
| vCPUs | 1 | 2 |
| RAM | 2 GB | 4 GB |
| Storage | 5 GB ephemeral (wiped after 7 days) | 20 GB persistent NVMe |
| Concurrent agents | 1 | 2 |

Limits are enforced with cgroups v2. A workspace that exceeds its memory limit is OOM-killed, which is the main upgrade driver to Tier 3 (see [`tier-breakdown.md`](../01-vision-and-strategy/tier-breakdown.md)).

---

## Isolation Model

Containers share the host kernel, so isolation relies on layered controls:

1. **Unprivileged containers** with dropped Linux capabilities and a default seccomp profile.
2. **gVisor (optional hardening)** interposes a user-space kernel for syscalls on untrusted workloads.
3. **No Docker socket** and no `--privileged`. Docker-in-Docker is disabled on these tiers.
4. **Network policy** from the host eBPF layer: block cloud metadata (`169.254.169.254`), block RFC1918 ranges between tenants, and block SMTP and crypto-mining pools (see [`networking-and-ingress.md`](./networking-and-ingress.md)).

Because the kernel is shared, a container escape is a real risk. That is why the enterprise persona requires Tier 4 microVMs (see [`target-personas.md`](../01-vision-and-strategy/target-personas.md)).

---

## Lifecycle

1. **Provision:** Scheduler picks a host with capacity, the daemon pulls the golden base image (cached per host), and attaches the workspace volume.
2. **Clone:** The repository is cloned into `/workspace` on the target branch.
3. **Run:** The agent executes commands. Preview ports are discovered and routed through ingress.
4. **Sleep / Resume:** Persistent volumes survive. Ephemeral volumes are discarded at the 7-day expiry.
5. **Destroy:** Container and ephemeral storage are removed.

Tier 1/2 workspaces have **no RAM forking** and **no full block-level undo**. They only get Git status rollback (see [`undo-and-time-machine.md`](../03-features-and-workflows/undo-and-time-machine.md)).

---

## Feature Support

| Capability | Tier 1 | Tier 2 |
|---|---|---|
| Docker Compose | ❌ | ❌ |
| Headless browser (Puppeteer) | ✅ | ✅ |
| Interactive desktop / WebRTC | ❌ | ❌ |
| Video PR proof | ❌ | ❌ |
| Human takeover | ❌ | ❌ |
| RAM forking | ❌ | ❌ |
| Git-based rollback | ✅ | ✅ |

---

## Open Questions

- Should gVisor be mandatory for all Tier 1 pods, or only for untrusted repos? The cost of gVisor is not yet measured.
- Should the 7-day ephemeral expiry warn the user before wiping `/workspace`?
- Can the headless Chromium on these tiers share the Tier 3 proof-recording pipeline, or is it a separate path?

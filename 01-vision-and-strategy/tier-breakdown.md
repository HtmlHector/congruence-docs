# Tier Breakdown & Monetization Architecture: Base Plans + Hardware Sliders

## Executive Summary

The platform operates on a **Hybrid Monetization Architecture**:
1. **Base Subscription Plans:** Provides platform access, Claude Code / Codex integration, Slack & Linear triggers, mobile diff reviews, and base hardware allocations.
2. **Dynamic Hardware Compute Sliders (Add-ons):** Lets developers seamlessly scale RAM, NVMe storage, extra parallel VM slots, or enable 24/7 Always-On mode as their codebase grows.

This model combines a low friction entry point ($0 and $19) with massive gross margins (>90%) on infrastructure upgrades.

---

## 1. The Core Base Plans

| Capability | **Tier 1 (Free)** | **Tier 2 (Starter)** | **Tier 3 (Pro Cloud PC)** ⭐ | **Tier 4 (Team / Enterprise)** |
|---|---|---|---|---|
| **Target Audience** | Open source, hobbyists, trial users | Solo developers, indie hackers | Full-stack engineers, tech leads | Engineering orgs, security teams |
| **Base Price** | **$0 / month** | **$19 / month** | **$49 / month** | **$99+ / mo or $149 / seat** |
| **Virtualization Engine** | Shared Docker Container | Isolated Docker Container | **Dedicated Micro-VM (KVM)** | **Dedicated Micro-VM (KVM)** |
| **Kernel Model** | Shared Host Linux Kernel | Shared Host Linux Kernel | **Dedicated Linux Kernel** | **Dedicated Linux Kernel** |
| **Base vCPU & RAM** | 1 vCPU • 2 GB RAM | 2 vCPUs • 4 GB RAM | **4 - 8 vCPUs • 16 GB RAM** | 8 - 16 vCPUs • 32 GB RAM |
| **Base Storage** | 10 GB Ephemeral (wipes 7d) | 25 GB Persistent NVMe | **60 GB Persistent NVMe** | 200 GB+ Dedicated NVMe |
| **Parallel Active Agents** | 1 Active Agent | 2 Parallel Agents | **5 Parallel Micro-VMs** | 15+ Parallel Micro-VMs |
| **Hardware Sliders** | ❌ Fixed specs | ✅ Expandable RAM & SSD | ✅ **Fully Custom Specs** | ✅ **Cluster-Wide Pools** |
| **Docker-in-Docker** | ❌ Blocked (security risk) | ❌ Blocked | ✅ **Full `docker compose`** | ✅ **Full `docker compose`** |
| **Display / GUI** | Headless scripts only | Headless Chrome (Puppeteer) | ✅ **Full Desktop (XFCE + Chrome)** | ✅ **Full Desktop + Multi-User** |
| **Live Staging Preview** | ❌ None | ❌ None | ✅ **Automatic Wildcard SSL URL** | ✅ **Custom Domain Preview** |
| **Live Screen Stream** | ❌ None | ❌ None | ✅ **WebRTC Sub-80ms Stream** | ✅ **WebRTC + Team Collab** |
| **Human Takeover** | ❌ None | ❌ None | ✅ **1-Click Web Handover** | ✅ **1-Click Handover + RBAC** |
| **RAM Forking (300ms)** | ❌ None | ❌ None | ✅ **Live Memory Cloning** | ✅ **Cluster-Wide Memory Fork** |
| **Time Machine Undo** | Git status rollback only | Git status rollback only | ✅ **Full OS Block-Level Undo** | ✅ **Full OS Block-Level Undo** |
| **Token Cost Optimizer** | ❌ None | Basic file search | ✅ **Local AST Context Daemon** | ✅ **Team Context Daemon** |
| **Private Networking** | ❌ None | ❌ None | ✅ **Tailscale / WireGuard** | ✅ **VPC Peering / Dedicated IP** |

---

## 2. The Hardware Add-on Menu (Compute Sliders)

Developers with large monorepos, local database requirements, or heavy build pipelines can customize their machines using interactive sliders in their dashboard:

```
┌────────────────────────────────────────────────────────────────────────┐
│                   DASHBOARD HARDWARE SLIDER CONTROLS                   │
│                                                                        │
│   Plan Base:        Tier 3 Pro Cloud PC                     $49 / mo   │
│   Extra RAM:        [ +8 GB RAM (Total 24 GB) ]             +$10 / mo  │
│   Extra Storage:    [ +100 GB NVMe (Total 160 GB) ]         +$10 / mo  │
│   Extra Parallel:   [ +2 Parallel VMs (Total 7 VMs) ]       +$20 / mo  │
│   Always-On Mode:   [ Enabled (24/7 No-Sleep) ]             +$15 / mo  │
│                                                                        │
│   Total Monthly Subscription:                               $104 / mo  │
└────────────────────────────────────────────────────────────────────────┘
```

### Add-on Pricing & Gross Profit Margins:

| Hardware Add-on | Monthly Price to User | Estimated Hetzner Cost | Gross Profit Margin |
|---|---|---|---|
| **Extra RAM (+4 GB)** | **$5 / month** | ~$0.40 / month | **~92% profit** |
| **Extra NVMe SSD (+50 GB)** | **$5 / month** | ~$0.50 / month | **~90% profit** |
| **Extra Parallel VM Slot** | **$10 / month** | ~$0.80 / month | **~92% profit** |
| **Always-On Mode (Never sleeps)** | **$15 / month** | ~$1.50 / month | **~90% profit** |
| **Dedicated Static IPv4** | **$5 / month** | ~$1.20 / month | **~76% profit** |

---

## 3. Deep Dive: What is "Always-On Mode"?

### Standard Behavior vs. Always-On

* **Default Behavior (Auto-Sleep / Hibernation):**
  * When an agent finishes a task or the user closes their browser, the Micro-VM waits 10 minutes.
  * If no new input or webhook arrives, the VM automatically **hibernates** (RAM is saved to NVMe disk and CPU allocation drops to 0%).
  * When a new prompt or webhook arrives, the VM **wakes up in ~1 second** with all windows and files preserved.
  * *Benefit:* Saves compute resources, allowing low base subscription costs.

* **Always-On Mode ($15/mo add-on):**
  * **Sleep is completely disabled.** The Micro-VM's CPU, RAM, and background daemons run continuously **24 hours a day, 7 days a week, 365 days a year**.

### Why Developers Pay for Always-On:
1. **Zero-Latency Staging Previews (0ms Wake):**
   * If a developer shares their preview link (`https://my-app.congruence.dev`) with a client, investor, or team lead in another timezone, the link opens instantly at any hour without waiting for a 1-second cold-wake.
2. **Background Cron Jobs & Queue Workers:**
   * Running background processes that must fire on fixed intervals (e.g. data scrapers, database health checks, Celery/BullMQ queues).
3. **Live Webhook Listeners:**
   * Applications listening for live webhooks from Stripe, Shopify, GitHub, or IoT devices that require instantaneous socket responses.

---

## 4. Bare-Metal Unit Economics (The Hetzner Model)

Using **one Hetzner AX42 Dedicated Bare-Metal Server** as a deployment unit:
* **Server Specs:** AMD Ryzen 7 7700 (8 cores, 16 threads, 5.3 GHz), 64 GB DDR5 RAM, 2x 1TB NVMe SSDs.
* **Monthly Host Cost:** **€59 / month** (~$65 USD).

### Revenue Model per Node:
* **Capacity:** 1 bare-metal node comfortably hosts:
  * **150+ dormant / sleeping Free tier users** (costing ~$0 when asleep).
  * **15 to 20 active Tier 3 ($49/mo) subscribers** with memory ballooning.
* **Gross Revenue:** 18 users × $49/mo = **$882 / month**
* **Hardware Add-ons:** ~4 users buy extra RAM/SSD/Always-on (+~$60/mo) = **$942 / month**
* **Server Cost:** **-$65 / month**
* **Net Margin per Node:** **~$877 / month (~93% Gross Margin)**

# Local Development Guide (macOS & Workstations)

## Overview

Because macOS uses its own native virtualization framework (`Hypervisor.framework`) rather than Linux KVM (`/dev/kvm`), developers building this platform need a seamless local development workflow that doesn't require purchasing expensive cloud servers for day-to-day coding.

This guide outlines how to build, run, and test the entire platform—including the **Graphical Desktop UI**, **Agent State Machine**, and **Execution Engines**—directly on a local Mac.

---

## 1. The Dual-Mode Development Strategy

Our platform's architecture is uniquely suited for local development because of its tiered design:

| Component | **Local Development on Mac** | **Production Cloud** |
|---|---|---|
| **Web Console & API** | Node.js / Go running on `localhost:8080` | Production Kubernetes / Nomad cluster |
| **Control Plane DB** | Local PostgreSQL in Docker | Managed PostgreSQL (RDS / Neon) |
| **Execution Engine** | **Docker Sandbox Mode (Tiers 1 & 2)** | **Cloud-Hypervisor Micro-VM (Tier 3)** |
| **Graphical Display** | `Xvfb` + `noVNC` inside local Docker container | `Xvfb` + WebRTC inside Micro-VM |
| **Desktop Preview** | `http://localhost:6080` in Chrome/Safari | `https://app.yourdomain.dev/viewer` |

> **Key Takeaway:** You can build, debug, and demo 95% of the platform—including live agent browser automation and mouse clicking—natively on your Mac without spinning up any cloud infrastructure.

---

## 2. Recommended Local VM & Container Runtimes for macOS

To run the local development environment on your Mac, choose one of the following runtimes:

### 1. OrbStack (Top Recommendation ⭐)
* **Website:** [orbstack.dev](https://orbstack.dev/)
* **Why it's best for this project:**
  * Replaces Docker Desktop, but **also spins up full native Ubuntu Linux VMs in under 2 seconds**.
  * Uses 1/10th the memory and CPU of Docker Desktop, keeping your Mac cool and saving battery.
  * Native two-way filesystem synchronization and automatic local networking domains (e.g. `http://my-desktop.orb.local`).

### 2. UTM (Best Native Visual VM GUI)
* **Website:** [mac.getutm.app](https://mac.getutm.app/)
* **Why use it:**
  * Free, open-source macOS app built directly on Apple's `Virtualization.framework`.
  * Perfect if you prefer an interactive native macOS window displaying your Ubuntu Linux desktop directly on your Mac desktop without needing a web browser.

### 3. Multipass (Canonical / Ubuntu CLI)
* **Website:** [multipass.run](https://multipass.run/)
* **Why use it:**
  * Command-line first tool built by Canonical.
  * Run `multipass launch --name dev-vm` to get a fresh Ubuntu instance in 20 seconds.

### 4. Docker Desktop (Standard Fallback)
* If already installed on your Mac, standard Docker Desktop is 100% compatible with our Tier 1 & 2 local development workflows.

---

## 3. Running the Graphical Desktop Locally on Mac

To test the graphical desktop, XFCE, Chromium, and input automation locally:

### A. Quickstart with Docker / OrbStack
Create a simple local test container running the display stack:

```dockerfile
# dev-desktop.Dockerfile
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y \
    xvfb \
    xfce4 \
    xfce4-terminal \
    chromium-browser \
    x11vnc \
    novnc \
    websockify \
    curl \
    git

# Startup script launching virtual display + desktop + web viewer
CMD ["sh", "-c", "Xvfb :1 -screen 0 1920x1080x24 & \
     DISPLAY=:1 startxfce4 & \
     x11vnc -display :1 -nopw -listen localhost -xkb & \
     websockify --web /usr/share/novnc 6080 localhost:5900"]
```

### B. Launching and Viewing the Desktop
1. Build and run the local desktop:
   ```bash
   docker build -t dev-desktop -f dev-desktop.Dockerfile .
   docker run -p 6080:6080 -d --name local-desktop dev-desktop
   ```
2. Open your browser on your Mac to:
   ```text
   http://localhost:6080/vnc.html
   ```
3. **Result:** An interactive 1080p Linux desktop renders in your browser tab. You can interact with it using your Mac's trackpad and keyboard.

---

## 4. Testing Real Micro-VMs (KVM Testing)

When you are ready to test the hardware-specific features of Tier 3 (raw Cloud-Hypervisor boots, live RAM forking, and kernel-level `/dev/uinput`):

1. **Option A: Nested Virtualization via OrbStack (Local on Mac)**
   - If using Apple Silicon / Intel Mac with modern virtualization, OrbStack can spin up an Ubuntu machine supporting software virtualization.
2. **Option B: $5/Month Remote KVM Dev Box (Recommended for Raw Kernel Testing)**
   - Spin up a low-cost bare-metal or KVM VPS (Hetzner, Vultr, or Scaleway).
   - Install Cloud-Hypervisor:
     ```bash
     curl -LO https://github.com/cloud-hypervisor/cloud-hypervisor/releases/latest/download/cloud-hypervisor
     chmod +x cloud-hypervisor
     ```
   - Connect via **VS Code Remote - SSH** from your Mac. You can edit host daemon code locally and execute microVM boots against real hardware.

# Implementation Roadmap & Engineering Milestones

## Overview

A structured, 4-phase engineering roadmap designed to take the platform from initial hypervisor proof-of-concept to production scale.

---

## Phase 1: Hypervisor & Display Core (Months 1–2)
* [ ] **Cloud-Hypervisor Harness:** Setup bare-metal Linux node with KVM and build Rust wrapper to launch minimal Linux guest VMs.
* [ ] **Guest Agent & `vsock`:** Implement lightweight Go guest daemon for process execution (`exec`) and memory reads over `virtio-vsock`.
* [ ] **Display Pipeline:** Configure `Xvfb` virtual display server, lightweight window manager (XFCE), and MIT-SHM screenshot capture.
* [ ] **WebRTC Streamer:** Integrate in-memory video encoding pipeline streaming 60fps to a simple test browser canvas.
* [ ] **Input Injection:** Verify mouse/keyboard simulation via `/dev/uinput`.

---

## Phase 2: Dual-Engine & Workstation MVP (Months 3–4)
* [ ] **Dual-Engine Orchestrator:** Build the unified control plane that allocates Docker containers (Tier 1/2) vs. Cloud-Hypervisor VMs (Tier 3).
* [ ] **Docker-in-MicroVM:** Pre-bake Docker daemon into the Golden Base OS image and test multi-container `docker-compose` stacks inside the VM.
* [ ] **Storage Layer & Fast CoW:** Implement read-only base rootfs + ephemeral Copy-on-Write overlay + persistent NVMe `/workspace` volume.
* [ ] **CLI & Basic Web Console:** Ship the terminal CLI for creating, listing, and connecting to workspaces.

---

## Phase 3: Developer Superpowers (Months 5–6)
* [ ] **Headful Browser Testing:** Build the automated test runner that opens Chromium to verify UI and console logs before PR creation.
* [ ] **1-Click Human Takeover:** Implement ephemeral signed URLs for seamless browser handover when agents hit 2FA/CAPTCHA walls.
* [ ] **RAM Forking & Time Machine:** Implement live memory snapshot cloning (sub-300ms fork) and block-level undo checkpoints.
* [ ] **Preview Ingress:** Dynamic wildcard SSL proxy routing `https://<box-id>-<port>.preview.yourdomain.dev`.

---

## Phase 4: Integrations & Enterprise Scale (Months 7+)
* [ ] **Linear & Slack Bots:** Native apps for assigning tickets directly to agents with real-time progress updates.
* [ ] **Mobile Companion App:** React Native / Flutter apps for iOS & Android with diff reviews and push approvals.
* [ ] **Enterprise Compliance:** SOC 2 audit logging, eBPF security network filtering, and private VPC peering / Tailscale integration.

# Cloud Workstation for AI Coding Agents
## Complete Architecture & Engineering Documentation

Welcome to the central architectural documentation for the **Next-Gen Cloud Workstation for AI Coding Agents** (working project codename: **Congruence**). 

This platform bridges the gap between lightweight developer sandboxes (like *Boxes.dev*) and low-level cloud microVM infrastructure (like *Maritime.sh*), providing coding agents (Claude Code, OpenAI Codex, etc.) with their own hardware-isolated, visual Cloud PCs.

---

## 📚 Documentation Index

### [01. Vision & Strategy](./01-vision-and-strategy/)
* **[`product-vision.md`](./01-vision-and-strategy/product-vision.md)**: The core problem, our solution, market positioning, and competitive comparison (vs. Boxes.dev & Maritime.sh).
* **[`tier-breakdown.md`](./01-vision-and-strategy/tier-breakdown.md)**: Pricing tiers, the dual-engine strategy (Docker vs. MicroVM), and upgrade drivers.
* **[`target-personas.md`](./01-vision-and-strategy/target-personas.md)**: Customer profiles, enterprise requirements, and buyer triggers.

### [02. System Architecture](./02-system-architecture/)
* **[`high-level-architecture.md`](./02-system-architecture/high-level-architecture.md)**: Full end-to-end system topology, component interactions, and host-guest relationships.
* **[`microvm-engine.md`](./02-system-architecture/microvm-engine.md)**: Deep dive into the KVM / Cloud-Hypervisor / Firecracker virtualization layer.
* **[`container-engine.md`](./02-system-architecture/container-engine.md)**: Architecture for Tiers 1 & 2 (lightweight, multi-tenant Docker sandboxes).
* **[`desktop-and-display.md`](./02-system-architecture/desktop-and-display.md)**: Xvfb, Wayland, WebRTC low-latency streaming, and human takeover mechanisms.
* **[`storage-and-snapshots.md`](./02-system-architecture/storage-and-snapshots.md)**: 3-tier storage architecture, NVMe persistence, and copy-on-write memory checkpointing.
* **[`networking-and-ingress.md`](./02-system-architecture/networking-and-ingress.md)**: eBPF egress filtering, vsock IPC control plane, and dynamic preview domain routing.

### [03. Features & Workflows](./03-features-and-workflows/)
* **[`docker-in-vm.md`](./03-features-and-workflows/docker-in-vm.md)**: Running Docker Compose, Postgres, Redis, and microservices inside the isolated microVM.
* **[`agent-notifications-and-states.md`](./03-features-and-workflows/agent-notifications-and-states.md)**: Real-time agent state machine, bi-directional Slack/Discord bots, Fleet Dock, and Kanban UI.
* **[`ui-design-system.md`](./03-features-and-workflows/ui-design-system.md)**: Frontend UI/UX specification blending Notion modularity and Vercel precision engineering.
* **[`browser-testing-and-live-preview.md`](./03-features-and-workflows/browser-testing-and-live-preview.md)**: Automated browser testing, screenshot inspection, and live preview links.
* **[`ram-forking-and-branching.md`](./03-features-and-workflows/ram-forking-and-branching.md)**: Instant 300ms cloning of active memory to let multiple agents test approaches in parallel.
* **[`undo-and-time-machine.md`](./03-features-and-workflows/undo-and-time-machine.md)**: Automated pre-command checkpoints and instantaneous block-level rollbacks.
* **[`developer-integrations.md`](./03-features-and-workflows/developer-integrations.md)**: VS Code/Cursor remote SSH, Slack/Linear ticket webhooks, and mobile companion app.

### [04. APIs & Protocols](./04-api-and-protocols/)
* **[`control-plane-api.md`](./04-api-and-protocols/control-plane-api.md)**: REST & WebSocket API specification for managing workspaces, agent tasks, and sessions.
* **[`guest-agent-protocol.md`](./04-api-and-protocols/guest-agent-protocol.md)**: Low-level `vsock` RPC protocol between the host supervisor and in-VM guest agent.

### [05. Roadmap & Implementation](./05-roadmap-and-implementation/)
* **[`milestones.md`](./05-roadmap-and-implementation/milestones.md)**: Step-by-step phases from local prototype to MVP to production cloud scale.
* **[`local-dev-guide.md`](./05-roadmap-and-implementation/local-dev-guide.md)**: Running and testing the graphical desktop, Docker sandbox, and KVM micro-VMs locally on a Mac.

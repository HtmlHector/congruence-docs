# Product Vision & Strategic Positioning

## Executive Summary

The current landscape of AI-assisted software engineering is rapidly shifting from interactive IDE completions to autonomous, long-running coding agents (e.g., Claude Code, OpenAI Codex). However, developers face a critical runtime dilemma:

1. **Local Execution Bottlenecks:** Running agents on local laptops exhausts battery, locks up CPUs, breaks when the laptop closes, and creates local git conflicts when running parallel tasks.
2. **Headless Cloud Devboxes (Boxes.dev):** Useful for offloading terminal execution, but fundamentally **blind and isolated**. They cannot inspect a graphical web browser, test UI interactions, run complex multi-service `docker-compose` stacks securely, or provide visual proof of changes.
3. **Low-Level MicroVM Infrastructure (Maritime.sh):** Powerful virtualization technology, but presented as low-level raw infrastructure without an integrated developer experience, ticketing workflows, or PR management.

**Our Mission:** Build the definitive **Cloud Workstation for AI Coding Agents**—combining a polished, developer-first workflow with a full-fledged, hardware-isolated **Cloud PC** (complete with screen, browser, and microVM) that empowers agents to write, run, and visually verify code before opening a PR.

---

## The Core Problem

When an autonomous coding agent works on a software repository, it needs to do more than edit text files. Real-world development requires:
- **Starting local databases and services** (`docker-compose up -d postgres redis`).
- **Running frontend dev servers** (`npm run dev`) and navigating to `localhost:3000`.
- **Inspecting visual UI elements** (verifying CSS, responsiveness, form submissions, and modals).
- **Taking over when stuck** (handling OAuth logins, 2FA, or CAPTCHAs).
- **Providing undeniable proof of work** so reviewers don't have to pull branches locally.

Headless containers cannot satisfy these requirements safely.

---

## Competitive Comparison

| Dimension | **Boxes.dev** | **Maritime.sh** | **Our Platform** |
|---|---|---|---|
| **Primary Abstraction** | Headless Devbox | Raw MicroVM / API | **Interactive Cloud PC** |
| **Agent Interface** | Terminal / CLI only | MCP Server / REST | **Terminal + Real Desktop GUI** |
| **Visual Verification** | ❌ None | ⚠️ Manual via VNC link | ✅ **Live browser testing & staging preview** |
| **Multi-Service Stacks** | ⚠️ Restricted (no DinD) | ✅ Full MicroVM | ✅ **Full Docker Compose support** |
| **State Forking** | ❌ Git re-cloning | ⚠️ Fast snapshot | ✅ **300ms Live RAM & Disk Forking** |
| **Disaster Recovery** | Git revert only | VM reboot | ✅ **1-Click "Undo" Time Machine** |
| **Developer Experience** | Slack, Linear, Mobile | Raw CLI, JSON flags | **Slack, Linear, VS Code, Mobile & PR UI** |

---

## Strategic Moats

1. **Dual-Engine Economics:** Multi-tenant Docker containers for free/starter tiers protect unit economics; hardware microVMs on paid tiers deliver unmatched power.
2. **Live Staging Preview URLs:** Generating instant, full-stack preview links with databases running eliminates the developer review tax.
3. **Hardware-Level RAM Forking:** Enabling multiple agents to diverge from an already-booted, warm memory snapshot cuts hours of setup time.

# Target Personas & Customer Validation

## Overview

Understanding our buyer profiles informs both our technical architecture and our product roadmap. Tier 3 is specifically optimized for developers and teams with high willingness to pay.

---

## 1. Persona A: "The Full-Stack Polymath" (Individual Pro Dev)
* **Role:** Senior Full-Stack Engineer, Freelancer, or Technical Solopreneur.
* **Tech Stack:** Next.js / TypeScript, Python / FastAPI, PostgreSQL, Redis, Tailwind.
* **Current Pain:**
  * Runs Claude Code locally. After 20 minutes, laptop fans are screaming, battery is at 20%, and they can't close the lid to go to a meeting.
  * Tries to run agents on two different features at once, causing local git conflicts and port collisions (`Port 3000 already in use`).
* **Why They Buy Tier 3:**
  * Offloads everything to the cloud.
  * Runs 3–5 parallel agents simultaneously on different branches.
  * Agent can run their full `docker-compose` backend + frontend stack.
* **Key Feature:** Parallel execution + Docker Compose + mobile PR approvals.

---

## 2. Persona B: "The Time-Strapped Tech Lead" (Startup / Growth Team)
* **Role:** VP of Engineering, Lead Architect, or Engineering Manager (teams of 5–30 engineers).
* **Current Pain:**
  * Junior engineers and AI tools submit pull requests that look syntactically correct, but break subtle CSS layouts, authentication flows, or responsive menus.
  * Reviewing PRs requires pulling the branch, running migrations, starting the server, and manually clicking around—eating 2 hours every day.
* **Why They Buy Tier 3 / Tier 4:**
  * **Automated Staging Preview:** Every AI-generated PR includes a live, interactive preview link where the app and database are already running.
  * Can click a single link to interact live with the preview instance.
  * Merges PRs in seconds with 100% confidence.

---

## 3. Persona C: "The Systems / Heavy App Engineer"
* **Role:** Backend/Infra Engineer working in Rust, Go, C++, or massive monorepos.
* **Current Pain:**
  * Standard container sandboxes (like GitHub Codespaces or Replit) choke or run out of memory (OOM) during heavy compiles.
  * Lack of `systemd` or custom kernel modules prevents testing background daemons, network tools, or low-level services.
* **Why They Buy Tier 3:**
  * Real micro-VM with dedicated kernel and 16 GB RAM.
  * Real `systemd` process supervisor.
  * Instant RAM snapshots and block-level undo when testing destructive scripts.

---

## 4. Persona D: "The Security-Conscious Enterprise"
* **Role:** Security Officer (CISO) / Enterprise DevOps Lead.
* **Current Pain:**
  * Banned cloud agent tools because container-escape exploits (e.g. CVEs in runc, Dirty COW, kernel privilege escalation) could expose proprietary enterprise IP on shared multi-tenant hosts.
* **Why They Buy Tier 4 (Enterprise):**
  * Hardware-level KVM virtualization ensures strict tenant separation.
  * MicroVMs cannot observe or communicate with adjacent tenant machines.
  * Enterprise audit logging records every shell command, keystroke, and screen frame.

# Storage, Snapshots & Time Machine

## Overview

The storage subsystem solves the fundamental conflict between speed, cost, and safety:
1. **Speed:** Micro-VMs must boot and fork in sub-second intervals.
2. **Cost:** Hundreds of gigabytes of base OS images cannot be duplicated per user.
3. **Safety:** Agents must be allowed to make destructive changes with instant 1-click rollbacks.

---

## 3-Tier Layered Storage Model

```
┌─────────────────────────────────────────────────────────────┐
│                 TIER 3: PERSISTENT WORKSPACE                │
│    Mounted at: `/workspace` or `/home/dev`                  │
│    Backed by: Local NVMe SSD or Distributed Block Storage   │
│    Survives: Sleeps, restarts, and image updates            │
├─────────────────────────────────────────────────────────────┤
│                 TIER 2: EPHEMERAL COPY-ON-WRITE (CoW)       │
│    Mounted as: OverlayFS / DeviceMapper Snapshot            │
│    Captures: Temporary OS writes, installed apt packages    │
│    Action: Instant discard, forkable, or roll-backable      │
├─────────────────────────────────────────────────────────────┤
│                 TIER 1: READ-ONLY GOLDEN BASE OS            │
│    Mounted at: Root `/`                                     │
│    Size: ~4 GB (SquashFS / RAW)                             │
│    Contents: Ubuntu LTS, Docker, XFCE, Node, Python, Chrome │
│    Shared: Exactly 1 immutable copy cached per host         │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. 300ms "RAM Forking" (Active Memory Cloning)

### The Problem
When a developer or agent wants to test 3 different architectural approaches simultaneously:
- Traditional method: Clone git repo 3 times, run `npm install` 3 times, boot 3 databases, seed mock data 3 times. Total elapsed time: 10–15 minutes.

### The Solution: Hypervisor Memory Cloning
Because Cloud-Hypervisor exposes a live snapshot API:
1. **Base State Snapshot:** When the repository is booted, the dev server is hot, and the database is seeded, the host triggers a memory pause:
   ```bash
   ch-remote snapshot --memory-path=/tmp/snap_mem --state-path=/tmp/snap_state
   ```
2. **Copy-on-Write Memory Mapping (`mmap` with `MAP_PRIVATE`):**
   - The 16 GB guest RAM file is marked Copy-on-Write using Linux kernel memory pages.
   - When 3 parallel Micro-VMs are spawned, they all read from the same underlying memory pages.
   - Only modified memory pages are written to private memory.
3. **Execution:** All 3 agent VMs start running in **~200–300 milliseconds**, already hot, already seeded, and completely isolated from each other.

---

## 2. The "Nuke & Revert" Time Machine (Block-Level Undo)

Autonomous agents frequently execute destructive shell commands (e.g., dropping database tables, deleting configuration files, or installing conflicting package versions).

### Automated Pre-Command Checkpoints
1. When the agent receives a prompt or plans to run an untested command sequence, the host daemon triggers a **lightweight CoW diff checkpoint**:
   - Freezes guest I/O for <5ms using `fsfreeze`.
   - Creates a new CoW branch on the disk overlay.
   - Logs the checkpoint ID to the session history.
2. **Instant Revert:**
   - If tests fail, files are corrupted, or the agent enters an unrecoverable state, the developer clicks **"Undo"** or the agent triggers `revert_checkpoint()`.
   - The host daemon resets the disk branch to the previous checkpoint block ID.
   - The filesystem and database state are restored in **under 100 milliseconds**.

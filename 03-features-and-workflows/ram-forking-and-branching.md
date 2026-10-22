# RAM Forking & Branching

## Overview

RAM forking lets one running, warmed-up workspace be cloned into several parallel agent workspaces in a few hundred milliseconds. Each clone starts with the same live memory, running dev server, and seeded database, so agents can try different approaches without repeating setup.

This is a Tier 3 and Tier 4 feature built on the Cloud-Hypervisor snapshot API. It is the main way the product beats the 10–15 minute setup cost of cloning, installing, and seeding by hand. The mechanism is detailed in [`storage-and-snapshots.md`](../02-system-architecture/storage-and-snapshots.md), and the endpoint is in [`control-plane-api.md`](../04-api-and-protocols/control-plane-api.md).

---

## User Flow

1. The developer (or agent) reaches a known-good state: repo cloned, dependencies installed, `docker-compose up -d` done, dev server running.
2. They trigger `/fork` in the console, or call `POST /workspaces/{id}/fork` with `count` and `names`.
3. The host pauses the VM, writes a memory and state snapshot, and resumes the parent.
4. Each child VM maps the snapshot memory copy-on-write (`MAP_PRIVATE`) and resumes within ~200–300 ms.
5. Each child is an independent workspace with its own branch name, preview URL, and agent.

Example request:

```json
POST /v1/workspaces/ws_9a8f21c0/fork
{ "count": 3, "names": ["approach-a", "approach-b", "approach-c"] }
```

Example response (202 Accepted, see control-plane docs): `latencyMs` is returned so the client can show actual fork time.

---

## Branching Model

Each fork is a **branch of the workspace**, not just a copy of memory:

| Layer | Behavior on fork |
|---|---|
| Guest RAM | Shared read-only pages, private on write |
| Disk (CoW overlay) | New overlay per child, parent's base shared |
| `/workspace` (virtio-fs) | Child gets its own working tree on a new Git branch |
| Network | New preview domain per child; no child can reach another child's ports (eBPF tenant isolation) |
| Secrets | Re-injected per child over `UpdateEnv` so each child can be given different values |

Children are siblings. Merging a child back is a Git operation (push the branch, open a PR), not a memory merge.

---

## Constraints

- **Parent must be quiescent enough to snapshot.** Long-running writes are flushed first, otherwise children may see torn disk state.
- **Memory cost scales with divergence.** Children that write a lot of memory consume host RAM quickly. Enforce the per-tier concurrent VM cap (5 for Tier 3, 15+ for Tier 4).
- **Hardware requirements.** Requires the same host and CPU model for snapshot restore. Cross-host forking is out of scope for MVP.
- **Snapshot time is not zero.** The ~300 ms figure is restore time. The initial pause to write the snapshot is additional and must be measured.

---

## Relationship to Undo

Forking and undo solve different problems:
- **Fork** creates parallel *siblings* from a point in time.
- **Undo** rolls *one* workspace back to an earlier checkpoint (see [`undo-and-time-machine.md`](./undo-and-time-machine.md)).

A checkpoint can be used as the fork point, so an agent can fork from "before the risky migration."

---

## Open Questions

- Is the ~200–300 ms restore target met on production-class NVMe with a 16 GB guest? It has not been benchmarked.
- How are child VMs billed against the parent's concurrency cap?
- Should forks inherit the parent's running agent session, or start a fresh agent session?

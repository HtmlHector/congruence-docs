# Undo & Time Machine

## Overview

Agents run commands autonomously. Some of those commands are destructive: dropping a table, deleting config, or installing conflicting packages. The Time Machine gives every Tier 3 and Tier 4 workspace automatic checkpoints before risky actions, and a one-click (or one-call) revert to any of them.

The block-level mechanism is described in [`storage-and-snapshots.md`](../02-system-architecture/storage-and-snapshots.md). This doc covers the user-facing behavior and the rules for when checkpoints are taken.

---

## Checkpoint Triggers

Checkpoints are created:

1. **Before each agent command plan that is classified as risky**, such as package installs, migrations, deletes, and writes outside the repo.
2. **On explicit request**, via `/checkpoint` in the console or a `create_checkpoint()` call.
3. **At task start**, so every run has a clean baseline.
4. **Before a fork**, so the fork point is always restorable (see [`ram-forking-and-branching.md`](./ram-forking-and-branching.md)).

Each checkpoint records its ID, timestamp, the command or plan that triggered it, and the originating agent.

---

## How a Checkpoint Is Taken

1. The host daemon freezes guest I/O with `fsfreeze` for under 5 ms.
2. A new CoW branch is created on the disk overlay.
3. The freeze is released and the checkpoint ID is written to the session history.

Because only changed blocks are tracked, checkpoints are cheap on disk and fast to create.

---

## How a Revert Works

1. The developer clicks **Undo** or runs `/revert`, or the agent calls `revert_checkpoint(id)`.
2. The host daemon resets the disk branch to the target checkpoint's block IDs.
3. Filesystem and database state are restored together.
4. The agent is notified and resumes from the checkpoint with the reverted state. The state machine has no `Failed` → `Working` transition, so a `Failed` agent must be restarted from the checkpoint as a new run (see [`agent-notifications-and-states.md`](./agent-notifications-and-states.md)).

Target revert time is under 100 ms for the disk. The database is only consistent if it is stopped or quiesced during the freeze. That requirement is an open item below.

---

## Scope by Tier

| Capability | Tier 1 | Tier 2 | Tier 3 | Tier 4 |
|---|---|---|---|---|
| Git status rollback | ✅ | ✅ | ✅ | ✅ |
| Block-level checkpoints | ❌ | ❌ | ✅ | ✅ |
| Automatic pre-command checkpoints | ❌ | ❌ | ✅ | ✅ |
| Revert time under 100 ms | ❌ | ❌ | ✅ | ✅ |

Git rollback only restores tracked repository files. It does not restore installed packages, database contents, or files outside the repo.

---

## Retention

- Checkpoints are kept for the life of the workspace.
- Old checkpoints are pruned when the workspace is destroyed.
- Retention limits per plan are not yet defined.

---

## Open Questions

- **Database consistency:** `fsfreeze` covers the filesystem, but a running Postgres may hold writes in memory. Does the checkpoint need a `pg_dump` or a quiesce hook for the database container?
- **Risk classification:** How is a "risky" command detected? A static allow/deny list, or the agent's own annotation?
- **Checkpoint storage cost:** How much disk do many small checkpoints add over a long session, and when do they get compacted?

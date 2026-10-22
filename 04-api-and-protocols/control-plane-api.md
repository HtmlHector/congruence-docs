# Control Plane API Specification

## Overview

The Control Plane exposes a unified REST and WebSocket API used by the Web Console, CLI, and integration bots to manage workspaces, agent sessions, and human takeover requests.

Base URL: `https://api.yourdomain.dev/v1`  
Authentication: `Authorization: Bearer <api_token>`

---

## Workspace Lifecycle Endpoints

### 1. Create Workspace
`POST /workspaces`

**Request Body:**
```json
{
  "name": "checkout-bugfix",
  "tier": "tier_3_microvm",
  "repository": {
    "url": "https://github.com/acme/webapp.git",
    "branch": "main",
    "targetBranch": "fix/checkout-styling"
  },
  "resources": {
    "vcpus": 4,
    "memoryMb": 8192
  },
  "enableDesktop": true,
  "env": {
    "DATABASE_URL": "postgresql://postgres:postgres@localhost:5432/testdb"
  }
}
```

**Response (201 Created):**
```json
{
  "id": "ws_9a8f21c0",
  "status": "provisioning",
  "tier": "tier_3_microvm",
  "previewUrl": "https://ws-9a8f21c0-3000.preview.yourdomain.dev",
  "createdAt": "2026-10-20T17:45:00Z"
}
```

---

### 2. Live WebRTC Stream Session
`POST /workspaces/{id}/viewer`

Generates an authenticated WebRTC SDP offer or signed viewer URL for live screen interaction.

**Request Body:**
```json
{
  "mode": "control",  // "watch" or "control"
  "ttlSeconds": 1800
}
```

**Response (200 OK):**
```json
{
  "viewerUrl": "https://app.yourdomain.dev/t/takeover_e94ba12",
  "expiresAt": "2026-10-20T18:15:00Z"
}
```

---

### 3. Trigger Snapshot / RAM Fork
`POST /workspaces/{id}/fork`

Clones an active, running Micro-VM from memory into parallel instances.

**Request Body:**
```json
{
  "count": 3,
  "names": ["approach-a", "approach-b", "approach-c"]
}
```

**Response (202 Accepted):**
```json
{
  "parentWorkspaceId": "ws_9a8f21c0",
  "forkedWorkspaces": [
    { "id": "ws_fork_1", "status": "running" },
    { "id": "ws_fork_2", "status": "running" },
    { "id": "ws_fork_3", "status": "running" }
  ],
  "latencyMs": 285
}
```

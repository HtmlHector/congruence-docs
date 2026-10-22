# Developer Integrations: VS Code, Slack, Linear & Mobile

## Overview

A great cloud workstation should feel frictionless to use. Developers should be able to trigger, monitor, and interact with agents from their existing tools: **VS Code, Cursor, Slack, Linear, and their phones**.

---

## 1. 1-Click "Open in VS Code & Cursor"

When a developer wants to inspect the code directly or work alongside the AI:
* The control plane generates an authenticated **SSH tunnel** or `code-server` bridge.
* Clicking the **"Open in VS Code"** button launches a deep-link:
  ```text
  vscode://vscode-remote/ssh-remote+agent-box-9281@gateway.yourdomain.dev/workspace
  ```
* The developer's local VS Code or Cursor client connects directly into the Micro-VM. Files, git history, and terminal access are live with zero sync delays.

---

## 2. Slack & Linear Ticket-to-PR Automation

```
[Create / Assign Linear Ticket]
"Fix responsiveness on user profile settings"
             │
             ▼ Webhook Trigger
[Control Plane Provisions Tier 3 Micro-VM]
Clones repo on branch `fix/linear-ENG-412`
Executes agent with ticket context & acceptance criteria
             │
             ▼
[Real-Time Slack Notification]
"🤖 Agent started working on ENG-412"
[View Live Desktop]  [Stream Logs]  [Abort]
             │
             ▼
[Completion & Auto-PR]
Posts PR link with live preview URL back to Linear & Slack
```

---

## 3. Mobile Companion App (iOS & Android)

Developers aren't always at their desks when an agent is running. The mobile companion app allows developers to:
1. **Review Diff Diffs on the Go:** Clean, syntax-highlighted code diffs formatted for mobile screens.
2. **Watch the Agent Live:** Stream the WebRTC desktop video directly on their phone.
3. **Approve Dangerous Commands:** Push notifications when the agent requests permission to run migrations or push branches:
   * **[View Command]** `npx prisma migrate dev`
   * **[Approve]** / **[Reject]**

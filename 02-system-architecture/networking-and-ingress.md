# Networking & Ingress Architecture

## Overview

The networking architecture serves three primary purposes:
1. **Security Isolation:** Preventing tenant-to-tenant communication and blocking access to cloud metadata services.
2. **Dynamic Ingress & Preview Domains:** Exposing services (e.g. `localhost:3000`) securely to the public internet with wildcard SSL, full WebSocket support for Hot Module Replacement (HMR), and configurable 3-mode authentication.
3. **Low-Latency Private IPC:** Connecting the host control plane to the in-VM guest agent over `virtio-vsock` without network bottlenecks.

---

## Network Topology

```
┌─────────────────────────────────────────────────────────────┐
│                       PUBLIC INTERNET                       │
│    https://<box-id>-<port>.preview.yourdomain.dev           │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTPS / WebSockets (Port 443)
┌──────────────────────────────▼──────────────────────────────┐
│                    HOST INGRESS GATEWAY (Envoy)             │
│  - Wildcard TLS Termination (*.preview.yourdomain.dev)      │
│  - Configurable 3-Mode Access Auth Gatekeeper               │
│  - Full Duplex WebSocket & HTTP/2 Proxy (Vite/Next.js HMR)  │
│  - Dynamic Session-to-Port Route Table                      │
└──────────────────────────────┬──────────────────────────────┘
                               │ Host Routing (veth / bridge)
┌──────────────────────────────▼──────────────────────────────┐
│                 HOST eBPF FIREWALL & EGRESS FILTER          │
│  - BLOCKS: 169.254.169.254 (Cloud Metadata API protection)  │
│  - BLOCKS: RFC1918 Private Ranges (Inter-VM protection)     │
│  - ALLOWS: Public Internet Outbound (GitHub, npm, APIs)     │
└──────────────────────────────┬──────────────────────────────┘
                               │ TAP Interface (tap0)
┌──────────────────────────────▼──────────────────────────────┐
│                   GUEST MICRO-VM (eth0)                     │
│  - IP: 172.20.X.2 (Assigned via static IPAM)                │
│  - Netlink Socket Watcher: Detects open ports instantly     │
│  - Running Service: Vite / Next.js dev server on 0.0.0.0:3000│
└─────────────────────────────────────────────────────────────┘
```

---

## 1. Zero-Poll Port Discovery (Linux Netlink & eBPF)

Instead of slow, wasteful port scanners probing the VM every second:
1. **Netlink Kernel Listener:** The In-VM Guest Agent binds to the Linux kernel **`NETLINK_INET_DIAG`** socket.
2. **Instant Event Trigger:** The millisecond a process executes `bind()` and `listen()` on a TCP port (e.g., Next.js binding to port 3000 or Vite on 5173), the kernel fires a netlink event directly to the Guest Agent.
3. **Registration over vsock:** The Guest Agent streams a `PortEvent(OPENED, 3000)` message over `vsock` to the Host Supervisor.
4. **Envoy Route Registration:** The Host Supervisor immediately updates the local Envoy cluster route table via dynamic EDS (Endpoint Discovery Service) in **<10 milliseconds**.

---

## 2. Configurable 3-Mode Access Control

Every dynamic preview URL (`https://<box-id>-<port>.preview.yourdomain.dev`) can be set to one of three security postures:

| Security Mode | Behavior | Ideal Use Case |
|---|---|---|
| **1. Public Unlisted** | Anyone with the cryptographically random URL can view the web app immediately without logging in. | Sharing preview links in Slack, Linear, or with external clients/QA for rapid verification. |
| **2. Team-Authenticated** | Intercepted by Envoy. Checks for a valid organization session cookie or redirects to the corporate SSO/OAuth provider (GitHub / Google / Okta). | Enterprise repositories, internal staging, and strict compliance environments. |
| **3. Password-Protected** | Envoy serves a branded password entry prompt. Upon entering the correct PIN/passphrase, an encrypted JWT cookie is set. | Sharing with external stakeholders who do not have accounts in your organization. |

---

## 3. WebSocket & HTTP/2 Proxying (Vite / Next.js HMR)

Modern developer tooling relies heavily on persistent WebSocket connections for **Hot Module Replacement (HMR)** and real-time frontend updates:
* **WebSocket Tunneling:** Envoy is configured with explicit `upgrade_configs: [{ upgrade_type: "websocket" }]`. When Vite or Next.js pushes code updates, the WebSocket connection remains open with zero disconnections.
* **Header Preservation:** The proxy transparently forwards `Host`, `X-Forwarded-For`, `X-Forwarded-Proto`, and WebSocket subprotocols (`vite-hmr`).
* **Connection Timeouts:** Configured with a 1-hour idle timeout for WebSocket streams, preventing premature drops during long debugging sessions.

---

## 4. eBPF Egress Guardrails & Cloud Security

Because AI agents can execute arbitrary commands or pull untrusted dependencies from npm/PyPI, outbound network traffic is strictly policed at the host kernel level using **eBPF (Extended Berkeley Packet Filter)**:

* **Metadata Protection:** Traffic destined for `169.254.169.254` (AWS/GCP/Azure instance metadata services) is intercepted and dropped immediately to prevent credential exfiltration.
* **Tenant Isolation:** Packets addressed to private IP ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) originating from a guest TAP interface are blocked, ensuring zero cross-VM lateral movement.
* **DDoS & Abuse Prevention:** Outbound TCP SYN floods, SMTP (port 25) traffic, and unauthorized crypto-mining pools are blocked automatically.

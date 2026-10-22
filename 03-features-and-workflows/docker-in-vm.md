# Docker-in-MicroVM Architecture

## Overview

Modern software development rarely happens in isolation. A standard web application requires background services:
- Relational Database (PostgreSQL / MySQL)
- Cache & Queue (Redis / RabbitMQ)
- Search Engine (Elasticsearch / Meilisearch)
- Mock Payment Gateway or External API stubs

In standard container sandboxes (Tiers 1 & 2), running Docker inside Docker is disabled because it requires privileged kernel access that exposes the host. **Tier 3 solves this by providing a dedicated Linux kernel per Micro-VM, allowing full, secure Docker Compose support.**

---

## Technical Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    HOST BARE-METAL SERVER                   │
│   (Protected: No privileged container access granted)       │
└──────────────────────────────┬──────────────────────────────┘
                               │ KVM Hardware Boundary
┌──────────────────────────────▼──────────────────────────────┐
│                    GUEST MICRO-VM (Own Kernel)              │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  Guest Kernel (cgroups v2 + user namespaces)        │   │
│   └──────────────────────┬──────────────────────────────┘   │
│                          │                                  │
│   ┌──────────────────────▼──────────────────────────────┐   │
│   │  Native dockerd (Storage Driver: overlay2)          │   │
│   └──────────────────────┬──────────────────────────────┘   │
│                          │                                  │
│         ┌────────────────┴────────────────┐                 │
│         │                                 │                 │
│   ┌─────▼───────────────┐           ┌─────▼─────────────┐   │
│   │ Postgres Container  │           │ Redis Container   │   │
│   │ Port: 5432          │           │ Port: 6379        │   │
│   └─────────────────────┘           └───────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. Why Docker-in-MicroVM is Completely Safe
* In a shared container environment, `docker run` inside a container requires `--privileged` mode or mounting the host's `/var/run/docker.sock`. This allows any malicious code or misbehaved AI to escape to the host node.
* In our Micro-VM architecture, the Docker daemon runs inside the **guest's private Linux kernel**.
* Even if an agent executes `docker run --privileged` or pulls a broken image inside the VM, it is physically fenced inside the KVM hardware sandbox. It has zero visibility into the physical host or other tenant machines.

---

## 2. Pre-Cached Local Images
To avoid wasting bandwidth and waiting 5 minutes for `docker pull postgres:16` or `redis:alpine` during agent runs:
* The Read-Only Golden OS Base image comes pre-loaded with local Docker cache layers for the top 50 most popular developer images:
  * `postgres`, `redis`, `mysql`, `mongo`, `elasticsearch`, `rabbitmq`, `nginx`, `localstack`.
* When the agent runs `docker-compose up -d`, containers start in **under 2 seconds** directly from local disk cache.

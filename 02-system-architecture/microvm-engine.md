# Micro-VM Engine Architecture

## Overview

The Tier 3 Micro-VM engine is the technical centerpiece of the platform. It provides every agent with a dedicated, hardware-isolated Linux virtual machine with full hardware acceleration, sub-second boot times, and deep system controllability.

---

## Hypervisor Selection: Cloud-Hypervisor vs. Firecracker

| Feature | **AWS Firecracker** | **Cloud-Hypervisor (Chosen)** |
|---|---|---|
| **Language & Safety** | Rust (`rust-vmm`) | Rust (`rust-vmm`) |
| **Primary Use Case** | Ephemeral serverless (AWS Lambda) | Modern Cloud Workloads & Desktop Sandboxes |
| **Boot Latency** | ~5ms - 50ms | ~50ms - 200ms |
| **PCI Device Support** | ❌ Minimal (virtio-mmio only) | ✅ Full `virtio-pci` support |
| **Filesystem Sharing** | ❌ virtio-block only | ✅ High-performance `virtio-fs` |
| **Memory Ballooning** | ⚠️ Basic | ✅ Dynamic live memory ballooning |
| **GPU / VFIO Pass-through**| ❌ Not supported | ✅ Supported via VFIO |
| **Snapshot / Restore** | ✅ Supported | ✅ Supported |

> **Decision:** We select **Cloud-Hypervisor** as the primary hypervisor for Tier 3 Micro-VMs. While Firecracker is faster by tens of milliseconds, Cloud-Hypervisor's native support for `virtio-fs`, `virtio-pci`, and GPU pass-through makes it vastly superior for interactive developer desktops and full Docker workloads.

---

## Virtual Hardware Topology

Each Micro-VM is spawned with an explicitly configured minimal virtual device tree:

```
[KVM Hypervisor Kernel Layer]
  ├── vCPUs: 4 - 8 vCPUs (pinned or fair-share cgroups)
  ├── Memory: 8 - 16 GB DDR4/DDR5 with virtio-ballooning
  ├── virtio-net: Dedicated TAP device bridged to Host CNI
  ├── virtio-blk: Root filesystem (Read-Only Golden Base + CoW layer)
  ├── virtio-fs: High-speed mount of `/workspace` from host NVMe
  ├── virtio-vsock: Bidirectional Host-to-Guest control channel
  └── virtio-rng: Hardware entropy device for fast cryptographic seeding
```

---

## Boot & Lifecycle Optimization

Traditional Linux VMs take 30–60 seconds to boot due to legacy BIOS initialization, PCI bus scanning, hardware probes, and bloated init systems. We achieve **sub-300ms boots** through:

1. **Direct Kernel Boot:** No BIOS or GRUB. Cloud-Hypervisor loads the uncompressed Linux kernel directly into memory and jumps execution directly to the kernel entry point.
2. **Minimal Kernel Config:** A custom-compiled Linux kernel with all legacy drivers (IDE, floppy, SCSI, sound cards, unused network protocols) stripped out. Only essential `virtio` drivers are built in (no module loading delays).
3. **Quiet Boot Parameters:** `console=ttyS0 root=/dev/vda rw quiet loglevel=0 lpj=10000000 init=/init` bypasses standard boot chatter and calibration delays.
4. **Pre-Initialized Rootfs:** The OS root filesystem is pre-seeded with all dependencies, system libraries, and pre-generated SSL certificates.

---

## Host-to-Guest IPC: The `vsock` Advantage

Instead of managing SSH keys, opening SSH ports, and suffering SSH handshake overhead, all host-guest interactions occur over **AF_VSOCK** (`virtio-vsock`).

* **Port 1024:** Guest Agent RPC (exec commands, environment variable injection, process monitoring).
* **Port 1025:** File streaming and bidirectional synchronization.
* **Port 1026:** Framebuffer streaming and input event injection.

`vsock` operates entirely in host memory with zero TCP/IP packetization overhead, providing sub-millisecond command execution latency.

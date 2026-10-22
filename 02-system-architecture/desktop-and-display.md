# Desktop & Display Subsystem

## Overview

The Desktop Subsystem gives the AI agent eyes and hands inside the Micro-VM. It renders a complete graphical desktop environment, allows programmatic screen inspection and input injection, and streams the visual session to humans with sub-100ms latency.

---

## Display Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GUEST MICRO-VM (X11/Wayland)             │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  Xvfb (Virtual Framebuffer Display :1)              │   │
│   │  Resolution: 1920x1080 @ 60Hz (24-bit TrueColor)    │   │
│   └──────────────────────┬──────────────────────────────┘   │
│                          │                                  │
│   ┌──────────────────────▼──────────────────────────────┐   │
│   │  Window Manager & Applications                      │   │
│   │  - XFCE4 / Openbox (Lightweight WM)                 │   │
│   │  - Chromium with remote devtools enabled            │   │
│   │  - Terminal / Code Editor / Devtools                │   │
│   └──────────────────────┬──────────────────────────────┘   │
│                          │                                  │
│         ┌────────────────┴────────────────┐                 │
│         │                                 │                 │
│   ┌─────▼───────────────┐           ┌─────▼─────────────┐   │
│   │ MIT-SHM Frame Grab  │           │ WebRTC Streamer   │   │
│   │ (Zero-copy PNG for  │           │ (H.264 / VP8      │   │
│   │ AI screenshot tool) │           │ hardware encode)  │   │
│   └─────────────────────┘           └─────────┬─────────┘   │
└───────────────────────────────────────────────┼─────────────┘
                                                │ WebRTC Video
                                                ▼ (Sub-100ms)
                                     ┌────────────────────────┐
                                     │ Human Web Console      │
                                     │ [Live Interactive View]│
                                     └────────────────────────┘
```

---

## 1. Zero-Latency AI Screen Inspection
When an agent calls `take_screenshot()`:
* **The Naive Way (Slow):** Running `import` or `scrot` spawns a process, writes a PNG to disk, and reads it back (200–500ms).
* **Our Optimized Pipeline (<15ms):**
  1. The Guest Agent uses the **MIT-SHM (X11 Shared Memory Extension)** to read raw pixel buffers directly from RAM.
  2. The raw BGRA buffer is compressed in-memory via SIMD-accelerated PNG/WebP encoders (`libspng`).
  3. The resulting image bytes are passed immediately to the agent via `vsock`.

---

## 2. Kernel-Level Input Injection (`/dev/uinput`)
Instead of using flaky software tools like `xdotool` (which fail on certain window managers or Wayland contexts), our Guest Agent interacts directly with the Linux kernel input subsystem:
* It creates a virtual mouse and virtual keyboard device under `/dev/uinput`.
* Absolute coordinate mouse moves (`uinput_emit_abs`), clicks, scrolls, and raw keycode presses are injected with hardware-level accuracy.
* The applications and browser believe a physical USB keyboard and mouse are connected.

---

## 3. WebRTC Live Streaming & Human Takeover

### The Protocol
* Video is captured directly from the display server and encoded via an in-memory GStreamer / Selkies pipeline into an **H.264 / VP8** video stream.
* Transported over **WebRTC** with Opus audio support.
* Round-trip latency is under **80ms**, allowing smooth 60fps interaction from any modern browser.

### The 1-Click Human Handover Flow
1. **Agent Gets Blocked:** An agent attempts an action and detects a CAPTCHA, SMS 2FA prompt, or complex OAuth wall.
2. **Takeover Request:** The agent calls `request_human_takeover({ reason: "Please complete Google 2FA" })`.
3. **Signed Link Generation:** The control plane generates an ephemeral, cryptographically signed URL (`https://app.yourdomain.dev/t/takeover_8f29c1`).
4. **Interactive Session:** The developer opens the link on their phone or laptop. Their mouse and keyboard events are routed back over the WebRTC DataChannel to `/dev/uinput`.
5. **Handback:** Once authenticated, the user clicks **"Done"**. The control plane revokes the link and signals the agent to resume execution.

---

## 4. Software-Only Graphical Emulation (No GPU / Physical Monitor Required)

A critical technical feature of this display architecture is that **no physical graphics card or monitor is required**:

1. **In-Memory Framebuffer (`Xvfb`):**
   * Acts as a purely virtual display device residing entirely in system RAM.
   * Exposes standard X11 display sockets (e.g. `:1`) that accept connections from XFCE, Chromium, and graphical Linux applications.
2. **Software Rasterization (Mesa `llvmpipe`):**
   * Standard 2D and 3D WebGL rendering is executed on the CPU using LLVM-accelerated software rasterization (`llvmpipe`).
   * This provides full support for CSS animations, HTML5 canvas, and modern web application rendering without dedicated GPU hardware.

---

## 5. Battle-Tested Precedent: Maritime.sh vs. Our Architecture

This exact virtual desktop architecture is proven in production by platforms like **Maritime.sh** (specifically their *"Maritime Computers"* product). Here is how our system maps to and upgrades their implementation:

| Architectural Component | **What Maritime Uses** | **Our Upgraded Architecture** |
|---|---|---|
| **Desktop Environment** | XFCE + Chromium | **XFCE4 / Openbox + Chromium** with automated profile seeding |
| **Display Emulation** | In-memory virtual framebuffer | **Xvfb** (1920x1080 @ 60Hz) with dynamic viewport scaling |
| **Streaming Protocol** | Standard VNC (RFB protocol over WebSocket) | **WebRTC (H.264 / VP8)** for sub-80ms streaming and smoother 60fps interaction |
| **Input Injection** | Standard VNC pointer/key events | **Kernel `/dev/uinput`** for native hardware-level USB mouse/keyboard emulation |
| **Model Resolution** | Fixed 1280x800 desktop (1200x750 model frame) | **Full HD 1920x1080** with configurable downscaling for LLM vision tokens |
| **Review Automation** | Manual viewer link generation | **Automated live preview staging URLs** linked directly into GitHub PR descriptions |
| **Human Takeover** | One-click signed web link (`/t/<id>`) | **One-click signed web link** with bi-directional Slack/Discord interactive handover |


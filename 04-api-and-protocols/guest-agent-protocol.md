# Guest Agent vsock RPC Specification (gRPC over vsock)

## Overview

The **Guest Agent** is a lightweight, statically compiled Go/Rust daemon running inside the guest Micro-VM as early PID 1 or an essential systemd service. It exposes a high-performance **gRPC service over `AF_VSOCK`** on CID `3` (Guest), Port `1024`.

By using gRPC over virtual sockets, the Host Supervisor can control the VM with sub-millisecond response times, full bidirectional streaming, and zero TCP/IP or SSH overhead.

---

## 1. Protobuf Service Definition (`guest_agent.proto`)

```protobuf
syntax = "proto3";

package guest.v1;

option go_package = "github.com/congruence/guest-agent/v1;guestv1";

// Primary control service exposed by the In-VM Guest Agent
service GuestControlService {
  // --- Terminal & Process Execution ---
  // Spawns a full interactive pseudo-terminal (PTY) with streaming stdin/stdout/stderr
  rpc ExecInteractive(stream ExecStreamRequest) returns (stream ExecStreamResponse);
  
  // Non-interactive execution for automated one-shot shell commands
  rpc ExecCommand(ExecCommandRequest) returns (ExecCommandResponse);

  // --- Display & Input Simulation ---
  // Zero-copy in-memory screenshot grab from virtual framebuffer (MIT-SHM)
  rpc CaptureScreen(CaptureScreenRequest) returns (CaptureScreenResponse);
  
  // Kernel-level hardware input injection via /dev/uinput
  rpc InjectInput(InjectInputRequest) returns (InjectInputResponse);

  // --- State, Telemetry & Heartbeat ---
  rpc GetHealth(HealthCheckRequest) returns (HealthCheckResponse);
  rpc WatchPortEvents(PortWatchRequest) returns (stream PortEvent);
  rpc UpdateEnv(UpdateEnvRequest) returns (UpdateEnvResponse);
}

// Interactive PTY Stream Request
message ExecStreamRequest {
  oneof payload {
    ExecInit init = 1;         // First message: command, args, env, initial terminal size
    bytes stdin_data = 2;      // Raw stdin byte stream
    TerminalResize resize = 3; // Window resize event (SIGWINCH)
    bool terminate = 4;        // Force SIGINT or SIGKILL
  }
}

message ExecInit {
  string command = 1;
  repeated string args = 2;
  repeated string env = 3;
  string working_dir = 4;
  uint32 rows = 5;
  uint32 cols = 6;
}

message TerminalResize {
  uint32 rows = 1;
  uint32 cols = 2;
}

// Interactive PTY Stream Response
message ExecStreamResponse {
  oneof payload {
    bytes stdout_data = 1;     // Raw stdout/stderr stream with ANSI color sequences
    int32 exit_code = 2;       // Final process exit code on completion
    string error_msg = 3;      // Daemon-level execution failure
  }
}

// Screen Capture
message CaptureScreenRequest {
  enum ImageFormat {
    PNG = 0;
    WEBP = 1;
  }
  ImageFormat format = 1;
  uint32 quality = 2; // 1-100 (WebP only)
}

message CaptureScreenResponse {
  bytes image_data = 1;
  uint32 width = 2;
  uint32 height = 3;
  uint64 frame_id = 4;
  uint32 latency_ms = 5;
}

// Input Injection via /dev/uinput
message InjectInputRequest {
  enum ActionType {
    MOUSE_MOVE = 0;
    MOUSE_DOWN = 1;
    MOUSE_UP = 2;
    MOUSE_CLICK = 3;
    MOUSE_SCROLL = 4;
    KEY_DOWN = 5;
    KEY_UP = 6;
    KEY_PRESS = 7;
    TYPE_TEXT = 8;
  }
  ActionType action = 1;
  int32 x = 2;
  int32 y = 3;
  int32 button = 4;       // 1 = Left, 2 = Middle, 3 = Right
  int32 scroll_delta = 5;
  uint32 key_code = 6;    // Linux evdev keycode (e.g., KEY_ENTER)
  string text = 7;        // Text payload for typing
}

message InjectInputResponse {
  bool success = 1;
  string error_message = 2;
}

// Dynamic Port Discovery Event
message PortWatchRequest {}

message PortEvent {
  enum EventType {
    OPENED = 0;
    CLOSED = 1;
  }
  EventType event = 1;
  uint32 port = 2;
  string protocol = 3;     // "tcp", "http", "ws"
  string process_name = 4; // e.g., "node", "vite", "python"
}

message HealthCheckRequest {}
message HealthCheckResponse {
  bool healthy = 1;
  float cpu_usage_percent = 2;
  uint64 memory_used_bytes = 3;
  uint64 memory_total_bytes = 4;
  uint64 uptime_seconds = 5;
}

message UpdateEnvRequest {
  map<string, string> env_vars = 1;
}

message UpdateEnvResponse {
  bool success = 1;
}
```

---

## 2. Interactive PTY & Terminal Resize Pipeline

When a human or an automated agent connects to the terminal:
1. **Host-to-Guest Init:** The Host Supervisor sends `ExecInit` specifying `rows: 24, cols: 80` and the command (`/bin/bash` or `claude-code`).
2. **PTY Master/Slave Allocation:**
   - The Guest Agent opens `/dev/ptmx` using standard Linux termios libraries (`creack/pty` in Go).
   - Starts the shell process attached to the slave pseudoterminal.
3. **Live Streaming:**
   - Stdin bytes from the client are written directly to the PTY master.
   - PTY master output is read in real-time and streamed as raw byte chunks over the gRPC stream. ANSI colors and curses layouts are preserved with zero corruption.
4. **Window Resizing (`SIGWINCH`):**
   - When the user resizes their browser window or terminal emulator, the client emits `TerminalResize`.
   - The Guest Agent calls `ioctl(fd, TIOCSWINSZ, &ws)`, dispatching `SIGWINCH` to the child process so interactive CLI programs (Vim, Htop, Claude Code) re-render immediately.

---

## 3. Storage Boundary: Zero-Copy `virtio-fs` vs. gRPC

* **Workspace Files (`/workspace`):** Bypasses gRPC entirely. It is directly mounted from the host's NVMe drive via **`virtio-fs`** with DAX (Direct Access) enabled. Reads and writes achieve native host SSD speeds (3,000+ MB/s).
* **Control & Secrets Injection (`.env`):** Passed over gRPC via `UpdateEnv` and written directly into guest memory or `/run/secrets/` tmpfs mounts, ensuring secrets never touch permanent disk.

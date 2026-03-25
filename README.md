# RemotePC Server

> A lightweight, high-performance Windows remote control server built with C# and Avalonia UI. Stream your screen, control your PC, transfer files, and monitor hardware — all over your local network.

[![.NET](https://img.shields.io/badge/.NET-10.0-512BD4?logo=dotnet)](https://dotnet.microsoft.com/)
[![Platform](https://img.shields.io/badge/Platform-Windows-0078D6?logo=windows)](https://www.microsoft.com/windows)
[![Avalonia](https://img.shields.io/badge/UI-Avalonia%2011.3-8B5CF6)](https://avaloniaui.net/)
[![Version](https://img.shields.io/badge/version-1.0.0.13-brightgreen)](https://github.com/Breadworks/RemotePC-CSharp/releases)

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Network Ports](#network-ports)
- [TCP Command Reference](#tcp-command-reference)
  - [Mouse Commands](#mouse-commands)
  - [Keyboard Commands](#keyboard-commands)
  - [Media Commands](#media-commands)
  - [Power & System Commands](#power--system-commands)
  - [Query Commands](#query-commands)
  - [Streaming Commands](#streaming-commands)
  - [File Transfer](#file-transfer)
- [System Stats Response Format](#system-stats-response-format)
- [UDP Protocols](#udp-protocols)
  - [Screen Streaming](#screen-streaming-udp-59452)
  - [Audio Streaming](#audio-streaming-udp-59453)
  - [Mouse Movement](#mouse-movement-udp-59454)
  - [LAN Discovery](#lan-discovery-udp-59451)
- [Security & Client Management](#security--client-management)
- [File Transfer Protocol (V2)](#file-transfer-protocol-v2)
- [UI & Settings](#ui--settings)
- [Dependencies](#dependencies)
- [Building from Source](#building-from-source)
- [CI/CD](#cicd)

---

## Overview

RemotePC Server runs as a background process on a Windows machine and exposes a multi-channel network interface for remote clients. A paired client application, available on the playstore and uptodown connects over the local network to stream the desktop, send keyboard/mouse input, control media playback, manage power, and transfer files, without any cloud infrastructure.

The server starts minimised to the system tray on subsequent launches and auto-detects VPN connections that may interfere with LAN connectivity.

---

## Features

- **Real-time screen streaming** at up to 30 FPS over UDP with per-frame chunking
- **Low-latency mouse control** via a dedicated UDP channel
- **Full keyboard emulation** including function keys, modifier keys, and Unicode input
- **Media playback control** (play, pause, next, previous) via Windows Media keys
- **System audio streaming** at 48 kHz / 16-bit stereo using WASAPI loopback capture
- **Hardware monitoring** — CPU usage & temperature, GPU temperature, RAM, battery, network I/O
- **File push** from client to server (up to 10 GB per file, up to 1000 files per transfer)
- **LAN auto-discovery** via UDP broadcast — no manual IP entry required
- **Client management** — real-time client tracking, banning, connection limits
- **VPN detection** — warns when a VPN adapter may block LAN connections
- **Auto-update** — checks GitHub Releases for new versions on startup
- **Start with Windows** — optional registry integration
- **System tray** — hides to tray after first run; double-click to restore

---

## Architecture

The server is composed of five independent network servers that run concurrently:

```
┌─────────────────────────────────────────────────────────┐
│                    RemotePC Server                      │
│                                                         │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────┐   │
│  │  TCP Server  │   │  UDP Stream  │   │ UDP Audio  │   │
│  │  Port 59450  │   │  Port 59452  │   │ Port 59453 │   │
│  │              │   │              │   │            │   │
│  │ Commands &   │   │  30 FPS JPEG │   │ 48kHz PCM  │   │
│  │ File Transfer│   │  Screen Feed │   │ Loopback   │   │
│  └──────────────┘   └──────────────┘   └────────────┘   │
│                                                         │
│  ┌──────────────┐   ┌──────────────┐                    │
│  │  UDP Mouse   │   │ UDP Discovery│                    │
│  │  Port 59454  │   │  Port 59451  │                    │
│  │              │   │              │                    │
│  │ Low-latency  │   │  Broadcast   │                    │
│  │ Cursor Input │   │  LAN Scan    │                    │
│  └──────────────┘   └──────────────┘                    │
│                                                         │
│           IntegrityAuthority (Client Manager)           │
└─────────────────────────────────────────────────────────┘
```

The **TCP Server** is the primary command channel. All text-based commands are sent here as UTF-8 newline-terminated strings. The UDP servers handle high-throughput, latency-sensitive data.

---

## Network Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| `59450` | TCP | Command channel — keyboard, mouse clicks, media, power, file transfer |
| `59451` | UDP | LAN discovery — broadcast-based server detection |
| `59452` | UDP | Screen streaming — JPEG frames at up to 30 FPS |
| `59453` | UDP | Audio streaming — PCM loopback at 48 kHz stereo |
| `59454` | UDP | Mouse movement — low-latency absolute/relative cursor positioning |

All ports must be open in your Windows Firewall for the server to be reachable on the LAN.

---

## TCP Command Reference

Connect to `<server-ip>:59450` with a plain TCP socket. Send commands as **UTF-8 strings terminated with a newline (`\n`)**. Responses (where applicable) are also newline-terminated UTF-8 strings.

Commands are **case-insensitive** unless noted otherwise.

---

### Mouse Commands

| Command | Format | Description |
|---------|--------|-------------|
| `MOUSEMOVE` | `MOUSEMOVE:dx,dy` | Move cursor by relative delta. Prefer the UDP mouse channel for continuous movement. |
| `LEFT_CLICK` | `LEFT_CLICK` | Simulate a left mouse button click (down + up) |
| `RIGHT_CLICK` | `RIGHT_CLICK` | Simulate a right mouse button click (down + up) |
| `MOUSE_DOWN` | `MOUSE_DOWN` | Press and hold the left mouse button |
| `MOUSE_UP` | `MOUSE_UP` | Release the left mouse button |
| `SCROLL_UP` | `SCROLL_UP` | Scroll wheel up (one notch, delta +120) |
| `SCROLL_DOWN` | `SCROLL_DOWN` | Scroll wheel down (one notch, delta −120) |

---

### Keyboard Commands

Special keys are sent as plain string tokens. Unicode text input uses a separate path — any single character not matching a keyword is injected as a `KEYEVENTF_UNICODE` event.

#### Navigation & Editing Keys

| Command | Key |
|---------|-----|
| `UP` | ↑ Arrow |
| `DOWN` | ↓ Arrow |
| `LEFT` | ← Arrow |
| `RIGHT` | → Arrow |
| `HOME` | Home |
| `END` | End |
| `PGUP` | Page Up |
| `PGDN` | Page Down |
| `INS` | Insert |
| `DEL` | Delete |
| `BACKSPACE` | Backspace |
| `ENTER` | Enter / Return |
| `TAB` | Tab |
| `ESC` | Escape |

#### Modifier Keys

| Command | Key |
|---------|-----|
| `CTRL` | Control |
| `SHIFT` | Shift |
| `ALT` | Alt |
| `WIN` | Windows / Super |

#### Function Keys

| Command | Key | Command | Key |
|---------|-----|---------|-----|
| `F1` | F1 | `F7` | F7 |
| `F2` | F2 | `F8` | F8 |
| `F3` | F3 | `F9` | F9 |
| `F4` | F4 | `F10` | F10 |
| `F5` | F5 | `F11` | F11 |
| `F6` | F6 | `F12` | F12 |

#### System Key Actions

| Command | Description |
|---------|-------------|
| `PRTSC` | Print Screen |
| `LOCK` | Lock the workstation (`LockWorkStation`) |
| `SLEEP` | Put the system to sleep |
| `KILLTASK` | Force-close the active foreground window |
| `TASK_MANAGER` | Launch Task Manager |

---

### Media Commands

| Command | Description |
|---------|-------------|
| `PLAY` | Send Media Play key |
| `PAUSE` | Send Media Pause key |
| `NEXT` | Send Media Next Track key |
| `PREV` | Send Media Previous Track key |

---

### Power & System Commands

| Command | Description |
|---------|-------------|
| `SHUTDOWN` | Immediately shut down the PC (`shutdown /s /t 0`) |
| `RESTART` | Immediately restart the PC (`shutdown /r /t 0`) |
| `SIGNOUT` | Sign out the current user (`shutdown /l`) |
| `LOCK` | Lock the workstation |
| `SLEEP` | Enter sleep mode |

> ⚠️ Power commands may execute immediately with no confirmation prompt.

---

### Query Commands

These commands return a response string over the same TCP connection.

| Command | Response Format | Description |
|---------|-----------------|-------------|
| `GET_CONNECTION_TEST` | `OK` | Connectivity ping |
| `GET_VOLUME` | `VOLUME:<0–100>` | Current system master volume as an integer percentage |
| `GET_MEDIA_STATE` | `MEDIA_STATE:PLAYING` or `MEDIA_STATE:PAUSED` | Whether audio is actively flowing from any WASAPI session |
| `GET_STATS` | See [System Stats](#system-stats-response-format) | Full hardware telemetry snapshot |

---

### Volume Command

| Command | Format | Description |
|---------|--------|-------------|
| `VOLUME` | `VOLUME:<0–100>` | Set master volume to the given integer percentage |

Example: `VOLUME:75` sets volume to 75%.

---

### Streaming Commands

| Command | Description |
|---------|-------------|
| `SCREENSHOT` | Capture a single frame and send it over TCP as a JPEG byte stream |
| `STOP_STREAM` | Signal the UDP Stream Server to stop sending frames and disconnect the streaming client |

For continuous streaming, use the dedicated UDP stream channel (port 59452) instead.

---

### File Transfer

| Command | Description |
|---------|-------------|
| `RECEIVE_FILES_V2` | Begin a binary file push from the client. The TCP connection switches to binary mode for the duration of the transfer. See [File Transfer Protocol (V2)](#file-transfer-protocol-v2). |

---

## System Stats Response Format

The `GET_STATS` command returns a single pipe-delimited line:

```
CPU:<usage>|RAM_USED:<bytes>|RAM_TOTAL:<bytes>|BATTERY:<percent>|CHARGING:<true|false>|NET_IN:<bytes>|NET_OUT:<bytes>|CPU_TEMP:<celsius>|GPU_TEMP:<celsius>
```

| Field | Type | Description |
|-------|------|-------------|
| `CPU` | `double` | CPU usage percentage (0.00 – 100.00) |
| `RAM_USED` | `ulong` | Bytes of physical RAM currently in use |
| `RAM_TOTAL` | `ulong` | Total physical RAM in bytes |
| `BATTERY` | `int` | Battery percentage (0–100), or `-1` if no battery |
| `CHARGING` | `bool` | `true` if AC power is connected |
| `NET_IN` | `ulong` | Cumulative bytes received across all active network adapters |
| `NET_OUT` | `ulong` | Cumulative bytes sent across all active network adapters |
| `CPU_TEMP` | `double` | CPU package temperature in °C from LibreHardwareMonitor, or `-1` if unavailable |
| `GPU_TEMP` | `double` | GPU core temperature in °C from LibreHardwareMonitor, or `-1` if unavailable |

Hardware temperature readings are sourced from **LibreHardwareMonitor** and require the server process to have sufficient privileges to read sensor data.

---

## UDP Protocols

### Screen Streaming (UDP 59452)

The screen is captured using `System.Drawing` (GDI+), JPEG-encoded, and chunked into packets with a 12-byte header.

**Packet structure:**

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 4 bytes | `frameId` | Monotonically increasing frame counter (uint32, big-endian) |
| 4 | 4 bytes | `chunkIndex` | Zero-based chunk index within this frame (uint32) |
| 8 | 4 bytes | `totalChunks` | Total number of chunks for this frame (uint32) |
| 12 | N bytes | `payload` | JPEG data slice (max 1188 bytes per packet) |

**Key parameters:**

| Parameter | Value |
|-----------|-------|
| Max packet size | 1200 bytes |
| Header size | 12 bytes |
| Max payload per packet | 1188 bytes |
| Target frame rate | 30 FPS |
| Frame interval | ~33 ms |
| Client timeout | 5000 ms (no heartbeat → stop streaming) |
| Send buffer | 1 MB |
| Receive buffer | 64 KB |

To start streaming, the client sends a `START_STREAM` datagram to port 59452. The server registers the client's endpoint and begins sending frames. A heartbeat from the client must arrive at least every 5 seconds or streaming stops.

---

### Audio Streaming (UDP 59453)

Audio is captured via **WASAPI Loopback** (`WasapiLoopbackCapture`) — it records whatever the system is playing, not microphone input.

| Parameter | Value |
|-----------|-------|
| Sample rate | 48,000 Hz |
| Channels | 2 (Stereo) |
| Bit depth | 16-bit PCM |
| Packet size | 960 samples (10 ms per packet) |
| Send buffer | 256 KB |

Raw PCM samples are sent directly in UDP datagrams. The client is responsible for buffering and playback. To receive audio, the client sends a datagram to port 59453 to register its endpoint.

---

### Mouse Movement (UDP 59454)

A dedicated UDP server handles high-frequency cursor movement for minimal latency. This is preferred over the TCP `MOUSEMOVE` command for real-time trackpad/joystick input.

The server uses `SendInput` (Win32) with `MOUSEEVENTF_ABSOLUTE` positioning or relative `MOUSE_MOVE` flags depending on the packet format. The client sends movement packets directly to port 59454.

---

### LAN Discovery (UDP 59451)

Clients can discover the server on the LAN without knowing its IP address.

**Discovery flow:**

1. Client broadcasts a discovery request packet to UDP port 59451
2. The server receives the broadcast, selects its primary non-VPN, non-virtual LAN IP, and replies with: `REMOTEPC_SERVER:<ip>:<tcp-port>`
3. The client parses the response to determine the server's address and connects on TCP port 59450

**Adapter filtering:** The discovery server actively excludes adapters matching known VPN keywords (`vpn`, `wireguard`, `proton`, `nordvpn`, etc.) and virtual adapter keywords (`vmware`, `hyper-v`, `docker`, `wsl`, etc.) to ensure the correct LAN IP is advertised.

---

## Security & Client Management

Client access is managed by the **IntegrityAuthority** singleton, which tracks all connections and persists state to disk as JSON.

### Client Tracking

Every connecting IP is recorded as a `TrackedClient`:

| Property | Description |
|----------|-------------|
| `IpAddress` | Client IP address |
| `FirstSeen` | Timestamp of first connection |
| `LastSeen` | Timestamp of most recent activity |
| `LastAction` | Most recent command received |
| `LastServer` | Which server channel handled the last action |
| `IsBanned` | Whether this client is currently blocked |

### Security Controls

| Control | Description |
|---------|-------------|
| **Ban / Unblock** | Block a specific IP from connecting. The ban is persisted to disk and survives restarts. |
| **Connection Limit** | Restrict to 1 client (exclusive mode) or allow multiple simultaneous clients. |
| **History** | Up to 10 historical connection records are retained and displayed in the UI. |

Banned clients are rejected at the TCP accept stage before any commands are processed.

---

## File Transfer Protocol (V2)

When the TCP server receives `RECEIVE_FILES_V2`, the connection enters binary mode. Up to **1000 files** can be pushed in a single transfer. Each file supports up to **10 GB**.

### Wire Format

```
[4 bytes]  File count (uint32, big-endian)

For each file:
  [4 bytes]  Filename length in bytes (uint32, big-endian)
  [N bytes]  Filename (UTF-8)
  [8 bytes]  File size in bytes (uint64, big-endian)
  [N bytes]  Raw file data

[8 bytes]  "V2_DONE\n" terminator string
```

Files are written to the server's **Downloads** folder. Safe path resolution prevents path traversal attacks — filenames with directory separators or `..` components are sanitised before writing.

---

## UI & Settings

The server uses an **Avalonia UI** window (MVVM, CommunityToolkit) with the Fluent theme. On first launch the window opens normally; on all subsequent launches it starts minimised to the system tray.

### Main Window

| Element | Description |
|---------|-------------|
| Local IP display | Shows the detected primary LAN IP address |
| Last command | Live display of the most recent command received; clears after a short delay |
| VPN warning | Banner shown when a VPN adapter is detected — warns that LAN connections may be blocked |
| Connection status | `READY FOR CONNECTIONS` (green) or `VPN DETECTED` (amber) |
| Connected clients list | Real-time list of active clients with last seen time, last action, and a Ban button |
| Connection history | Up to 10 past connections |
| Connection limit | Toggle between exclusive (1 client) and multi-client mode |
| Start with Windows | Writes/removes a `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` registry entry |

### System Tray

The server minimises to the notification area on close. Double-click the tray icon to restore the window. Right-click for a context menu with Quit.

### Auto-Update

On startup, the server queries the **GitHub Releases API** for the configured repository. If a newer tag is found, a notification is shown with a direct download link to the `.exe` installer asset (if one is attached to the release), or a link to the release page as a fallback.

---

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| [Avalonia](https://avaloniaui.net/) | 11.3.11 | Cross-platform UI framework (Windows target) |
| [Avalonia.Desktop](https://avaloniaui.net/) | 11.3.11 | Desktop-specific Avalonia extensions |
| [Avalonia.Themes.Fluent](https://avaloniaui.net/) | 11.3.11 | Fluent design theme |
| [Avalonia.Fonts.Inter](https://avaloniaui.net/) | 11.3.11 | Inter font family |
| [CommunityToolkit.Mvvm](https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/) | 8.2.1 | MVVM source generators and base classes |
| [LibreHardwareMonitorLib](https://github.com/LibreHardwareMonitor/LibreHardwareMonitor) | 0.9.5 | CPU / GPU temperature and sensor data |
| [NAudio](https://github.com/naudio/NAudio) | 2.2.1 | WASAPI loopback audio capture and streaming |

### Runtime Requirements

| Requirement | Detail |
|-------------|--------|
| OS | Windows 10 (build 19041 / version 2004) or later |
| .NET | .NET 10.0 (Windows) |
| Architecture | x64 recommended |
| Privileges | Elevated privileges required |

---

## License

Copyright © 2026 Breadworks. All rights reserved.

---

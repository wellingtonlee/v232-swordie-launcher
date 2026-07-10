# v232 Swordie Launcher

![Status](https://img.shields.io/badge/status-as--is-red)
![Maintenance](https://img.shields.io/badge/maintenance-none-lightgrey)
![MapleStory](https://img.shields.io/badge/maplestory-v232-blue)
![.NET](https://img.shields.io/badge/.NET%20Framework-4.8-512BD4)
![UI](https://img.shields.io/badge/UI-WPF-2b579a)
![Platform](https://img.shields.io/badge/platform-Windows%20x64-0078D6)

A lightweight, custom-skinned game launcher for a **MapleStory v232** private server built
on the **Swordie v232** public release. It handles account login and registration,
authenticates against a private backend, optionally verifies client files, and launches
the game client.

Released for **reference and personal use** by private-server operators.

<img width="1275" alt="v232 Swordie Launcher screenshot" src="https://github.com/user-attachments/assets/ea805de0-870e-4fb6-ad91-877309a940bc" />

> **Naming note:** the repository and built executable are named **"v232 Launcher"**, while
> the in-app title and settings folder are branded **"RoyalStory Launcher"**. They refer to
> the same application.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [How It Works](#how-it-works)
- [Configuration](#configuration)
- [Building](#building)
- [Usage & Setup](#usage--setup)
- [Disclaimer](#disclaimer)
- [Project Status](#project-status)
- [Credits](#credits)
- [Links & Resources](#links--resources)
- [Final Note](#final-note)

---

## Overview

This is a Windows desktop launcher (WPF / .NET Framework 4.8) that sits in front of a
MapleStory v232 client. It presents a custom login/registration UI, talks to a private
authentication backend over TCP, and — on success — starts the game client and applies the
server's auth hook.

It is intended as a **base or reference** for people running their own private servers. You
are expected to adapt the configuration, supply the required external files, and modify the
code to fit your own setup.

---

## Features

**Account**
- Login and registration directly from the launcher
- "Remember Me" username, persisted between sessions
- Client-side validation for usernames, passwords, and email (with a basic SQL-injection filter)
- Compatible with the Swordie v232 `users` table by default

**Interface**
- Two switchable skins — **Classic** and **Neon** — with the choice saved between runs
- Animated GIF backgrounds
- Looping background music with a play/pause toggle and volume control
- Live online/offline server status indicator
- Borderless custom window with in-app Discord and Credits popups

**Backend**
- Raw TCP authentication client with length-framed packet communication
- Structured request/response packet protocol
- Backend-controlled launch approval via a returned auth token

**Client**
- Optional client / WZ file integrity verification
- Launches the MapleStory client after successful authentication and applies the auth hook

---

## Tech Stack

| Area      | Technology                                             |
| --------- | ------------------------------------------------------ |
| Language  | C#                                                      |
| Framework | .NET Framework 4.8 (WPF, `WinExe`, x64)                |
| UI        | XAML + [WpfAnimatedGif](https://www.nuget.org/packages/WpfAnimatedGif) 2.0.2 (vendored) |
| Transport | Raw `System.Net.Sockets` TCP with custom packet framing |
| Build     | MSBuild / Visual Studio 2022, GitHub Actions (Windows) |

There are no automated tests.

---

## Project Structure

```
v232-swordie-launcher/
├─ App.xaml / App.xaml.cs        # WPF entry point; loads the default theme
├─ MainWindow.xaml / .xaml.cs    # Main window + all UI logic (login, register, themes, BGM)
├─ Models/
│  ├─ Configs.cs                 # Server IP/port, tokens, LocalLogin toggle
│  ├─ OutPacket.cs               # Outgoing packet writer (little-endian)
│  └─ InPacket.cs                # Incoming packet reader
├─ Services/
│  ├─ Client.cs                  # TCP socket client + length-prefixed framing
│  ├─ OutPackets.cs              # Packet opcodes / builders
│  ├─ Handlers.cs                # Auth + account-create request handling
│  ├─ LoginService.cs            # Authentication, WZ checks, launch + DLL injection
│  └─ RegisterService.cs         # Sign-up validation
├─ Themes/
│  ├─ DarkTheme.xaml             # Classic skin
│  └─ NeonTheme.xaml             # Neon skin
├─ Assets/                       # Slime.ico, background GIFs, background music
├─ .github/workflows/build.yml   # Windows CI build + optional Release
└─ v232.Launcher.WPF.csproj/.sln
```

---

## How It Works

1. **Connect.** On startup the launcher opens a TCP connection to the configured host and
   API port (`Services/Client.cs`) and updates the online/offline status dot.
2. **Authenticate.** Login sends an auth request packet; the backend replies with a result
   code and, on success, an **auth token** and account type (`Services/Handlers.cs`).
   Registration follows the same request/response pattern.
3. **Launch.** With a valid token, the launcher starts the game with
   `CreateProcess("MapleStory.exe", " WebStart <token>", CREATE_SUSPENDED)`, injects the
   auth-hook DLL (`Localhost.dll`) into the suspended process using the standard Win32
   sequence (`OpenProcess` → `VirtualAllocEx` → `WriteProcessMemory` → `CreateRemoteThread`
   + `LoadLibraryA`), then resumes the process (`Services/LoginService.cs`).
4. **Optional WZ verification.** When running in non-local mode (`LocalLogin = false`), the
   launcher checks that exactly **26 `.wz` files** are present, MD5-hashes each (skipping a
   built-in whitelist), and verifies them against the server via checksum packets.

**Packet protocol.** Messages use a 4-byte big-endian length header. Opcodes are defined in
`Services/OutPackets.cs`:

| Opcode | Request       |
| ------ | ------------- |
| 100    | Auth request  |
| 101    | Create account |
| 10002  | File checksum |
| 11000  | Heartbeat     |

**External files you must supply** (not included in this repo): `MapleStory.exe`, the 26
`.wz` client files, and `Localhost.dll` (the injected auth hook), plus your authhook /
bypasses.

---

## Configuration

All server settings live in [`Models/Configs.cs`](Models/Configs.cs). The IP fields are
**Base64-encoded**, and `GetServerIP()` decodes whichever one applies based on `LocalLogin`.

| Field           | Type     | Purpose                                                                 |
| --------------- | -------- | ----------------------------------------------------------------------- |
| `LocalLogin`    | `bool`   | `true` uses `LocalIP` and skips the updater/WZ check; `false` uses `ServerIP` and enables WZ verification. |
| `LocalIP`       | `string` | Base64-encoded IP used when `LocalLogin = true`.                        |
| `ServerIP`      | `string` | Base64-encoded IP used when `LocalLogin = false`.                       |
| `WebServerToken`| `string` | Base64-encoded token for the updater/web server.                        |
| `APIServerPort` | `int`    | TCP port for the authentication backend (default `8483`).               |
| `WebServerPort` | `int`    | Port for the web/updater server (default `80`).                         |

Encode and decode the IP values with:

```csharp
// Encode:
Convert.ToBase64String(Encoding.UTF8.GetBytes("127.0.0.1"));
// Decode:
Encoding.UTF8.GetString(Convert.FromBase64String(encoded));
```

> The IP values shipped in `Configs.cs` are placeholders — replace them with your own
> server's Base64-encoded address before building.

**Runtime settings** (theme, remember-me username, and BGM state) are saved per user under
`%AppData%\RoyalStoryLauncher\` as `theme.cfg`, `user.cfg`, and `bgm.cfg`.

---

## Building

This is a **.NET Framework 4.8 WPF** app, so the `.exe` can only be built on **Windows**.
You don't need a local Windows machine, though — a GitHub Actions workflow builds it for you
on a Windows runner.

**No Windows box (macOS/Linux):** trigger the CI build and download the artifact.

```bash
gh workflow run build.yml   # manual build; uploads the .exe as an artifact
gh run watch                # wait for it to finish
gh run download             # grab v232-launcher-win-x64 (the .exe + WpfAnimatedGif.dll)
```

Pushing a version tag (`git tag v232.x && git push origin v232.x`) builds **and** publishes
a GitHub Release with the `.exe` attached. See
[`.github/workflows/build.yml`](.github/workflows/build.yml). The workflow is manual-only
(`workflow_dispatch` + `v*` tag push); it does not run on ordinary pushes or PRs.

**On Windows:** open the solution in Visual Studio 2022, or from a Developer prompt run:

```powershell
nuget restore v232.Launcher.WPF.sln
msbuild v232.Launcher.WPF.csproj /p:Configuration=Release /t:Rebuild
# -> bin\Release\v232 Launcher.exe  (copy it + WpfAnimatedGif.dll to your game folder)
```

> A Linux Docker container **cannot** build this (WPF is Windows-only), and Docker Desktop
> on Apple Silicon can't run Windows containers — hence the CI route.

---

## Usage & Setup

1. Prepare your **authhook / bypasses** and the injected `Localhost.dll` for your server.
2. Set your server's **Base64-encoded IP** (and ports/token if needed) in
   [`Models/Configs.cs`](Models/Configs.cs), then compile.
3. Copy **`v232 Launcher.exe`** and **`WpfAnimatedGif.dll`** into your MapleStory game folder.
4. Run the launcher **as Administrator** (required for DLL injection).
5. If launching fails at the WZ step, server-side WZ checks may need to be disabled, or run
   with `LocalLogin = true` to skip verification.

---

## Disclaimer

This launcher is provided for **educational and reference purposes**, intended for use with
**private servers that you own or are authorized to operate**. It performs DLL injection into
the game client and must run with administrator privileges. Do **not** use it against
official, live, or third-party services you are not authorized to access.

A few honest caveats to be aware of if you build on this:

- The server IP is only **Base64-encoded, not encrypted** — treat it as obfuscation, not security.
- Credentials are currently sent to the backend in **plaintext** framed packets.
- You are responsible for how you deploy and use this software.

---

## Project Status

> This project is **not actively maintained** and is provided **as-is**, with **no guarantees**.

- No support or troubleshooting
- No feature requests
- Issues and pull requests may go unanswered
- Expect to modify and fix things yourself

If you're not comfortable adapting the code on your own, this repository may not be for you.

---

## Credits

**Backend**
- Level
- NutNNut

**Launcher Design**
- Lynx

---

## Links & Resources

- [Authhook & Launcher — RageZone thread](https://forum.ragezone.com/threads/v232-swordie-launcher.1258180)

---

## Final Note

Forking and modifying this project is completely fine.

Please do **not** rebrand or sell this as your own work — it was released to give back to the
community, not to profit from.

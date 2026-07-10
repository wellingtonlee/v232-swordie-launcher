# v232-Swordie-Launcher

![Status](https://img.shields.io/badge/status-as--is-red)
![Maintenance](https://img.shields.io/badge/maintenance-none-lightgrey)
![Version](https://img.shields.io/badge/maplestory-v232-blue)

Lightweight custom launcher built on the base of the **Swordie v232** public release.

Handles account actions, basic client verification, and launching the game through a custom backend.

Released for **reference and personal use**.

---

## ✨ Features

**Account**
- Login & registration directly from the launcher
- Compatible with Swordie v232 `users` table by default

**Backend**
- TCP-based authentication backend
- Structured packet communication
- Backend-controlled launch approval

**Client**
- Optional client / WZ verification
- Launches the MapleStory client after successful authentication

---

## 🚀 How to Use
- Use Authhook and bypasses
- Add your Base64 encoded IP Address into Configs.cs, Compile
- Move the v232 Launcher.exe and WpfAnimatedGif.dll to your game folder, Run it as Admin
- Wz Checks server side may need to be disabled in order for it to work
---

## ⚠️ Project Status

> This project is **not actively maintained**.  
> Provided **as-is**, with **no guarantees**.

- No support or troubleshooting  
- No feature requests  
- Issues and pull requests may be ignored  
- Expect to modify things yourself  

If you don’t know how to fix things on your own, this repo is **not for you**.

---

## 🧩 Usage

- Intended for private servers and personal projects  
- Meant as a base or reference  
- You are expected to adapt it to your own setup  

---

## 🛠️ Building

This is a **.NET Framework 4.8 WPF** app, so the `.exe` can only be built on
Windows. You don't need a local Windows machine, though — a GitHub Actions
workflow builds it for you on a Windows runner.

**No Windows box (macOS/Linux):** trigger the CI build and download the artifact.

```bash
gh workflow run build.yml   # manual build; uploads the .exe as an artifact
gh run watch                # wait for it to finish
gh run download             # grab v232-launcher-win-x64 (the .exe + WpfAnimatedGif.dll)
```

Pushing a version tag (`git tag v232.x && git push origin v232.x`) builds **and**
publishes a GitHub Release with the `.exe` attached. See
`.github/workflows/build.yml`.

**On Windows:** open the solution in Visual Studio 2022, or from a Developer
prompt run:

```powershell
nuget restore v232.Launcher.WPF.sln
msbuild v232.Launcher.WPF.csproj /p:Configuration=Release /t:Rebuild
# -> bin\Release\v232 Launcher.exe  (copy it + WpfAnimatedGif.dll to your game folder)
```

> Note: a Linux Docker container **cannot** build this (WPF is Windows-only), and
> Docker Desktop on Apple Silicon can't run Windows containers — hence the CI route.

---

## 🧾 Credits

**Backend**
- Level  
- NutNNut  

**Launcher Design**
- Lynx  

---

## 🔗 Links & Resources

- [Authhook & Launcher](https://forum.ragezone.com/threads/v232-swordie-launcher.1258180)

---


## 📌 Final Note

Forking and modifying this project is completely fine.

Do **not** rebrand or sell this as your own work.  
This was released to give back, not to make a dollar off it.

---

## Images

<img width="1275" height="715" alt="Screenshot_2" src="https://github.com/user-attachments/assets/ea805de0-870e-4fb6-ad91-877309a940bc" />


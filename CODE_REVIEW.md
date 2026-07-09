# Peer Review — v232 Swordie Launcher

A defensive code review of this .NET Framework 4.8 WPF launcher for a MapleStory v232
(Swordie) private server. The app handles login/registration over a raw TCP socket, verifies
`.wz` client files, launches `MapleStory.exe` suspended, and DLL-injects `Localhost.dll`.

Findings are grouped by severity. Line references reflect the state of the repo at review time.

---

## 🔴 Critical — committed secrets & credentials

1. **Hardcoded credential pair — `Models/Configs.cs:18`.**
   `WebServerToken = "djIxNF9VcGRhdGVyOk1hcGxldjIxNFVwZGF0ZXI3MTYhQA=="` base64-decodes to
   `v214_Updater:Maplev214Updater716!@` — a `user:password`. It is in source **and in git
   history**, so removing the file is not enough; the credential must be **rotated on the
   server**. Base64 is encoding, not encryption.

2. **Server IP leaked + misleading comments — `Configs.cs:15-16`.**
   Both `LocalIP` and `ServerIP` decode to `26.3.188.61` (a Radmin-VPN range), while the
   comments claim `127.0.0.1` and `192.168.50.10`. The comments are wrong and the real backend
   IP is exposed. The value is also baked into the committed `.exe`.

3. **Compiled binaries committed.**
   `obj/Debug/v232 Launcher.exe`, its `.pdb`, stale `v214.Launcher` build artifacts, and the
   entire NuGet `packages/` tree (incl. `wpfanimatedgif.zip`) are tracked. The published
   `.exe`/`.pdb` embed the same secrets and IP and are downloadable by anyone.

---

## 🟠 High — the .wz integrity check is effectively non-functional

Two independent bugs in `LoginService.LaunchMaple`, either of which alone defeats it:

1. **`LoginService.cs:217`** — `Parallel.ForEach(wz_files, async wz_file => …)`.
   `Parallel.ForEach` takes an `Action`, so the async lambda compiles to **`async void`** and is
   never awaited. The loop returns before any `await GetFileChecksum` completes, so the
   `passChecksum` check at line 256 races ahead of the actual verification.

2. **`LoginService.cs:239`** — `checkFileChecksum.Equals(1)` compares a **`byte`** to a boxed
   **`int`** literal via `Byte.Equals(object)`, which is **always `false`**. The failure branch
   never fires. Should be `checkFileChecksum == 1`.

(Also note: default `Configs.LocalLogin = true` skips this whole block anyway.)

---

## 🟠 High — networking is unsafe and fragile

4. **Credentials sent in plaintext over an unencrypted TCP socket.**
   `OutPackets.AuthRequest` / `CreateAccountRequest` write username/password as raw ASCII
   (`OutPacket.WriteString`). No TLS, no hashing → sniffable / MITM-able. Note the unused
   `Handlers.Sha256Hex` (`Handlers.cs:85`) — hashing was apparently intended but never wired up.

5. **Concurrent socket use corrupts the protocol.**
   `Client` uses **static** `ManualResetEvent`s (`connectDone/sendDone/receiveDone`) shared across
   all instances, and `Parallel.ForEach` fires many concurrent `Send`/`Receive` calls on one
   socket. Interleaved sends/receives on a single stream socket with shared signaling → garbled
   frames.

6. **Deadlock / buffer-overflow risks in `Client`.**
   - `Send` calls `sendDone.WaitOne()` with **no timeout**, and `SendCallback` only `Set()`s on
     success — if `EndSend` throws, the caller blocks forever.
   - `sendDone` is never `Reset()` before a send, so a leftover set state lets `WaitOne` return
     prematurely.
   - `Receive`/`ReceiveCallback` write into a fixed **256-byte** `InPacket.bufData`; a server
     packet >256 bytes overflows the array. The wire length is a full 32-bit value, trusted with
     no bounds check.

7. **Client-side SQL-injection blocklist is security theater — `RegisterService.IsSafeFromSqlInjection`.**
   Keyword/`;`/comment filtering is trivially bypassable (patch the client) and simultaneously
   rejects legitimate input. The real defense is server-side parameterized queries.

8. **DLL injection has no integrity check on the injected DLL — `LoginService.Inject`.**
   Inherent to the launcher's purpose, but worth stating: `CreateRemoteThread` + `LoadLibraryA` is
   the textbook injection pattern (trips AV/EDR, needs admin), and it injects whatever
   `Localhost.dll` sits next to the exe — ironic given `.wz` files are checksummed but the DLL is not.

---

## 🟡 Medium — robustness & quality

9.  **Auth token logged to stdout** — `Handlers.cs:27` (`Console.WriteLine("Token: " + token …)`);
    `InPacket.readString` also logs lengths. Leaks the session token if run from a console.
10. **Silent failure everywhere** — dozens of empty `catch { }` blocks (config load/save, connect,
    BGM, checksum loop) swallow all errors, making field diagnosis very hard.
11. **`GC.Collect()` + `WaitForPendingFinalizers()` per `.wz` file** (`LoginService.cs:251`) —
    needless and slow; the `using` blocks already dispose the streams.
12. **Validation mismatches its messages** — `IsValidPassword` only checks the allowed char set, not
    that letters+numbers+specials are all present as the error text claims. Email regex `{2,4}`
    rejects valid TLDs (`.info`, `.online`).
13. **No `.gitignore`** — `obj/`, build output, and `packages/` should be ignored, not tracked.

---

## ⚪ Low — hygiene / consistency

14. Inconsistent branding (copy-paste lineage): project `v232`, token `v214_Updater`, config folder
    `RoyalStoryLauncher`, README says `Swordie`.
15. `GetFileChecksum` is declared `async` but never awaits anything.
16. Startup connection (`InitializeConnection`) never retries; if the server is down at launch the UI
    stays "Offline" with no reconnect path.

---

## Priorities

1. **Rotate the `v214_Updater` credential on the server** — it is compromised (git history + shipped
   `.exe`), independent of any repo cleanup.
2. **Fix the two `.wz`-check bugs** — today the integrity check never rejects a tampered file.
3. **Harden the `Client` socket layer** — per-instance signaling, send timeouts, and bounds-checked
   lengths.

Server-side items (TLS on the auth channel, parameterized queries, secret rotation) live outside this
repo but are required for the client-side fixes to mean anything.

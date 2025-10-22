# WaveBreaker - Web & Desktop Builds

WaveBreaker is a Raylib-powered top-down shooter that targets both native Windows and the browser (via Emscripten). This guide covers building, serving, and the current configuration behaviour (built-in defaults, no runtime fetch).

---

## Requirements
- Raylib sources at `C:\raylib\raylib`
- Raylib w64devkit toolchain (`C:\raylib\w64devkit\bin`)
- Emscripten SDK installed at `C:\Users\bora2\emsdk`

---

## Build (Desktop)
```powershell
Set-Location "C:\Users\bora2\OneDrive\Desktop\GameDev Projects\Projects\Raylib_TopsownShooter"
& "C:\raylib\w64devkit\bin\mingw32-make.exe" RAYLIB_PATH="C:/raylib/raylib" PROJECT_NAME=main BUILD_MODE=DEBUG
```
The executable is emitted as `main.exe` in the project root.

---

## Build (Web with emcc)
1. **Activate Emscripten**
   ```powershell
   & "C:\Users\bora2\emsdk\emsdk_env.ps1"
   ```

2. **Rebuild Raylib for the Web target** (ensures `libraylib.a` is wasm compatible):
   ```powershell
   Set-Location "C:\raylib\raylib\src"
   & "C:\raylib\w64devkit\bin\mingw32-make.exe" PLATFORM=PLATFORM_WEB
   ```

3. **Compile the game** (produces HTML + WASM bundle and overrides any existing `index.*` files):
   ```powershell
   Set-Location "C:\Users\bora2\OneDrive\Desktop\GameDev Projects\Projects\Raylib_TopsownShooter"
   & "C:\raylib\w64devkit\bin\mingw32-make.exe" PLATFORM=PLATFORM_WEB `
       RAYLIB_PATH="C:/raylib/raylib" BUILD_MODE=RELEASE
   ```

Artifacts land in the project root as `index.html`, `index.js`, `index.wasm`, and `index.data`. To start from a clean slate before rebuilding:
```powershell
& "C:\raylib\w64devkit\bin\mingw32-make.exe" clean PLATFORM=PLATFORM_WEB
```

---

## Serve & Play in a Browser
Use any static file server; Python is convenient:
```powershell
Set-Location "C:\Users\bora2\OneDrive\Desktop\GameDev Projects\Projects\Raylib_TopsownShooter"
python -m http.server 8000
```
Open <http://localhost:8000/index.html> in a modern browser.

---

## Runtime Configuration
At startup the web build performs an asynchronous daily-seed fetch that can gently tune gameplay variables (enemy counts, drop rate, power-up timer window, starting wave, and a menu MOTD). Native desktop builds continue to use the built-in defaults.

### Daily Seed Web Request
- **Endpoint**: `https://worldtimeapi.org/api/timezone/Etc/UTC` (public WorldTimeAPI).
- **What we read**: The response JSON contains either `unixtime` or (if missing) we fall back to the local clock via `Date.now()`. Both values represent seconds since the Unix epoch.
- **How it derives the seed**: The JavaScript helper `FetchDailySeed()` divides the epoch seconds by 86,400 to get "days since epoch", producing a stable integer that only changes once per UTC day. That integer is passed into `_ApplyDailySeed` (the C++ `ApplyDailySeed` function exported with `EMSCRIPTEN_KEEPALIVE`).
- **Timeout & fallback**: An `AbortController` cancels the fetch after 1.5 s to avoid stalling frames. On fetch failure or timeout, it computes the day locally via `Math.floor(Date.now() / 86400000)` so offline play still gets a deterministic seed.
- **What changes in-game**: `ApplyDailySeed` clamps and applies the day-based modifiers to the global variables (`enemyCountMultiplier`, `powerUpSpawnIntervalMin/Max`, `enemyDropChance`, `startingWaveOverride`) and formats the MOTD (`g_MOTD`) so the menu visibly reflects the remote config.

---

## Extras
- Animated preview: `Wavebreaker.gif`
- itch.io page: <https://bora0dev.itch.io/wavebreaker>

Bundle the `index.*` files and `index.data` when deploying to itch.io or another static host.

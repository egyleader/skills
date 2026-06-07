---
name: android-emulator-optimize
description: >
  Optimize Android emulator performance on any developer machine. Trigger on:
  "emulator is slow/laggy", "process not responding", "optimize my emulator",
  "set up Android AVD", "emulator using too much RAM", "android emulator freezing",
  or any request to configure/tune/create an Android virtual device. Audits host
  specs dynamically and produces a lean, hardware-accelerated AVD with correct
  config.ini settings, shell PATH setup, and a launch alias.
---

# Android Emulator Optimizer

Audit the host → select a lean image → create AVD → tune config.ini → wire shell → create alias.
All values (RAM, API, arch) are computed from actual host specs, not hardcoded.

---

## Phase 1 — Audit host specs

Run in parallel:

```bash
# Total host RAM in MB
sysctl hw.memsize | awk '{printf "%.0f\n", $2/1024/1024}'

# CPU architecture (arm64 = Apple Silicon)
uname -m

# Locate SDK root
echo "${ANDROID_SDK_ROOT:-${ANDROID_HOME:-$HOME/Library/Android/sdk}}"

# What system images are already installed?
ls "${ANDROID_SDK_ROOT:-${ANDROID_HOME:-$HOME/Library/Android/sdk}}/system-images/" 2>/dev/null

# Existing AVDs (avoid name collisions)
avdmanager list avd 2>/dev/null | grep "Name:"
```

---

## Phase 2 — Select image and device

### Architecture
| Host | Image ABI |
|---|---|
| Apple Silicon (`arm64`) | `arm64-v8a` |
| Intel Mac / Linux / Windows | `x86_64` |

### API version
List what's available then pick:
```bash
sdkmanager --list 2>/dev/null | grep "system-images;android-3[0-9];google_apis;$(uname -m | sed 's/x86_64/x86_64/;s/arm64/arm64-v8a/')"
```
- **Pick**: highest stable API (`google_apis` variant only)
- **Currently ideal**: `android-34` → fallback `android-35`
- **Reject**: `ps16k`, `-ext*`, `_atd`, `android-wear`, `android-tv`, any pre-release or `android-37.0`

### Device
- **Pick**: real Pixel device 2–3 generations behind current flagship
- **Default**: `pixel_4a` — 1080×2340, no foldable overhead, well-tested
- **Avoid**: Resizable, Experimental, Foldable, Tablet, TV, Automotive — all carry background overhead irrelevant to app dev
- Confirm: `avdmanager list device 2>/dev/null | grep -A2 "pixel_4a"`

### RAM allocation
`emulator_ram_mb = min(host_ram_mb / 4, 4096)` — never exceed 50% of host RAM.  
Exceeding this causes the host OS to swap, which defeats the purpose.

| Host RAM | Emulator RAM |
|---|---|
| 8 GB | 2048 MB |
| 16 GB | 4096 MB |
| 32 GB+ | 4096 MB (cap) |

---

## Phase 3 — Ensure Java and install image

On macOS, `sdkmanager` needs the Android Studio bundled JDK if system Java is absent:

```bash
export JAVA_HOME="/Applications/Android Studio.app/Contents/jbr/Contents/Home"
export ANDROID_SDK_ROOT="${ANDROID_SDK_ROOT:-$HOME/Library/Android/sdk}"
export PATH="$JAVA_HOME/bin:$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$PATH"

# Verify before installing
java -version 2>&1 | head -1

# Install (skip if already in system-images dir)
echo "y" | sdkmanager "system-images;android-34;google_apis;arm64-v8a"
```

On Linux/Windows, rely on system Java (`java -version`). No `JAVA_HOME` override needed.

---

## Phase 4 — Create AVD

```bash
avdmanager create avd \
  --name "<DeviceModel>_API<level>" \
  --device "<device_id>" \
  --package "system-images;android-<api>;google_apis;<abi>" \
  --force
```

Name convention: `Pixel4a_API34`. The `--force` flag overwrites if it already exists.

---

## Phase 5 — Tune config.ini

Read `~/.android/avd/<AVD_NAME>.avd/config.ini` then apply:

| Key | Value | Why |
|---|---|---|
| `hw.gpu.enabled` | `yes` | Enable GPU path |
| `hw.gpu.mode` | `host` | Uses host Metal/Vulkan — must be explicit, not `auto` |
| `hw.ramSize` | `<computed>` | From Phase 2 RAM rule |
| `vm.heapSize` | `512` | Default 228 causes frequent GC pauses |
| `hw.cpu.ncore` | `4` | Parallel rendering threads |
| `showDeviceFrame` | `no` | Removes pixel-perfect skin rendering |
| `hw.camera.back` | `none` | Virtualscene renderer burns GPU cycles; re-enable if app uses camera |
| `hw.audioInput` | `no` | Fewer background threads |
| `hw.audioOutput` | `no` | Same |
| `hw.keyboard` | `yes` | avdmanager defaults this to `no` for real-device profiles — enables host keyboard |
| `hw.gltransport.drawFlushInterval` | `400` | Default 800 causes visible frame latency |
| `fastboot.forceColdBoot` | `yes` | Prevents failed snapshot load warnings |
| `fastboot.forceFastBoot` | `no` | |
| `firstboot.bootFromDownloadableSnapshot` | `no` | Snapshot saving is incompatible with `hw.gpu.mode=host` |
| `firstboot.bootFromLocalSnapshot` | `no` | |
| `firstboot.saveToLocalSnapshot` | `no` | |

---

## Phase 6 — Shell PATH setup

Check `~/.zshrc` (macOS/zsh) or `~/.bashrc` (Linux) for existing Android entries. Add what's missing:

```bash
# Android SDK
export ANDROID_HOME=$HOME/Library/Android/sdk        # macOS; Linux: $HOME/Android/Sdk
export ANDROID_SDK_ROOT=$ANDROID_HOME
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin  # sdkmanager, avdmanager
export PATH=$PATH:$ANDROID_HOME/platform-tools             # adb
export PATH=$PATH:$ANDROID_HOME/emulator                   # emulator binary

# macOS only — Android Studio bundled JDK
export JAVA_HOME="/Applications/Android Studio.app/Contents/jbr/Contents/Home"
export PATH=$PATH:$JAVA_HOME/bin
```

`cmdline-tools/latest/bin` is the one most often missing — it's where `sdkmanager` and `avdmanager` live.

---

## Phase 7 — Launch alias

```bash
alias emltr='emulator -avd <AVD_NAME> -gpu host -no-audio -no-boot-anim &'
```

The `&` returns the terminal immediately. `-no-boot-anim` cuts cold boot time. All persistent performance settings live in config.ini — the alias stays simple.

---

## Cleanup (when replacing an old AVD)

```bash
avdmanager delete avd --name "<OldAVDName>"
sdkmanager --uninstall "system-images;<old-package-id>"
rm -rf ~/Library/Android/sdk/system-images/<old-api-dir>  # remove any 0-byte stale dirs
```

Stale system image directories (no `package.xml` inside) are not recognized by avdmanager — safe to delete.

---

## ⚠️ Pitfalls — do not do these

**`hw.gpu.mode=auto`** — silently falls back to software `lavapipe` rendering. Always set `host` explicitly.

**`adb shell settings put global hide_error_dialogs 1`** — triggers `system_server` restart → ADB goes offline → Expo/Metro port reverse fails with "device offline". Not a safe way to suppress ANR dialogs.

**`adb shell pm disable-user --user 0 <pkg>`** for debloating — system packages have hidden inter-dependencies. Disabling them causes the device to go offline; app dev tools (Expo, Metro) can't reconnect.

**Allocating >50% host RAM** — host OS starts swapping; the performance gain from more emulator RAM is wiped out by swap I/O.

**`-feature VulkanNativeSwapchain`** — incompatible with snapshot saving, produces `UNSUPPORTED_VK_APP` on every snapshot attempt.

**Resizable / Foldable / Experimental devices** — simulate hinge sensors and multiple form factors in background threads. All overhead, no dev benefit.

**Pre-release images** (`ps16k`, `-ext*` variants, `android-37.0`) — extra overhead, poor emulator stability.

**Post-boot ADB scripting for persistent state** — settings applied via `adb shell settings put` can disappear when `system_server` restarts. Prefer config.ini for anything that must persist.

**`hw.keyboard=no` (avdmanager default for real-device profiles)** — `avdmanager` sets `hw.keyboard=no` for all real Pixel profiles (phones have no hardware keyboard). This silently disables host keyboard input in the emulator. Always set `hw.keyboard=yes` in config.ini after creating the AVD.

**Snapshot saving with `-gpu host`** — gfxstream/Vulkan rendering is not serializable; snapshot save will silently fail (`UNSUPPORTED_VK_APP`). Accept cold boots (~25–40 s) and set `fastboot.forceColdBoot=yes`.

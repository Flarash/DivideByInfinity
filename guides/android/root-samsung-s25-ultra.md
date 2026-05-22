# Rooting Samsung Galaxy S25 Ultra (SM-S938B) on One UI 7 via Full Firmware Downgrade + Magisk

> This guide walks you through rooting a Samsung Galaxy S25 Ultra international Exynos model
> (SM-S938B) from scratch — including ADB setup, debloating, disabling One UI 8's blocked OEM
> Unlock toggle via a One UI 7 firmware downgrade, unlocking the bootloader, and installing Magisk
> root with Play Integrity bypass modules. Expect 4–6 hours end to end, a stable internet
> connection, and ~75 GB of free disk space. Knox will be permanently tripped.

---

## Prerequisites

### Hardware
- **Phone**: Samsung Galaxy S25 Ultra **international** (SM-S938B) — Exynos variant sold outside the US/Canada. US Snapdragon models (SM-S938U/U1/W) have no root path whatsoever.
- **PC**: Windows 10/11 desktop or laptop. ~75 GB free on a drive with good USB throughput. A D: or secondary drive is recommended.
- **USB cable**: USB-A to USB-C (not USB-C to USB-C). Connected directly to the motherboard's back-panel USB port — no hubs, no front-panel ports.
- **Internet on the phone**: the SIM must be active OR you must have a Wi-Fi hotspot available for ~30 seconds. Needed for Samsung's KG server validation step.

### Phone state requirements
| Check | Requirement |
|---|---|
| Model | SM-S938B (not SM-S938U) |
| Carrier installment plan | Fully paid off (Knox Guard must be released) |
| Knox warranty bit | 0 (untripped) — confirm with `adb shell getprop ro.boot.warranty_bit` |
| Firmware / BIT level | Must know your BIT. See Phase 1 for how to check. |
| Samsung account | Not strictly required, but must be able to skip it in setup wizard |

### Software (downloaded during the guide)
- **Google Android Platform Tools** (ADB + Fastboot) — installed via `winget`
- **SamloaderKotlin Bifrost** firmware downloader — https://github.com/zacharee/SamloaderKotlin/releases
- **Odin 3.14.1** flasher — standard version (not patched; S938B → S938B same-model flash)
- **Magisk v30.7 APK** — https://github.com/topjohnwu/Magisk/releases
- **Play Integrity Fork**, **Tricky Store**, **Shamiko** Magisk modules

### Skill level
Comfortable with Windows PowerShell, navigating Android settings, and following exact file-path instructions. No programming required.

---

## Important Warnings (Read First)

1. **Never use a US Snapdragon S25 Ultra (SM-S938U).** Samsung hardware-locked the bootloader on all North American variants. No root method exists.

2. **Never let Bifrost's "Check for Updates" auto-select firmware.** It will return the latest BIT (may be 9+), which permanently burns the anti-rollback fuse and locks you out of downgrading. Always use Manual mode with an explicit version string.

3. **Never flash firmware whose BIT digit exceeds your current BIT.** The BIT is the 5th digit from the end of the PDA string (e.g., `S938BXXU**5**BYI3` = BIT 5). Flashing a BIT 9 firmware on a BIT 5 device burns the fuse permanently and destroys the root path.

4. **Never load super.img without the PIT file + Re-Partition checked.** On S25 Ultra, every Odin flash that touches `super.img` (AP slot) requires the PIT file loaded in the Pit tab and Re-Partition checked. Without it, Odin always fails with `FAIL!` at `super.img`.

5. **Never use Re-Partition after bootloader unlock.** It re-wipes `frp` which re-locks the bootloader. All subsequent flashes must be AP-slot-only with no PIT and no Re-Partition.

6. **Never accept Samsung OTA updates after rooting.** OTA firmware has a different `vbmeta`, which will fail Magisk's verified-boot check and may trip anti-rollback.

7. **Samsung Pay, Secure Folder data, banking apps (MB WAY, Revolut, bank apps), and OTA updates are permanently lost** the moment you long-press Vol-Up to unlock the bootloader. Knox is a one-way door.

8. **The DroidWin hybrid-BL trick does NOT work on S25 Ultra.** Samsung rotated AVB signing keys between One UI 7 and One UI 8 on this model. A full One UI 7 downgrade is required instead.

9. **Never use `adb reboot download` to enter Download Mode before the bootloader unlock step.** It skips the blue Warning screen entry path, which can prevent the Vol-Up long-press from triggering the unlock prompt.

10. **Always run Odin as Administrator.** Right-click → Run as administrator.

---

## Phase 1 — ADB Setup and Pre-Flight Checks

### Step 1 — Install ADB

Open PowerShell and run:

```powershell
winget install Google.PlatformTools
```

After install, refresh the PATH in your current shell session (winget doesn't auto-update the running session):

```powershell
$env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
```

Verify:

```powershell
adb version
```

Expected output:
```
Android Debug Bridge version 1.0.41
```

### Step 2 — Enable USB Debugging on the phone

1. **Disable Auto Blocker** (blocks USB debugging by default on modern Samsung):
   - Settings → Security and privacy → **Auto Blocker** → toggle **Off** → confirm

2. **Enable Developer Options**:
   - Settings → About phone → Software information → tap **Build number** 7 times → enter PIN if prompted → "Developer mode enabled"

3. **Enable USB Debugging**:
   - Settings → Developer options → **USB debugging** → On

4. Connect USB cable. Accept the **"Allow USB debugging?"** popup on the phone and check **"Always allow from this computer"**.

### Step 3 — Verify connection and device info

```powershell
adb devices
```

Expected:
```
List of devices attached
<serial>    device
```

Collect device details:

```powershell
adb shell getprop ro.product.model
adb shell getprop ro.build.display.id
adb shell getprop ro.boot.warranty_bit
adb shell getprop ro.boot.kg
```

| Property | What to check |
|---|---|
| `ro.product.model` | Must be `SM-S938B` |
| `ro.build.display.id` | Contains your PDA string — note the **5th digit from the end** (your BIT level) |
| `ro.boot.warranty_bit` | Must be `0` (untripped Knox) |
| `ro.boot.kg` | Must be `0x4` or state `Completed` (Knox Guard released = phone paid off) |

**How to read the BIT level:**

```
S938BXXU5BYI3
        ^--- 5th from end = BIT 5
```

Write down your BIT number. You must download One UI 7 firmware at the **same BIT**.

---

## Phase 2 — Firmware Downloads

You need two firmware packages:
- **Current One UI 8** firmware (your running build) — for AP, CP, CSC slots
- **One UI 7 at matching BIT** — this is what you will flash to restore the OEM Unlock toggle

### Step 1 — Set up workspace

```powershell
New-Item -ItemType Directory -Force D:\S25U-Root\firmware\OneUI8
New-Item -ItemType Directory -Force D:\S25U-Root\firmware\OneUI7
New-Item -ItemType Directory -Force D:\S25U-Root\tools
New-Item -ItemType Directory -Force D:\S25U-Root\hybrid
```

> **Space requirement**: each firmware zip is ~24 GB. You need ≥ 72 GB free on D: to hold both.

### Step 2 — Download Bifrost, Odin, and Magisk

```powershell
# Download Bifrost (firmware downloader)
# From: https://github.com/zacharee/SamloaderKotlin/releases
# Pick the Windows amd64 zip. Extract to D:\S25U-Root\tools\bifrost\
# Verify D:\S25U-Root\tools\bifrost\bin\Bifrost.exe AND D:\S25U-Root\tools\bifrost\conf\ both exist.

# Download Odin 3.14.1 (standard version)
# From XDA or SamMobile: Odin3 v3.14.1.exe → place at D:\S25U-Root\tools\Odin\

# Download Magisk APK
# From: https://github.com/topjohnwu/Magisk/releases
# Latest stable → Magisk-vXX.X.apk → place at D:\S25U-Root\tools\Magisk-vXX.X.apk
```

> **Bifrost directory warning**: the `conf/` directory must exist next to `bin/`. If you move or copy only the `bin/` folder, Bifrost will throw `trustAnchors parameter must be non-empty` (Java SSL keystore error). Re-extract from the original zip if this happens.

### Step 3 — Download One UI 8 firmware (current build)

1. Launch Bifrost: double-click `D:\S25U-Root\tools\bifrost\bin\Bifrost.exe`
2. Tick ✅ **Manual** checkbox.
3. Fill in the fields:

| Field | Value |
|---|---|
| **Model** | `SM-S938B` |
| **Region** | `MEO` (your sales code) |
| **Firmware** | `<your-PDA>/S938BOXM5BYI3/<your-PDA>/<your-PDA>` (slash-separated 4-part string) |
| **Output folder** | `D:\S25U-Root\firmware\OneUI8` |

> The 4-part firmware string format is `PDA/CSC/PHONE/DATA`. Replace `<your-PDA>` with your actual build ID (e.g., `S938BXXU5BYI3`). The CSC part uses `S938BOXM5BYI3`.

4. Click **Download**. Wait (7–24 GB at your ISP speed).

> **Always verify the output folder before clicking Download.** Bifrost remembers the last path used, even across re-installs. If it shows an old path, change it.

### Step 4 — Download One UI 7 BIT-5 firmware

Determine the correct One UI 7 firmware version for BIT 5:

| BIT Level | One UI 7 firmware version to use |
|---|---|
| BIT 5 | `S938BXXS5AYG2` (August 2025) |
| BIT 4 | `S938BXXS4AYE1` (May 2025) |

In Bifrost, same settings but different firmware field and output folder:

| Field | Value |
|---|---|
| **Model** | `SM-S938B` |
| **Region** | `MEO` |
| **Firmware** | `S938BXXS5AYG2/S938BOXM5AYG2/S938BXXS5AYG2/S938BXXS5AYG2` |
| **Output folder** | `D:\S25U-Root\firmware\OneUI7` |

Click **Download**. The "Bifrost may not supply…" warning in Manual mode is normal — ignore it.

### Step 5 — Extract BL tarballs

Both downloads produce a large `.zip`. You only need to extract the BL file immediately (the others can wait):

```powershell
# Extract BL from One UI 8 zip
Add-Type -AssemblyName System.IO.Compression.FileSystem
$zip8 = [IO.Compression.ZipFile]::OpenRead("D:\S25U-Root\firmware\OneUI8\SM-S938B_*.zip")
$blEntry = $zip8.Entries | Where-Object { $_.Name -like "BL_*.tar.md5" }
[IO.Compression.ZipFileExtensions]::ExtractToFile($blEntry, "D:\S25U-Root\firmware\OneUI8\extracted\$($blEntry.Name)", $true)
$zip8.Dispose()

# Same for One UI 7
$zip7 = [IO.Compression.ZipFile]::OpenRead("D:\S25U-Root\firmware\OneUI7\SM-S938B_*.zip")
$blEntry7 = $zip7.Entries | Where-Object { $_.Name -like "BL_*.tar.md5" }
[IO.Compression.ZipFileExtensions]::ExtractToFile($blEntry7, "D:\S25U-Root\firmware\OneUI7\extracted\$($blEntry7.Name)", $true)
$zip7.Dispose()
```

> Or use 7-Zip GUI: right-click the zip → 7-Zip → Extract files → choose the `extracted\` subfolder.

Delete the encrypted `.enc4` files if present (they are not needed once decrypted):
```powershell
Remove-Item D:\S25U-Root\firmware\OneUI8\*.enc4 -ErrorAction SilentlyContinue
Remove-Item D:\S25U-Root\firmware\OneUI7\*.enc4 -ErrorAction SilentlyContinue
```

---

## Phase 3 — Extract AP, CP, CSC and PIT from One UI 7

The One UI 7 full flash needs all four Odin slots. Extract them now:

```powershell
$zip7 = [IO.Compression.ZipFile]::OpenRead((Get-Item "D:\S25U-Root\firmware\OneUI7\*.zip").FullName)
$targets = @("AP_", "CP_", "CSC_OXM", "HOME_CSC")
foreach ($entry in $zip7.Entries) {
    foreach ($prefix in $targets) {
        if ($entry.Name.StartsWith($prefix) -and $entry.Name.EndsWith(".tar.md5")) {
            $dest = "D:\S25U-Root\firmware\OneUI7\extracted\$($entry.Name)"
            [IO.Compression.ZipFileExtensions]::ExtractToFile($entry, $dest, $true)
        }
    }
}
$zip7.Dispose()
```

Also extract the **PIT file** from inside the CSC tar:

```powershell
New-Item -ItemType Directory -Force "D:\S25U-Root\firmware\OneUI7\extracted\pit" | Out-Null
tar -xf "D:\S25U-Root\firmware\OneUI7\extracted\CSC_OXM_*.tar.md5" --wildcards "*.pit" -C "D:\S25U-Root\firmware\OneUI7\extracted\pit\"
```

Verify you now have a `PA3Q_EUR_OPENX.pit` file (~17 KB) in the `pit\` folder.

---

## Phase 4 — Flash One UI 7 (Full Downgrade)

This flash replaces your One UI 8 with One UI 7, which still contains the OEM Unlock toggle Samsung removed in One UI 8.

### Step 1 — Enter Download Mode

1. Power off the phone completely (Power + Vol-Down → Power off → confirm).
2. With it **fully off**, hold **Vol-Up + Vol-Down** simultaneously, then plug in the USB cable while keeping both held.
3. Blue warning screen appears → press **Vol-Up** to continue → Download Mode screen.

> Download Mode shows: dark teal background, model/build info, `<!>` red triangle icon (normal), "Downloading… Do not turn off target!!" text.

### Step 2 — Configure Odin

1. Right-click `D:\S25U-Root\tools\Odin\Odin3 v3.14.1.exe` → **Run as administrator**.
2. Verify the top-left ID:COM box shows a port number in light-blue (e.g. `0:[COM7]`). If empty, USB driver issue — unplug/replug.
3. Load files into slots by clicking each button:

| Slot | File |
|---|---|
| **BL** | `D:\S25U-Root\firmware\OneUI7\extracted\BL_S938BXXS5AYG2_*.tar.md5` |
| **AP** | `D:\S25U-Root\firmware\OneUI7\extracted\AP_S938BXXS5AYG2_*_meta_OS15.tar.md5` |
| **CP** | `D:\S25U-Root\firmware\OneUI7\extracted\CP_S938BXXS5AYG2_*.tar.md5` |
| **CSC** | `D:\S25U-Root\firmware\OneUI7\extracted\CSC_OXM_S938BOXM5AYG2_*.tar.md5` ⚠ plain CSC (not HOME_CSC) |

> **When you click AP**, Odin will appear frozen / "Not Responding" for 30–60 seconds while it loads the 25 GB file. This is normal. Wait it out.

4. **Pit tab** → click **PIT** button → confirm warning → browse to `D:\S25U-Root\firmware\OneUI7\extracted\pit\PA3Q_EUR_OPENX.pit`.

5. **Options tab**:

| Option | State |
|---|---|
| Auto Reboot | ☐ OFF (unchecked) |
| Re-Partition | ☑ ON (checked) — critical |
| F. Reset Time | ☑ ON |
| Nand Erase | ☐ OFF |
| Flash Lock | ☐ OFF |

### Step 3 — Flash

Click **Start**. The flash takes ~10–12 minutes. Progress in the Log tab:

```
<ID:0/007> abl.elf
... (35 BL files)
<ID:0/007> super.img        ← slowest, takes 3-5 min
... (CP and CSC files)
<OSM> All threads completed. (succeed 1 / failed 0)
PASS!
```

When the green **PASS** box appears:

1. Odin shows PASS but Auto Reboot is off — phone is still in Download Mode.
2. Hold **Power + Vol-Down for 15 seconds** until screen goes black.
3. Release all buttons. Phone boots on its own.
4. You will see: Samsung logo → "Powered by Android" splash → boot animation → "Android is starting…" → "Optimizing apps X of YYY" (takes 5–15 min on first boot — do not interrupt) → Setup Wizard.

### Step 4 — First boot setup

In the Setup Wizard:
- **Skip Wi-Fi** for now OR connect — but **connect to internet** before the OEM Unlock step.
- **Skip Samsung account** (look for hidden Skip in top-right 3-dot menu, or go back and disable Wi-Fi during that screen).
- Minimal setup only — you will factory-reset again after unlock.

At the home screen:
1. **Enable Developer Options**: Settings → About phone → Software information → tap **Build number 7 times**.
2. **Enable OEM Unlocking**: Settings → Developer options → OEM Unlocking → **On**.
3. **Connect to internet** (Wi-Fi or mobile data) — wait ~30 seconds with OEM Unlocking toggled on. Samsung's server validates the toggle; without this, the long-press Vol-Up in Download Mode does nothing.
4. **Enable USB Debugging**: Settings → Developer options → USB debugging → On.

---

## Phase 5 — Bootloader Unlock (Knox Trip — Point of No Return)

⚠️ **This step permanently trips the Knox e-fuse.** Samsung Pay, Secure Folder data, banking apps, warranty, and OTA updates are lost forever after this step.

### Step 1 — Verify unlock is ready

```powershell
adb shell settings get global oem_unlock_supported
```

If this returns `1` or `null` (toggle visible), proceed. Connect USB, accept RSA prompt.

### Step 2 — Enter Download Mode via hardware keys

> Do NOT use `adb reboot download` for this step. It skips the warning screen and the unlock prompt may not respond.

1. Unplug USB cable from phone.
2. Power off: hold **Power + Vol-Down** for 15 seconds → screen goes black.
3. With phone fully off, hold **Vol-Up + Vol-Down** simultaneously.
4. Plug USB cable back in while holding both buttons.
5. Blue **Warning** screen → press **Vol-Up** once → Download Mode screen.

### Step 3 — Trigger unlock

On the Download Mode screen:
1. **Press and HOLD Vol-Up continuously for 10 full seconds.** Do not release, do not tap repeatedly. There is no visual feedback until the 7-second mark.
2. Screen switches to **"Device Unlock Mode"** warning (orange/red text describing what you're about to lose).
3. Press **Vol-Up** to confirm.
4. Phone factory-resets and reboots — takes 5–15 minutes.

### What to expect after unlock

- Phone boots with an **orange "Custom binary"** warning screen at every boot (permanent).
- Setup Wizard appears again (data wiped).
- Run through minimal setup: skip Samsung account, set a PIN.
- Enable Developer Options + USB Debugging again.
- Verify unlock from PC:

```powershell
adb shell getprop ro.boot.flash.locked
# Expected: 0

adb shell getprop ro.boot.verifiedbootstate
# Expected: orange
```

---

## Phase 6 — Magisk Root Install

### Step 1 — Extract init_boot.img from One UI 7 AP

On the PC:

```powershell
New-Item -ItemType Directory -Force D:\S25U-Root\root-work
# Extract init_boot.img.lz4 from the AP tar
tar -xf "D:\S25U-Root\firmware\OneUI7\extracted\AP_S938BXXS5AYG2_*.tar.md5" init_boot.img.lz4 -C D:\S25U-Root\root-work\
# Decompress lz4
lz4 -d D:\S25U-Root\root-work\init_boot.img.lz4 D:\S25U-Root\root-work\init_boot.img
```

> `lz4.exe` is included in the Android Platform Tools or available standalone. Alternatively, extract `init_boot.img.lz4` and rename it `.img` — Magisk accepts both.

### Step 2 — Push Magisk APK and init_boot to phone

```powershell
adb install "D:\S25U-Root\tools\Magisk-v30.7.apk"
adb push D:\S25U-Root\root-work\init_boot.img /sdcard/Download/init_boot.img
```

### Step 3 — Patch init_boot on phone

1. Open the **Magisk** app on the phone.
2. At the top, in the "Magisk" card, tap **Install**.
3. Choose **"Select and Patch a File"**.
4. File picker → navigate to Downloads → tap `init_boot.img`.
5. Tap **Let's Go** (top-right arrow button).
6. Wait 30–60 seconds → "All Done!" message.
7. The output file is at `/sdcard/Download/magisk_patched-XXXXX_XXXXX.img`.

### Step 4 — Pull patched image and repack as tar

```powershell
adb pull /sdcard/Download/magisk_patched-*.img D:\S25U-Root\root-work\magisk_patched.img
# Repack as tar.md5 for Odin
Push-Location D:\S25U-Root\root-work
tar -H ustar -cf magisk_patched.tar magisk_patched.img
$md5 = (Get-FileHash magisk_patched.tar -Algorithm MD5).Hash.ToLower()
Add-Content -NoNewline magisk_patched.tar "`n$md5  magisk_patched.tar"
Rename-Item magisk_patched.tar magisk_patched.tar.md5
Pop-Location
```

### Step 5 — Flash patched AP via Odin

Enter Download Mode (hardware keys method from Phase 5 Step 2).

In Odin (Run as Administrator):

| Slot | File |
|---|---|
| **AP** | `D:\S25U-Root\root-work\magisk_patched.tar.md5` |
| BL | *(leave empty)* |
| CP | *(leave empty)* |
| CSC | *(leave empty)* |

⚠️ **Options tab for this flash:**

| Option | State |
|---|---|
| Auto Reboot | ☑ ON |
| Re-Partition | ☐ **OFF** — critical |
| F. Reset Time | ☑ ON |
| Nand Erase | ☐ OFF |

⚠️ **Do NOT load a PIT file for this flash.** AP-only flashes never use PIT.

Click **Start**. This flash is small (~8 MB) and finishes in under 1 minute. Expect PASS then automatic reboot.

### Step 6 — Handle "Can't load Android System" (expected)

After the AP flash reboot, a stock Recovery screen may appear:

```
Can't load Android system. Your data may be corrupt.
[Try again]  [Factory data reset]
```

This is **normal** on the first Magisk root. The userdata encryption needs to be re-keyed against the patched `init_boot`. Press **Factory data reset** → confirm → wait 5–15 minutes.

After the automatic reboot and minimal setup:
1. Open **Magisk** app → verify "Installed" with a green checkmark and version number.
2. From PC:
```powershell
adb shell su -c "id"
# Expected: uid=0(root) gid=0(root)
```

Root is confirmed.

---

## Phase 7 — Post-Root Hardening

### Step 1 — Re-disable Auto Blocker

Settings → Security and privacy → Auto Blocker → **Off**. (It re-enables after every factory reset.)

### Step 2 — Enable Zygisk

1. Magisk app → gear icon (Settings) → **Zygisk** → toggle On.
2. Reboot.

### Step 3 — Install root-hiding modules

Download the three modules to the phone:

```powershell
# Download from GitHub releases to PC, then push to phone
# PlayIntegrityFork: https://github.com/chiteroman/PlayIntegrityFix/releases
# TrickyStore: https://github.com/5ec1cff/TrickyStore/releases
# Shamiko: https://github.com/LSPosed/LSPosed.github.io/releases

adb push PlayIntegrityFork-vXX.zip /sdcard/Download/
adb push Tricky-Store-vX.X.X-*.zip /sdcard/Download/
adb push Shamiko-vX.X.X-release.zip /sdcard/Download/
```

Install in Magisk (install all three before rebooting):
1. Magisk → **Modules** tab → **Install from storage** → select PlayIntegrityFork zip → Install.
2. Install from storage → TrickyStore zip → Install.
3. Install from storage → Shamiko zip → Install.
4. Reboot once.

### Step 4 — Configure DenyList (Shamiko root hiding)

1. Magisk → Settings → **Enforce DenyList** → On.
2. Magisk → **Configure DenyList** → add every banking app, Samsung Wallet, Netflix, and any app that fails root/integrity checks.

### Step 5 — Verify Play Integrity

Install **"Play Integrity API Checker"** by `@1nikolas` from Play Store. Tap Check. You should see:

| Verdict | Target |
|---|---|
| MEETS_BASIC_INTEGRITY | ✅ |
| MEETS_DEVICE_INTEGRITY | ✅ |
| MEETS_STRONG_INTEGRITY | May fail (optional) |

---

## Troubleshooting

### Odin FAIL at `super.img` (no DDP message)

**Symptom**: `FAIL!` at `super.img`, no red text on phone screen.
**Cause**: USB transmission error on the large sparse-LZ4 transfer.
**Fix**: In Odin click Reset (don't change files), move USB cable to a different back-panel port, close all other apps, click Start again. If it fails a third time, try a different USB cable.

---

### Odin FAIL with "DDP: Download failed due to incompatible DOWNLOAD program"

**Symptom**: Flash fails at `super.img`, phone shows red text "DDP: Download failed due to incompatible DOWNLOAD program".
**Cause**: PIT file not loaded or Re-Partition not checked. Required on S25 Ultra for any flash that writes `super.img`.
**Fix**:
1. In Odin, click Reset.
2. Pit tab → click PIT → load `PA3Q_EUR_OPENX.pit` from the extracted firmware folder.
3. Options tab → check Re-Partition.
4. Start again.

---

### Bifrost `trustAnchors parameter must be non-empty`

**Symptom**: Bifrost crashes immediately with a Java SSL error.
**Cause**: The `conf/security/java.security` file is missing from the Bifrost installation directory.
**Fix**: Re-extract Bifrost from the original zip into a clean directory. The `conf/` folder must exist alongside `bin/`.

---

### OEM Unlock toggle missing in Developer Options

**Symptom**: "OEM unlocking" does not appear in Developer Options.
**Cause**: You are on One UI 8 (Samsung removed it globally in July 2025), or the One UI 7 flash hasn't been done yet.
**Fix**: Complete Phase 4 (One UI 7 full downgrade). The toggle reappears after booting One UI 7.

---

### Vol-Up long-press does nothing in Download Mode

**Symptom**: Holding Vol-Up for 10 seconds on the Download Mode screen has no effect.
**Cause (most common)**: Phone has not connected to the internet since enabling OEM Unlock. Samsung's server must validate the toggle before the bootloader accepts the unlock command.
**Fix**:
1. Force-shutdown: Power + Vol-Down 15 seconds.
2. Boot to Android.
3. Connect to Wi-Fi or enable mobile data.
4. Settings → Developer options → OEM Unlocking — confirm it is still On.
5. Wait 30 seconds.
6. Power off → enter Download Mode via hardware keys → long-press Vol-Up.

**Alternative cause**: Entered Download Mode via `adb reboot download` (skips warning screen).
**Fix**: Use the hardware key method exclusively (Phase 5 Step 2).

---

### Hybrid BL method fails, phone auto-falls to Download Mode

**Symptom**: Odin reports PASS for hybrid BL flash, but phone cannot boot — automatically enters Download Mode with no DDP error.
**Cause**: Samsung rotated AVB signing keys between One UI 7 and One UI 8 on S25 Ultra. The One UI 7 `abl.elf` cannot verify One UI 8's `vbmeta_system`. The hybrid-BL (Droidwin) trick is patched on this model.
**Fix**: Do a full One UI 7 downgrade (Phase 4) instead. Do not attempt the hybrid method on SM-S938B.

---

### "Can't load Android system" after Magisk AP flash

**Symptom**: After flashing Magisk-patched AP, Recovery shows "Can't load Android system".
**Cause**: Normal — userdata encryption context doesn't match the patched `init_boot`.
**Fix**: Press **Factory data reset** on that recovery screen. Magisk root is baked in; it will be active after the factory reset and reboot.

---

### SetupConnection stuck in Odin after previous FAIL

**Symptom**: After retrying a flash, Odin gets stuck at `SetupConnection..` indefinitely.
**Cause**: Bootloader is in a limbo state from the prior failed session.
**Fix**:
1. Close Odin entirely.
2. Unplug USB.
3. Power + Vol-Down 15 seconds → screen black → release all.
4. Re-enter Download Mode via hardware keys (Vol-Up + Vol-Down + USB).
5. Reopen Odin as Administrator → reload files → Start.

---

## Lessons Learned

- Samsung removed the OEM Unlock toggle **globally** from One UI 8 firmware in July 2025. The only way to get it back without exploit-based root is to flash One UI 7.
- The BIT anti-rollback fuse is the critical compatibility check. You can downgrade across major OS versions freely as long as the BIT digit matches.
- S25 Ultra Odin flashes are unique: every full flash (any firmware touching `super.img`) needs the PIT file + Re-Partition. Skip either and you get the DDP error every single time.
- The DroidWin hybrid-BL trick (swap One UI 7 `abl.elf.lz4` into One UI 8 BL) does not work on SM-S938B because Samsung rotated the AVB signing keys. Z Fold 6 tutorials do not transfer.
- Magisk patches `init_boot.img` on S25 Ultra (not `boot.img`) because this device uses system-as-root with no ramdisk in `boot`.
- AP-only flashes (no PIT, no Re-Partition) preserve the bootloader unlock state. Full flashes with Re-Partition wipe `frp` and re-lock the bootloader.
- After rooting, never accept Samsung OTA updates. Flash firmware manually via AP-only Odin after re-patching `init_boot` with Magisk each time.
- `adb reboot download` is convenient but bypasses the blue warning screen — always use hardware keys for the bootloader unlock step.
- Bifrost's "Check for Updates" is dangerous: it auto-selects the latest firmware which may be a higher BIT. Always use Manual mode with an explicit firmware string.
- Bifrost remembers the last output folder across launches. Always visually verify the output path before clicking Download.

---

## Files / Artifacts Produced

| Path | Description |
|---|---|
| `D:\S25U-Root\firmware\OneUI8\SM-S938B_*_S938BXXU5BYI3_*.zip` | Stock One UI 8 firmware zip (keep as restore point) |
| `D:\S25U-Root\firmware\OneUI8\extracted\BL_*.tar.md5` | Extracted One UI 8 bootloader |
| `D:\S25U-Root\firmware\OneUI8\extracted\AP_*.tar.md5` | Extracted One UI 8 Android partitions (25 GB) |
| `D:\S25U-Root\firmware\OneUI8\extracted\CP_*.tar.md5` | Extracted One UI 8 modem |
| `D:\S25U-Root\firmware\OneUI8\extracted\HOME_CSC_OXM_*.tar.md5` | Extracted One UI 8 CSC (data-preserving) |
| `D:\S25U-Root\firmware\OneUI8\extracted\pit\PA3Q_EUR_OPENX.pit` | Partition table file (required for all full flashes) |
| `D:\S25U-Root\firmware\OneUI7\SM-S938B_*_S938BXXS5AYG2_*.zip` | One UI 7 BIT-5 firmware zip (keep — needed for re-root) |
| `D:\S25U-Root\firmware\OneUI7\extracted\` | Extracted One UI 7 BL, AP, CP, CSC, pit (all slots for the downgrade flash) |
| `D:\S25U-Root\tools\bifrost\` | Bifrost firmware downloader (full directory — do not move bin/ alone) |
| `D:\S25U-Root\tools\Odin\Odin3 v3.14.1.exe` | Odin flasher |
| `D:\S25U-Root\tools\Magisk-vXX.X.apk` | Magisk installer APK |
| `D:\S25U-Root\root-work\init_boot.img` | Stock unpatched One UI 7 init_boot (keep for re-rooting after firmware updates) |
| `D:\S25U-Root\root-work\magisk_patched.tar.md5` | Magisk-patched init_boot — the file actually flashed via Odin to root the phone |
| `<copilot-workspace>\artifacts\restore-tierA.ps1` | ADB script to restore the 16 Tier A packages removed in debloat |
| `<copilot-workspace>\artifacts\packages-all.txt` | Snapshot of all packages before debloat |

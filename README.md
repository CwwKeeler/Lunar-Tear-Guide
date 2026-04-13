# Lunar Tear Guide

A worked Windows setup guide for getting the NieR Re[in]carnation private server **Lunar Tear** running.

This guide is based only on the path that actually worked in practice. It does **not** try to document every possible setup path.

## What This Guide Covers

- Windows prerequisites
- required repo layout
- server asset placement
- patching the master database
- generating `master_data`
- patching and signing the APK
- launching the server
- using a real Android device, MuMu, or Android Studio emulator
- pitfalls that actually came up during setup

## Repos Needed

Keep these two repos side by side:

- `lunar-tear`
- `nier-rein-apps`

The `nier-rein-apps` repo matters because Lunar Tear's master-data tools rely on this schema directory:

- `nier-rein-apps/src/NierReincarnation.Core/Dark`

Without that folder, the encrypted master DB can exist, but `master_data/*.json` will not be generated correctly.

## Prerequisites

These tools need to be installed and usable from PowerShell:

- Go
- Python
- Java JDK (`keytool`)
- Android SDK Platform Tools (`adb`)
- Android SDK Build Tools (`zipalign`, `apksigner`)
- `apktool`

Useful verification commands:

```powershell
go version
python --version
keytool -help
adb version
apktool --version
apksigner
zipalign
```

## Server Asset Layout

Inside `lunar-tear/server/assets`, the final working structure should look like this:

- `release/`
- `revisions/`
- `master_data/`

Meaning:

- `release` contains the encrypted master DB `.bin.e`
- `revisions` contains revision folders, `list.bin`, and asset bundles
- `master_data` contains generated JSON tables

## Step 1: Clone the Repos

Example layout:

```text
Projects/
  lunar-tear/
  nier-rein-apps/
```

Build `nier-rein-apps` so its schema directory is available.

## Step 2: Put the Assets in the Right Folders

Place the encrypted master DB file in:

- `lunar-tear/server/assets/release/<your file>.bin.e`

Place revision/list/bundle files in:

- `lunar-tear/server/assets/revisions/`

## Step 3: Patch the Master DB

Run:

```powershell
python .\scripts\patch_masterdata.py .\server\assets\release\<YOUR_FILE>.bin.e
```

This patches the master database so Lunar Tear can use it correctly.

## Step 4: Generate `master_data`

Run:

```powershell
python .\scripts\dump_masterdata.py --input .\server\assets\release\<YOUR_FILE>.bin.e --output .\server\assets\master_data
```

Result:

- `server/assets/master_data` should fill with JSON files

Important:

- this step depends on `nier-rein-apps/src/NierReincarnation.Core/Dark`

## Step 5: Choose Your Target Device Type

Before patching the APK, choose where the game will run.

### Option A: Real Android Device or LAN Emulator

Use your PC's LAN IP, for example:

- `192.168.5.170`

### Option B: Android Studio Emulator on the Same PC

Use:

- `10.0.2.2`

That special IP lets the emulator talk back to the host machine.

## Step 6: Decompile the APK

Use your chosen game APK, then decompile it with `apktool`.

Example output directory:

- `client/patched`

## Step 7: Patch the APK

Use Lunar Tear's patcher against the decompiled APK directory.

### For a Real Device / LAN Emulator

```powershell
python .\scripts\patch_apk.py .\client\patched --server-ip 192.168.5.170 --http-port 8080
```

### For Android Studio Emulator

```powershell
python .\scripts\patch_apk.py .\client\patched --server-ip 10.0.2.2 --http-port 8080
```

What this patcher changes:

- metadata strings for the private server
- `libil2cpp.so`
- cleartext/network security settings
- manifest/network config

## Step 8: Rebuild and Sign the APK

Run:

```powershell
apktool b .\client\patched -o .\client\patched.apk
zipalign -f 4 .\client\patched.apk .\client\patched-aligned.apk
apksigner sign --ks .\client\debug.keystore --ks-pass pass:android --out .\client\patched-signed.apk .\client\patched-aligned.apk
```

Installable result:

- `client/patched-signed.apk`

If you want to keep separate builds, use different names such as:

- `patched-signed.apk` for LAN/device
- `patched-emu-signed.apk` for Android Studio emulator

## Step 9: Start the Server

A working PowerShell launcher can be used to start Lunar Tear.

### For Android Studio Emulator

```powershell
powershell -ExecutionPolicy Bypass -File ".\scripts\start_lunar_tear.ps1" -HostIp 10.0.2.2 -HttpPort 8080
```

### For Real Device / LAN Emulator

```powershell
powershell -ExecutionPolicy Bypass -File ".\scripts\start_lunar_tear.ps1" -HostIp 192.168.5.170 -HttpPort 8080
```

The launcher should verify assets and then run the server in the foreground so logs are visible.

## Step 10: Install the Patched APK

Install the correct signed APK onto your target device or emulator.

Example:

```powershell
adb install -r .\client\patched-signed.apk
```

If more than one Android device is attached, use `adb -s <DEVICE_ID>`.

## What Actually Mattered Server-Side

A stock fresh user state was not enough to get past early title/login flow.

The working setup ended up depending on these server-side behaviors:

- `GetUserData` needed to use the full projected user snapshot instead of a stripped first-entrance table set
- fresh users needed starter state seeded:
  - starter characters
  - starter costumes
  - starter weapons
  - starter companions
  - a default quest deck
  - a minimal initial quest row
- starter deck characters could not be sent with `power: 0`

In practice, healthy early `GetUserData` output included non-empty rows for:

- `IUserCharacter`
- `IUserCostume`
- `IUserWeapon`
- `IUserCompanion`
- `IUserDeckCharacter`
- `IUserDeck`
- `IUserQuest`
- `IUserTutorialProgress`

## Android Studio Emulator Notes

This is the least certain part because emulator images vary.

What is known:

- using an emulator build patched for `10.0.2.2` worked conceptually
- some newer Android Studio emulator images use **16 KB page size** and will fail to load this game

A failing emulator will show an error like:

- `libmain.so program alignment (4096) cannot be smaller than system page size (16384)`

If that happens, that emulator image will not work for this APK.

Recommended rule:

- use an Android 12 or Android 13 image that does **not** have the 16 KB page-size incompatibility

I am intentionally not naming one exact emulator image here because Android Studio image availability changes over time.

## MuMu / Frida Notes

Frida was useful for debugging, but it was **not required** to get the setup itself working.

What did work well enough for debugging later:

- install `frida-tools`
- use the correct `frida-server` build for the emulator ABI
- push it to the emulator
- forward ports to `127.0.0.1:27042`

What was less reliable:

- MuMu process/module visibility for Unity/IL2CPP debugging was inconsistent compared to a cleaner Android Studio emulator flow

So for a first setup, focus on getting:

1. assets in place
2. master DB patched
3. `master_data` generated
4. APK patched and signed
5. server running
6. app installed and connecting

## Working Command Summary

### Patch master DB

```powershell
python .\scripts\patch_masterdata.py .\server\assets\release\<file>.bin.e
```

### Generate `master_data`

```powershell
python .\scripts\dump_masterdata.py --input .\server\assets\release\<file>.bin.e --output .\server\assets\master_data
```

### Patch APK for Android Studio emulator

```powershell
python .\scripts\patch_apk.py .\client\patched --server-ip 10.0.2.2 --http-port 8080
```

### Patch APK for real device / LAN emulator

```powershell
python .\scripts\patch_apk.py .\client\patched --server-ip 192.168.x.x --http-port 8080
```

### Rebuild and sign APK

```powershell
apktool b .\client\patched -o .\client\patched.apk
zipalign -f 4 .\client\patched.apk .\client\patched-aligned.apk
apksigner sign --ks .\client\debug.keystore --ks-pass pass:android --out .\client\patched-signed.apk .\client\patched-aligned.apk
```

### Start server for Android Studio emulator

```powershell
powershell -ExecutionPolicy Bypass -File ".\scripts\start_lunar_tear.ps1" -HostIp 10.0.2.2 -HttpPort 8080
```

### Start server for real device / LAN emulator

```powershell
powershell -ExecutionPolicy Bypass -File ".\scripts\start_lunar_tear.ps1" -HostIp 192.168.x.x -HttpPort 8080
```

## Final Notes

If you already have all the files and want the shortest path:

1. populate `release`, `revisions`, and `master_data`
2. patch the APK for your target IP
3. rebuild/sign it
4. start the server
5. install the signed APK

If the app connects but stalls very early, the most likely cause is not basic networking anymore. It is usually one of these:

- missing or bad `master_data`
- incomplete starter user-data projection
- wrong target IP for the chosen emulator/device

---

If you improve this guide with a more reliable Android Studio AVD recommendation, that would be the main area still worth tightening up.
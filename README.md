# Grampus, TrimUI Brick PAK

A terminal-based ORCA-inspired grid sequencer paired with built-in polyphonic synth engines for the TrimUI Brick.

## Install

Available via **PAK Store** on NextUI. Connect your TrimUI Brick to Wi-Fi, open PAK Store from the Tools menu, and install Grampus.

## User Data

Projects, recordings, and demo projects are stored in:

```
/mnt/SDCARD/Grampus/
```

This folder is on the root of your SD card for easy access. It survives PAK updates. On every launch, the bundled `_demo_projects/` folder is refreshed inside `/mnt/SDCARD/Grampus/projects/_demo_projects/`, your own saves under `/mnt/SDCARD/Grampus/projects/` are never touched.

## Manual Installation

If for whatever reason you do not have the PAK Store:

1. Download the latest release from this repo.
2. Unzip the release download.
3. Copy the `Grampus.pak` folder to `SD_ROOT/Tools/tg5040`.
4. Reinsert your SD Card into your device.
5. Launch Grampus from the Tools menu.

## Desktop / Other Platforms

Grampus also runs on macOS, Linux, and Windows. Desktop builds are distributed at: https://boorch.itch.io/grampus

The full operator reference, instrument docs, and feature manual live in [MANUAL.md](MANUAL.md), same content as the desktop build's README.

## Running Grampus on non-TrimUI-Brick devices (unsupported)

> **This is not officially supported.** Grampus's TrimUI build is developed and tested only on the TrimUI Brick. If you've installed the PAK on a different device (Smart Pro, an RG-series handheld, or anything else) buttons may behave wrong or do nothing, because the gamepad mapping shipped in `launch.sh` is hand-tuned for the Brick's exact controller. The notes below are a courtesy guide for users who want to try Grampus on those devices anyway. **No support is offered for issues on non-Brick hardware.**

The PAK ships a helper script, `apply_gamepad_mapping.sh`, that auto-detects whatever SDL sees for your gamepad and rewrites `launch.sh` with the right mapping. Brick users never need to touch it.

There are two ways to run it.

### Method 1: File Manager (no SSH required)

1. On your device, open any file manager that can launch shell scripts (e.g. **Files.pak** on NextUI, or whatever your firmware's equivalent is).
2. Navigate into the **Grampus.pak** folder. Path varies per device, but on TrimUI variants installed via PakStore it's typically `/mnt/SDCARD/Tools/tg5040/Grampus.pak/`.
3. Find the file **`apply_gamepad_mapping.sh`** and execute it (usually a tap or "Run" action).
4. You almost certainly won't see any visible feedback (most file managers run shell scripts silently). That's fine; the script is doing its work.
5. Re-launch Grampus. If buttons feel correct now, you're done.

**Recovery / Undo.** The first time you run the script it creates a backup file called `launch.sh.bak` in the same folder, containing the original Brick-tuned mapping. (Subsequent runs do NOT overwrite this backup, so you always have the truly-original.) To revert: delete `launch.sh`, rename `launch.sh.bak` to `launch.sh`, re-launch Grampus.

### Method 2: SSH (recommended if you have it set up)

You'll see actual output telling you what changed.

1. SSH into your device. (If your firmware doesn't have SSH out of the box, NextUI offers a `NativeSSH.pak` you can install from PakStore.)
2. `cd` into the Grampus.pak folder. On most TrimUI variants installed via PakStore:
   ```sh
   cd /mnt/SDCARD/Tools/tg5040/Grampus.pak/
   ```
   Adjust the path if your device puts PAKs elsewhere.
3. Run the apply script:
   ```sh
   ./apply_gamepad_mapping.sh
   ```
4. You'll see output similar to:
   ```
   Probing connected joystick...
   Saved original launch.sh → launch.sh.bak

   OLD mapping (will be commented out):
     030000005e0400008e02000014010000,TrimUI Player1,...

   NEW mapping (auto-detected):
     0300abcd1234.....,Your Device Name,...

   Applied. Re-launch Grampus from the Tools menu for the change to take effect.
   To revert: cp launch.sh.bak launch.sh
   ```
5. Re-launch Grampus from the device menu.

**Recovery via SSH.** A single command restores the original mapping:
```sh
cp launch.sh.bak launch.sh
```

### Troubleshooting

- **"Mappings are identical, launch.sh already up to date."** SDL on your device happens to report exactly the same mapping that was already in launch.sh. Nothing to do; the original mapping was correct.
- **"ERROR: Couldn't read a mapping from gamepad_config -l output."** SDL didn't detect any joystick at all on your device. At this point you're on your own to figure out what's wrong; this is well outside what Grampus can help with.
- **Buttons still wrong after applying.** Your device might genuinely need a custom interactive walkthrough. Run `./probe_gamepad.sh -c 0` (SSH only) for the full prompt-by-prompt mapper, then manually paste the resulting string into `launch.sh`'s `SDL_GAMECONTROLLERCONFIG=` line.
- **Want to verify what SDL actually sees?** Run `./probe_gamepad.sh -l` (SSH). This shows the raw joystick + GameController info side by side, including the auto-detected mapping string.

## Screenshots

![Splash](.github/screenshots/splash.jpg)

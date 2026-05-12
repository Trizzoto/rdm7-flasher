# RDM-7 Dash — Recovery Flasher

A static web page that lets users reflash an RDM-7 Dash over USB. Built with
[ESP Web Tools](https://esphome.github.io/esp-web-tools/).

## Two modes

**Firmware Update** (recommended — `manifest.json`)
Writes only the firmware image + OTA selector. Preserves Wi-Fi credentials,
all settings (NVS), and layouts/images/fonts (LittleFS). The everyday recovery
path when OTA updates aren't working.

**Full Recovery Flash** (last resort — `manifest-full.json`, `erase-first`)
Erases the entire flash and reinstalls bootloader + partition table +
firmware. Wipes Wi-Fi, settings, layouts, everything. User redoes the
first-run wizard. Only for bricked devices or boot loops.

Audience: end users with a deployed dashboard stuck on an old firmware version
that can't OTA-update — or a fully bricked device for the full-recovery path.

## Hosting

This is a fully static site — no build step, no server-side code. Drop the
folder onto any static host.

**Important:** Web Serial API requires HTTPS (or `http://localhost`). Hosts
that work out of the box:

- Cloudflare Pages
- GitHub Pages
- Netlify
- Vercel

Just connect this repo and point the host at the root directory.

For local testing:

```bash
# Python 3
python -m http.server 8000
# then open http://localhost:8000
```

## Browser support

Web Serial API only works in Chromium-based desktop browsers:

- Chrome 89+
- Edge 89+
- Opera 75+

It does **not** work in Safari, Firefox, iOS, or Android browsers. The page
detects this and shows an unsupported-browser message.

## Updating the firmware

When you publish a new firmware release:

1. Build the firmware: `idf.py build` in the main `RDM-7_Dash` repo.
2. Copy the four binaries from `RDM-7_Dash/build/` into `firmware/`:
   - `bootloader/bootloader.bin`             → `firmware/bootloader.bin`
   - `partition_table/partition-table.bin`   → `firmware/partition-table.bin`
   - `ota_data_initial.bin`                  → `firmware/ota_data_initial.bin`
   - `esp32-firmware.bin`                    → `firmware/esp32-firmware.bin`
3. Bump the `version` field in `manifest.json` to match.
4. Update the `version-pill` in `index.html` so the user sees the new version.
5. Commit and push — your static host will redeploy.

Or use the convenience script:

```bash
./tools/sync-firmware.sh /path/to/RDM-7_Dash
```

(The script doesn't exist yet — easy to add if useful.)

## Repo layout

```
.
├── index.html              Landing page + guide
├── manifest.json           ESP Web Tools manifest — firmware update mode (2 parts)
├── manifest-full.json      ESP Web Tools manifest — full recovery mode (4 parts + erase)
├── firmware/               Firmware binaries (committed; not built here)
│   ├── bootloader.bin
│   ├── partition-table.bin
│   ├── ota_data_initial.bin
│   └── esp32-firmware.bin
└── assets/
    └── logo.png            RDM logo
```

## Partition offsets

Both manifests use these offsets, which match `partitions.csv` in the firmware repo:

| Binary                  | Offset (hex) | Offset (decimal) | In `manifest.json` | In `manifest-full.json` |
|-------------------------|--------------|------------------|--------------------|-------------------------|
| `bootloader.bin`        | `0x0000`     | 0                | No                 | Yes                     |
| `partition-table.bin`   | `0x8000`     | 32768            | No                 | Yes                     |
| `ota_data_initial.bin`  | `0x2D000`    | 184320           | Yes                | Yes                     |
| `esp32-firmware.bin`    | `0x30000`    | 196608           | Yes                | Yes                     |

### Why Firmware Update skips bootloader and partition-table

Deployed devices already have these and they haven't changed since the
project's initial commit. Writing them risks affecting the adjacent NVS
partition at `0x9000` (Wi-Fi credentials, settings). Skipping them keeps the
flash region a comfortable distance from NVS while still installing new
firmware and pointing the bootloader at it.

If you ever change `partitions.csv` or the bootloader, you'd need to either:
1. Add those back to `manifest.json` for one release — but users would lose
   settings on that update.
2. Or instruct users to use Full Recovery for that one release, then revert
   to Firmware Update for subsequent ones.

## What gets preserved by each mode

| Partition                                | Firmware Update | Full Recovery |
|------------------------------------------|-----------------|---------------|
| `nvs` (Wi-Fi, settings, calibrations)    | Preserved       | Erased        |
| `littlefs` (layouts, images, fonts)      | Preserved       | Erased        |
| `phy_init` (RF calibration)              | Preserved       | Erased*       |
| `bootloader`                             | Preserved       | Rewritten     |
| `partition-table`                        | Preserved       | Rewritten     |
| `otadata` (OTA selector)                 | Reset           | Reset         |
| `ota_0` / `ota_1` (app slots)            | `ota_0` written | `ota_0` written |

*Full Recovery uses `erase-first` on the install button, which issues
`esptool.eraseFlash()` before writing any parts — wiping the whole 16 MB chip.

`manifest.json` has `"new_install_prompt_erase": false` so ESP Web Tools
doesn't even offer an erase option during Firmware Update — preventing
accidental factory resets.

`manifest-full.json` has `"new_install_prompt_erase": true` *and* the button
uses the `erase-first` attribute, so the user is explicitly opting into a
full wipe via that button.

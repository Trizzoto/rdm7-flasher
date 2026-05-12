# RDM-7 Dash — Recovery Flasher

A static web page that lets users reflash an RDM-7 Dash over USB when Wi-Fi
updates don't work. Built with [ESP Web Tools](https://esphome.github.io/esp-web-tools/).

Audience: end users with a deployed dashboard stuck on an old firmware version
that can't successfully OTA-update.

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
├── index.html         Landing page + guide
├── manifest.json      ESP Web Tools install manifest (chip, parts, offsets)
├── firmware/          Firmware binaries (committed; not built here)
│   ├── bootloader.bin
│   ├── partition-table.bin
│   ├── ota_data_initial.bin
│   └── esp32-firmware.bin
└── assets/
    └── logo.png       RDM logo
```

## Partition offsets

The offsets in `manifest.json` match the partition table from the firmware build:

| Binary                  | Offset (hex) | Offset (decimal) | In manifest? |
|-------------------------|--------------|------------------|--------------|
| `bootloader.bin`        | `0x0000`     | 0                | No           |
| `partition-table.bin`   | `0x8000`     | 32768            | No           |
| `ota_data_initial.bin`  | `0x2D000`    | 184320           | Yes          |
| `esp32-firmware.bin`    | `0x30000`    | 196608           | Yes          |

**Why bootloader and partition-table are skipped:** they haven't changed
since the project's initial commit, and writing them risks affecting the
adjacent NVS partition at `0x9000`. Deployed devices already have the
correct bootloader and partition table, so flashing them again is wasted
work — and risks wiping settings. If you ever change `partitions.csv` or
the bootloader, add those back to the manifest for one release, then
remove them again.

The two parts we keep are necessary:
- `ota_data_initial.bin` — resets the OTA selector so the bootloader picks
  `ota_0` (where we just wrote the new firmware).
- `esp32-firmware.bin` — the new firmware image, written to `ota_0`.

The firmware binaries in `firmware/` still include `bootloader.bin` and
`partition-table.bin` for completeness (they're kept in case a future
release needs to flash them).

## What gets preserved during a flash

The flasher writes only the OTA app slot + the OTA selector. It does **not**
touch:

- `nvs` (Wi-Fi credentials, calibrations, all user settings)
- `littlefs` (layouts, images, fonts)
- `phy_init` (RF calibration)
- `bootloader` / `partition-table`

Users keep all saved data and reconnect to Wi-Fi automatically.

`manifest.json` has `"new_install_prompt_erase": false` so ESP Web Tools
doesn't offer a full-chip-erase option — preventing accidental factory resets.

# RDM-7 Dash — Recovery Flasher

A static web page that lets users reflash an RDM-7 Dash over USB. Built with
[ESP Web Tools](https://esphome.github.io/esp-web-tools/).

## Live log viewer (921600 baud)

The page also includes a **Live Device Logs** panel that streams plain-text
ESP-IDF logs over USB at 921600 baud — the rate the firmware uses for its
UART1 protocol. esp-web-tools' built-in console is hardcoded at 115200 and
can't be reconfigured via manifest, so the page uses a separate Web Serial
connection for log viewing.

The viewer:
- Opens its own serial port via `navigator.serial.requestPort()` — user
  picks the same port they flashed with
- Strips ANSI color codes and re-colors lines by ESP-IDF level (I/W/E/D/V)
- Caps at 2000 lines and auto-scrolls (pauses auto-scroll on manual scroll)
- Has Pause / Clear / Disconnect controls

It is independent from the install dialog — flashing and log viewing don't
share state. After a flash completes, the user disconnects the install
dialog's port, then clicks Connect on the log viewer.

## Two modes

Both modes pop up the same esp-web-tools dialog that asks "Erase device?" with
a checkbox. The user's choice on that checkbox determines whether NVS / LittleFS
data is preserved or wiped. The two modes differ in **which binaries are
written**.

**Firmware Update** (`manifest.json` — 2 parts)
Writes `ota_data_initial.bin` and `esp32-firmware.bin`. Skips bootloader and
partition table. Use for everyday firmware updates over USB. User should leave
the erase checkbox **unchecked** to preserve settings.

**Full Recovery Flash** (`manifest-full.json` — 4 parts)
Writes all four binaries (bootloader, partition table, otadata, firmware).
Use when the bootloader or partition table got corrupted. User can leave the
erase checkbox **unchecked** to repair without losing data, or **checked** for
a true factory reset.

## Critical: how `new_install_prompt_erase` actually works

This was tripped over during development — documenting because the manifest
field name is misleading.

Looking at [esp-web-tools install-dialog.ts line 318-323](https://github.com/esphome/esp-web-tools/blob/main/src/install-dialog.ts):

```typescript
if (this._manifest.new_install_prompt_erase) {
  this._state = "ASK_ERASE";
} else {
  // Default is to erase a device that does not support Improv Serial
  this._startInstall(true);   // ← TRUE means erase the chip
}
```

So for devices that don't run [Improv-Serial](https://www.improv-wifi.com/serial/)
(our case — the firmware doesn't include it):

- **`new_install_prompt_erase: false`** → no prompt, **chip auto-erases**
- **`new_install_prompt_erase: true`** → user gets an erase checkbox (default unchecked)

The field name reads like "prompt the user about erase, default no" — but it
really means "show a prompt at all, or just erase automatically."

Both our manifests use `"new_install_prompt_erase": true` so the user is in
control via the checkbox.

The `erase-first` attribute on `<esp-web-install-button>` is declared on the
element (`install-button.ts:57`) but **never propagated** to the install dialog
by `connect.ts` — it's effectively dead code in current esp-web-tools. Don't
rely on it.

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

## What gets preserved by each mode and checkbox state

| Partition                       | FW Update<br>unchecked | FW Update<br>checked* | Full Recovery<br>unchecked | Full Recovery<br>checked |
|---------------------------------|------------------------|-----------------------|----------------------------|--------------------------|
| `bootloader`                    | Preserved              | **WIPED, NOT WRITTEN** | Rewritten                  | Rewritten (after erase)  |
| `partition-table`               | Preserved              | **WIPED, NOT WRITTEN** | Rewritten                  | Rewritten (after erase)  |
| `nvs` (Wi-Fi, settings)         | Preserved              | Erased                | Preserved                  | Erased                   |
| `otadata`                       | Reset                  | Reset                 | Reset                      | Reset                    |
| `phy_init` (RF calibration)     | Preserved              | Erased                | Preserved                  | Erased                   |
| `ota_0` (running app)           | New firmware           | New firmware          | New firmware               | New firmware             |
| `ota_1` (other app slot)        | Preserved (old fw)     | Erased                | Preserved (old fw)         | Erased                   |
| `littlefs` (layouts, fonts)     | Preserved              | Erased                | Preserved                  | Erased                   |

\* **Firmware Update + erase checkbox checked = brick.** The chip gets erased,
but the Firmware Update manifest doesn't include the bootloader, so the device
boots into 0xFF flash and prints `invalid header: 0xffffffff` forever. Recover
with Full Recovery (either checkbox state works). The on-page guide tells users
to leave the box unchecked for Firmware Update.

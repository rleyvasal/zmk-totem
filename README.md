# ZMK Totem

Firmware for the Totem split keyboard: generic [ZMK Next](https://github.com/rleyvasal/zmk-next)
plus a small set of Totem-specific patches. Nothing here is meant for other keyboards; use
ZMK Next directly instead.

```
zmkfirmware/zmk  ->  rleyvasal/zmk-next  ->  rleyvasal/zmk-totem (this repo, branch totem)  ->  rleyvasal/totem-zmk-config
```

- **`totem`** (default branch) is ZMK Next plus the patches below.
- Every Monday, totem-zmk-config's _Bump ZMK_ workflow merges the latest ZMK Next release into
  `totem`, runs the Totem firmware tests, pushes it here, and opens a PR in totem-zmk-config that
  moves the firmware pin. The keyboard only changes when that PR is flashed and merged.
- Most behaviour is switched by `CONFIG_TOTEM_*` options defined in totem-zmk-config's `Kconfig`
  and set in its `config/totem.conf`. With those options off, the patched code behaves like
  ZMK Next, apart from the always-on changes marked below.

## Totem patches

| Area                           | Files                                                                                         | What it does                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ------------------------------ | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Host advertising control       | `app/src/ble.c`                                                                               | All advertising changes run on one work queue with a versioned target, so a profile switch or USB change mid-operation never leaves the keyboard advertising for the wrong host. Adds: pausing advertising after the selected host has been away (`TOTEM_ADV_THROTTLE`, a key press resumes it), faster advertising after a profile switch (`TOTEM_ADV_BOOST`), only the selected bonded host may connect (`TOTEM_ACTIVE_ADV_FILTER`, Filter Accept List), optional directed advertising first (`TOTEM_DIR_THEN_OPEN`), retrying when advertising can't start, and pressing the active `BT_SEL` again to force a reconnect (`TOTEM_RESELECT_RECONNECT`). |
| Profile handoff                | `app/src/ble.c`                                                                               | With exclusive-host mode, switching profiles disconnects the other bonded computer before advertising for the new one, and waits for unresolved private addresses to resolve first.                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Private address (RPA) handling | `app/src/ble.c`                                                                               | Matches a computer that connects with a rotating private address to its saved profile, so macOS still counts as "the active host" and gets keystrokes. Always on.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Stale connections              | `app/src/ble.c`                                                                               | Only a fully connected link counts as the active host; a half-torn-down connection object is skipped. Always on.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Passkey entry                  | `app/src/ble.c`, `app/Kconfig`                                                                | Collects exactly six digits, Backspace deletes, Escape cancels, Enter submits; a failed or cancelled pairing is cleaned up.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| USB takes over from Bluetooth  | `app/src/endpoints.c`, `app/src/ble.c`                                                        | While the cable is in, keystrokes never fall back to Bluetooth and the left half stops advertising to computers (macOS drops the Bluetooth keyboard when the same keyboard appears on USB). The split link to the right half keeps running. Always on for the central half.                                                                                                                                                                                                                                                                                                                                                                              |
| Keyboard notifications         | `app/src/hog.c`                                                                               | Keeps Zephyr's managed CCC handling (bond-persisted subscriptions) and adds hooks so diagnostics can see subscriptions and report delivery.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Battery                        | `app/src/battery.c`, `app/include/zmk/battery.h`                                              | Remembers the last battery reading taken while unplugged (USB holds the cell at charge voltage, which reads as 100%), saves it, and exposes it with the raw millivolts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Split battery for hosts        | `app/src/split/bluetooth/central_bas_proxy.c`, `app/src/split/bluetooth/central.c`            | Publishes the left half on an extra battery service and the right half on the main one (the layout Mighty Mitts expects), rate-limited, and keeps the right half's last value across a split disconnect instead of reporting 0%. Assumes one peripheral.                                                                                                                                                                                                                                                                                                                                                                                                 |
| Startup ordering               | `app/src/main.c`, `app/include/zmk/settings.h`, `app/Kconfig`                                 | Adds a `zmk_settings_loaded()` hook after settings load, moves battery saves to the low-priority queue, and enlarges that queue's stack, so flash writes can't stall Bluetooth at boot.                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Studio diagnostics channel     | `app/src/studio/rpc.c`, `app/src/studio/uart_rpc_transport.c`, `app/include/zmk/studio/rpc.h` | Lets totem-zmk-config send framed log and battery lines over the Studio USB serial port without blocking (`zmk_rpc_tx_raw_payload`), accepts `C`/`D` control frames for live logging and log dumps, and re-arms the serial port after USB finishes enumerating.                                                                                                                                                                                                                                                                                                                                                                                          |
| Tests and CI                   | `app/tests/...`, `.github/workflows/...`                                                      | Host-side tests for the above (`test_active_connection.py`, `test_advertising_owner.py`, `test_keyboard_ccc.py`, `test_profile_notifications.py`, `test_raw_tx.py`, `test_settings_loaded.py`, `test_persistence.py`) and a passkey simulator test.                                                                                                                                                                                                                                                                                                                                                                                                      |

Hooks used by totem-zmk-config's own modules: `zmk_ble_totem_*()` and
`zmk_ble_advertising_observed()` in `zmk/ble.h`, `zmk_hog_*()` in `zmk/hog.h`,
`zmk_settings_loaded()`, `zmk_battery_last_cell_soc()`, `zmk_rpc_tx_raw_payload()` and
`zmk_studio_control_payload()`.

---

# ZMK Next

ZMK Next is a firmware runtime-configuration platform built on ZMK's hardware,
transport, and behavior foundations. It makes supported behavior _instances_
live-configurable while retaining compiled firmware engines and fixed resource
budgets.

The protocol contract lives in `zmk-next-messages`; the editor and device client
live in `zmk-next-configurator`. This repository owns firmware validation,
runtime dispatch, safe activation, and persistent configuration generations.

The initial architecture is documented in
[`docs/architecture/runtime-config.md`](docs/architecture/runtime-config.md).

## Try it on a keyboard

Your existing config repo needs two edits:

1. Point the `zmk` project in `config/west.yml` at a ZMK Next release tag.
2. Add `snippet: zmk-next` to the keyboard's USB/central build in `build.yaml`.
3. Build and flash the firmware as usual.
4. Run the downloadable ZMK Next Configurator locally and connect in a
   Web Serial-capable browser.

See the [generic integration guide](docs/quick-start.md) for copy-and-paste
examples for both single-piece and split keyboards, plus the few ZMK Studio
prerequisites that firmware cannot infer automatically.

ZMK Next is based on [ZMK Firmware](https://zmk.dev/), the MIT-licensed keyboard
firmware built on the [Zephyr Project](https://www.zephyrproject.org/) RTOS. The
upstream documentation and community remain the reference for its compiled
hardware, transport, and behavior foundations.

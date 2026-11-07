# Project summary: adman234/kiln-controller

A record of the work done on this fork of
[jbruce12000/kiln-controller](https://github.com/jbruce12000/kiln-controller),
and notes for anyone (person or AI) picking it up later.

- **Target hardware:** Raspberry Pi Zero W (v1, armv6, 512 MB), Raspberry Pi OS
  Bookworm or Trixie.
- **User-facing change list:** [CHANGES.md](CHANGES.md).
- **Install guide:** [docs/install.md](docs/install.md).
- **Safety behaviour:** [docs/safety.md](docs/safety.md).

---

## 1. Timeline of the work

| PR | What |
| --- | --- |
| #1 | Big modernization (see section 2): bug and security fixes, settings in the web UI, °C/°F, autotune, learned cost estimates, more sensor boards, schedule library and editor, delayed start, SSE instead of websockets, phone UI, backup/restore, zero-touch install, SD card protection, docs, tests, CI |
| #2 | Zero-touch install fix: Wi-Fi didn't connect because cloud-init runs vendor-data *before* Imager's own setup. vendor-data now installs a one-shot `kiln-firstboot.service` that runs after Imager's setup has finished |
| #3, #4 | Current sensor (CT clamp on an ADS1115), software updates from the web UI, first round of safety defaults (from a parallel session) |
| #5 | **All safety features removed** at the user's request, so they could be reviewed and approved one by one |
| #6 | **Approved safety features re-added**, plus the setup wizard, the SETUP button and the GPIO pin map (section 3) |
| #7 | Setup wizard no longer reopens on every page load (the reload cancelled the "setup done" request on a Pi Zero). Safer graph editing: fixed axes, click to add a point, select and delete points |
| — | GPL-3.0 `LICENSE` file added (matching upstream) |
| #8, #9 | Optional **public read-only status page** (`tools/status-server.py`, port 8082) for sharing through a reverse proxy; [docs/public-status.md](docs/public-status.md) covers Nginx Proxy Manager and Cloudflare |

Work happened on branches `claude/exciting-thompson-vhlbwc` and
`claude/compassionate-carson-910b53`. Each was merged to `main` through a PR.

---

## 2. What the modernization added (PR #1 and follow-ups)

### Architecture
- **Web server:** Python 3, bottle 0.13 on a gevent WSGIServer (monkey-patched).
- **Live status:** a **Server-Sent Events** stream at `/api/events`. Everything else is a JSON API ([docs/api.md](docs/api.md)). The old websockets are gone.
- **Frontend:**
  - jQuery 3.7.1, Bootstrap 3.4.1 and flot 0.8.3, all served locally, no CDN.
  - `public/assets/js/picoreflow.js` is the main UI.
  - `pins.js` is the GPIO header picker.
  - `setup.js` is the wizard.
- **Settings:**
  - `config.py` holds only the defaults.
  - Changes made in the UI are saved to `storage/settings.json`, which overrides them.
  - `lib/settings.py` `SCHEMA` drives the Settings UI: groups, kinds, help text and restart flags.
- **Units:** temperatures are **Celsius internally** and converted only for display (`lib/units.py`). PID gains are converted for old Fahrenheit configs.
- **Modules:** split into `lib/` — `oven.py`, `sensors.py`, `settings.py`, `profiles.py`, `history.py`, `autotune.py`, `notify.py`, `current.py`, `limits.py`, `sdnotify.py`, `updater.py`, `units.py` and `publicstatus.py`.
- **Hardware access:** Blinka is imported lazily, so simulations run on any computer.

### Features
- Settings, diagnostics and a relay test in the web UI.
- One-click °C/°F switch.
- **PID autotune** in the GUI (relay / Åström–Hägglund method, four tuning rules, one-click apply).
- **Cost estimates** that learn from previous firings. Only the last 100 short records are kept.
- **Sensors:** MAX31855, MAX31856 (all thermocouple types), MAX6675, MCP9600/9601 (I2C) and MAX31865 (RTD).
- **Schedule library:** created/modified dates, notes, search, sort, edit, copy, rename, delete, import and export.
- **Editor:** minutes, hours or h:mm; editable ramp rates; holds; a "ramp to, hold" builder; drag points with mouse or finger.
- **Delayed start** ("start in 6 h" / "start at 5:00"), which survives reboots.
- Backup and restore. Backups never include secrets.
- Phone-friendly layout and "Add to Home Screen".
- Phone alerts via ntfy, Pushover or a webhook, with a test button. Deduplicated to one alert per key per 15 minutes.
- Software updates from the web UI, with roll back.
- Optional password. Cross-origin requests are blocked. A warning shows if the controller is reachable from the internet without a password.

### Bugs and vulnerabilities fixed
Full list in CHANGES.md. The main ones:
- Path traversal through schedule names.
- Stored XSS through schedule names.
- Cross-site control of the kiln from other web pages.
- A corrupt `state.json` blocked startup.
- Restart after a power cut used the wrong temperature units.
- PID divide-by-zero and derivative kick.
- Cost was calculated wrongly.
- gevent was used without monkey patching.

### Installation
- **Zero-touch install:**
  1. Flash the card with Raspberry Pi Imager.
  2. Copy `provision/vendor-data` to the boot partition. Optionally also copy `kiln-install.conf` and `kiln-settings.json`.
  3. Boot. It installs itself and shows a progress page.

  It uses cloud-init NoCloud `seedfrom: file:///boot/firmware`. See "Wi-Fi troubleshooting" in docs/install.md.
- **Manual install or update:** `install.sh` (apt gevent, so nothing is compiled on a Zero; venv; SPI/I2C; systemd service; journald cap; Wi-Fi power-save off). Options: `--protect-sd`, `--tailscale`, `--settings FILE`, `--no-start`, `--branch`.
- **Re-running a zero-touch install on a card that has already booted:** bump the instance-id (e.g. add `-2`) in `meta-data` and `cmdline.txt` so cloud-init runs again.

---

## 3. Safety features (approved one by one, PR #6)

The general rule: **alert first, and only stop the firing if the problem is
still there after the safety wait** (`safety_grace_minutes`, default 5).
`Oven.raise_issue()` and `clear_issue()` in `lib/oven.py` implement this. The
UI shows active issues in a red banner with a countdown to the stop.

| Feature | Behaviour | Default |
| --- | --- | --- |
| Emergency shutoff temperature | User-set. Stops the firing at once. Warns when a schedule's peak is within 50°C (90°F) | on |
| MAX6675 limit (1023°C) | Schedules above it are refused (UI, API, delayed start, autotune). Warning within 50°C | always |
| Stuck relay (temperature rises with the elements off) | Alert, stop after the wait | 20°C in 15 min |
| No heat (elements at full power, no rise) | Alert, stop after the wait | <3°C in 45 min |
| Thermocouple dropout | Elements **held off** and schedule waits. Alert after 30 s, stop after the wait | on |
| Behind schedule | Alert only | 60 min |
| Software error in the control loop | Alert only, the firing carries on | on |
| Software watchdog (systemd `WatchdogSec=90`) and hardware watchdog (`RuntimeWatchdogSec=15s`) | Restart and resume with an alert. A second restart within the wait stops the firing | on |
| Safety contactor output (`gpio_contactor`) | Closed only while heating is possible. Closes 0.3 s before the SSR switches | -1 (none) |
| Heartbeat output | **Removed** | — |
| CT pre-fire check | 1 s pulse at start (and at a delayed start's start time). No current → screen flashes, alert, firing waits. Buttons: Try again / Ignore current sensor / Cancel | on |
| CT: no current while firing | Alert after 60 s of on-time. Optional stop | alert only |
| CT: stuck SSR (current with elements off) | Alert after 60 s. Optional stop | alert only |
| CT: low current | Warning below a set value | off (0 A) |

### Setup wizard and SETUP button
- **First launch:** the wizard opens when `setup_done` is false; "Skip – use defaults" closes it. Steps: units/cost, sensor board and pins, relay (yes/no and pin), contactor, current sensor, emergency temperature, alerts, review.
- **SETUP button (top right):**
  - Re-runs the wizard.
  - Or resets to defaults. Before resetting it shows the list from `/api/settings/reset-preview` and asks for confirmation.
- **Pin picker:**
  - Clickable 40-pin header (26-pin for the original Model A/B).
  - The Pi model is read from `/proc/device-tree/model` via `/api/board`.
  - It blocks conflicting pins, bus pins and the HAT EEPROM pins.
  - The same picker appears under Settings → Hardware and Settings → Safety.

### Decisions made along the way (the user may want to revisit)
- During a thermocouple dropout the elements stay off. This replaces the original "stop after too many thermocouple errors".
- The MAX6675 block also applies in **simulation**. The `config.py` default board is `max6675`, so hot schedules are refused in simulation until the board setting is changed.
- If you run the wizard on a computer that isn't a Pi, simulation mode stays on.

### Open question for the user (not yet answered)
Two items from the original safety list were **not** approved, so they are
not implemented:
1. Switch the relay off when the controller service stops or crashes (an `ExecStopPost` that drives the pin low).
2. Send an alert when a firing stops unexpectedly.

---

## 4. Hardware notes

### Current sensor
The user has a **YHDC SCT-013-050 (50A/1V)** clamp on a 3.5 mm-jack breakout
(pads marked TIP, RING1, RING2, SLEEVE).
- The Pi has no analog input, so an **ADS1115** ADC is required.
- The breakout is only a connector; it doesn't measure anything.
- The -050 has a built-in burden resistor, so **no 33 Ω resistor** is needed.

Wiring:
- **ADS1115 to the Pi:**
  - VDD → pin 1 (3.3V)
  - GND → pin 6
  - SCL → pin 5
  - SDA → pin 3
  - ADDR → GND (address 0x48)
- **Clamp to the ADS1115:**
  - TIP → A0
  - SLEEVE → A1
  - Bias A1 to 1.65 V with 10 kΩ to 3.3V, 10 kΩ to GND, and 10 µF to GND.
  - RING1 and RING2 are unused.

Settings:
- Input: *A0 – A1 (differential)*
- Calibration: **50** A/V
- Mains voltage: as wired

Clamp it around **one** element conductor only; with both inside, the currents cancel. Details: [docs/current-sensor.md](docs/current-sensor.md).

### Other wiring
- [docs/wiring-pi-zero-max6675.md](docs/wiring-pi-zero-max6675.md) covers a Pi Zero W with a MAX6675 and SSRs, including the dual-D4184 trigger board.
- IRF520N modules don't switch reliably at 3.3 V.

---

## 5. Notes for continuing the project

### Getting a Pi up to date
- Code-only changes install from **Settings → Advanced → Software update**.
- Changes to system files (the service file, the watchdog config, apt packages) need **`./install.sh`** over SSH. PR #6 was one of these: Type=notify service and hardware watchdog.

### Development setup and checks (run before every push)
```bash
python3 -m pytest -q -p no:logging            # repo root; conftest.py adds lib/
python3 -m pyflakes kiln-controller.py lib tools kiln-logger.py kiln-tuner.py
shellcheck -S warning install.sh start-on-boot provision/vendor-data
node -e "new Function(require('fs').readFileSync('public/assets/js/picoreflow.js','utf8'))"
```
CI (`.github/workflows/ci.yml`) runs:
- Tests on Python 3.11 and 3.13.
- pyflakes.
- A server smoke test.
- shellcheck.
- A JS syntax check of `picoreflow.js`, `state.js` and `status.js`.

**To do:** add `pins.js` and `setup.js` to the CI JS syntax check.

### Running the simulator locally
- Run `python3 kiln-controller.py` and open http://localhost:8081. With no Pi, it runs in simulation mode.
- To speed it up, put `{"sim_speedup_factor": 10}` in `storage/settings.json`.
- To test the current-sensor fault screens, set `ct_sensor` to `ads1115` in simulation. That uses `SimulatedCt`. Then set `SimulatedCt.fault` to `"no_current"` or `"stuck"`, e.g. from a small wrapper script that patches `current.SimulatedCt.__init__`.
- For browser tests, Playwright with Chromium worked well. Test both desktop and phone (375 px) widths.

### Gotchas learned the hard way
- **Static files:** Bottle's `static_file(..., mimetype=True)`, not `"auto"`.
- **flot:** needs its CSS loaded before the scripts.
- **Stacked Bootstrap 3 modals:** need a z-index fix. The power-check overlay sits at z-index 1035, below the modals, so the "cancel firing?" confirm shows above it.
- **cloud-init:** runs vendor-data scripts before Imager's `runcmd`, so anything needing Wi-Fi or the user account must be deferred (that's why `kiln-firstboot.service` exists).
- **Pi Zero reloads:** a page reload right after a POST can cancel the request. Wait for the response first (this was the wizard bug fixed in PR #7).
- **Tests:** must not write into the real `storage/`. The sim fixture patches `RESTARTS_FILE` and `LAST_EXIT_FILE`.
- **Killing test servers:** don't use `pkill -f` with a pattern that also matches your own shell command line.
- **Changing temperature settings:** the server reads temperatures posted to `/api/settings` in the scale it had *before* that save. The wizard converts the emergency temperature accordingly.

### Git and PR conventions used so far
- Work on a `claude/...` branch from `origin/main`. Open a PR and merge it into `main` with a merge commit.
- If a branch's PR is already merged, restart the branch from `origin/main` before new work.
- Commit messages describe the change in plain words. The PR body lists what changed and how it was tested.

### Ideas not yet done
- The two unapproved safety items in section 3.
- Test on real hardware: pre-fire check with the real CT, contactor timing, watchdog restarts (`systemctl kill -s SIGSTOP kiln-controller` simulates a freeze).
- Calibrate the CT against a clamp meter, then set *Elements count as on above* and *Low current*.

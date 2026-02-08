# Klipper Router + LED Instance Guide (E3V3SE)

This document explains how to replicate this setup on another printer.

Goal:
- Keep main printer macros mostly untouched.
- Run LED effects/status in a separate Klipper instance.
- Use `klipper-router` to bridge and subscribe status from main -> LED.
- Keep manual LED test buttons available on main.

## Architecture

Components:
- `main` Klipper instance: your real printer.
- `led` Klipper instance: standalone LED controller (separate MCU).
- `klipper-router`: JSON-RPC bridge between instances.

Data flow:
1. Main and LED connect to router sockets.
2. LED subscribes to main status objects (`print_stats`, `idle_timeout`, etc.).
3. Router forwards object updates to LED callback macro.
4. LED callback selects a status macro/effect.
5. Main also has manual bridge macros to directly trigger LED macros for testing.

## Repository Layout

Main instance files:
- `e3v3se/klipper/01_router_api.cfg`: wrapper macros for router RPC methods.
- `e3v3se/klipper/02_router_led_hooks.cfg`: startup baseline LED hook.
- `e3v3se/klipper/03_router_led_bridges.cfg`: manual bridge/test buttons.
- `e3v3se/printer.cfg`: includes `klipper/*.cfg`.

Router files:
- `e3v3se/router/router.cfg`: router instance definitions and sockets.

LED instance files:
- `e3v3se/router/instances/led/printer.cfg`: LED instance root config.
- `e3v3se/router/instances/led/klipper/10_disco_effects.cfg`: disco effects.
- `e3v3se/router/instances/led/klipper/20_status_macros.cfg`: status macros/loops.
- `e3v3se/router/instances/led/klipper/30_router_event_subscriptions.cfg`: object subscriptions and status arbitration.

## Prerequisites

1. `klipper-router` installed and runnable on the host.
2. Two Klipper sockets available:
- Main socket (example): `/home/pi/printer_data/comms/klippy.sock`
- LED socket (example): `/tmp/klippy_led_uds`
3. LED controller connected to its own MCU and LED pin configured.

## Step 0: Install klipper-router

Install into `~/klipper-router`:

```bash
cd /home/pi
git clone https://github.com/paxx12/klipper-router.git ~/klipper-router
```

Optional sanity check:

```bash
/home/pi/klippy-env/bin/python /home/pi/klipper-router/src/klipper_router.py --help
```

## Step 1: Configure Router

File: `e3v3se/router/router.cfg`

Example:
```ini
[klippy main]
sock: /home/pi/printer_data/comms/klippy.sock
on_connect: M118 Router connected to main

[klippy led]
sock: /tmp/klippy_led_uds
on_connect: M118 Router connected to LED

[router]
default_instance: main
```

## Step 2: Main Klipper Setup

1. Include `klipper/*.cfg` from `printer.cfg` (already done in this repo).
2. Load router wrappers: `e3v3se/klipper/01_router_api.cfg`
3. Load manual LED bridge macros: `e3v3se/klipper/03_router_led_bridges.cfg`
4. Optional startup baseline:
File `e3v3se/klipper/02_router_led_hooks.cfg`
```ini
[delayed_gcode startup_status_light]
initial_duration: 3.0
gcode:
    DISCO_STOP
    DISCO_IDLE
```

Manual test examples from main console:
- `DISCO_PARTY`
- `DISCO_BREATHING`
- `DISCO_STOP`
- `STATUS_READY`
- `STATUS_HEATING`
- `STATUS_LEVELING`

## Step 3: LED Klipper Setup

File: `e3v3se/router/instances/led/printer.cfg`

Key points:
- `kinematics: none`
- include LED macro files:
```ini
[include klipper/*.cfg]
```

The current LED instance uses:
- status/effects macros in `10_disco_effects.cfg` and `20_status_macros.cfg`
- subscriptions/arbitration in `30_router_event_subscriptions.cfg`

## Step 4: Subscribe to Main Status Objects

File: `e3v3se/router/instances/led/klipper/30_router_event_subscriptions.cfg`

Core macro:
- `SUBSCRIBE_MAIN_STATUS` calls:
  - `router/objects/subscribe`
  - `target="main"`
  - `objects={...}`
  - `gcode_callback="ON_STATUS_UPDATE"`

Subscribed objects in this setup:
- `webhooks.state`
- `print_stats.state`
- `pause_resume.is_paused`
- `idle_timeout.state`
- `toolhead.homed_axes`
- `probe.last_z_result`
- `extruder.temperature,target`
- `heater_bed.temperature,target`

Startup/reconnect behavior:
- `ROUTER_ON_READY` schedules subscription.
- `ROUTER_ON_CONNECTED` (for `main`) re-subscribes and syncs state.
- `LED_SYNC_MAIN_STATUS_ONCE` does a one-shot `router/objects/query` to avoid stale startup LED.

## Step 5: Status Arbitration Model

`ON_STATUS_UPDATE`:
1. Caches last-known values (important because updates can be sparse).
2. Computes one `desired` state.
3. Applies LED macro only when state changes (`last_status` guard).

Current mapping (summary):
- `error` -> `STATUS_ERROR`
- paused -> `STATUS_PAUSED`
- printing -> `STATUS_RUNNING`
- complete -> `STATUS_COMPLETE`
- cancelled -> `STATUS_WARNING`
- heating delta -> `STATUS_HEATING`
- active probe updates -> `STATUS_LEVELING`
- partial homed axes -> `STATUS_HOMING`
- `idle_timeout=idle` -> `DISCO_IDLE`
- standby/ready -> `STATUS_READY`

Disconnect fallback:
- If main disconnects, delayed fallback can set breathing (`DISCO_BREATHING`) after timeout.
- Reconnect cancels fallback.

## Step 6: Persistent Status Effects

File: `e3v3se/router/instances/led/klipper/20_status_macros.cfg`

This setup uses looped effects for:
- `STATUS_HEATING`
- `STATUS_HOMING`
- `STATUS_LEVELING`

All loops are stopped by `_STATUS_STOP_LOOPS` when changing to another state.

## Bring-Up / Restart Order

Recommended:
1. Restart `klipper-router`.
2. Restart LED Klipper instance.
3. Restart main Klipper instance.

Why:
- Clears stale router subscriptions.
- Ensures `ROUTER_ON_CONNECTED` paths run cleanly.

## Systemd Services

Repository copies:
- `e3v3se/klipper-led.service`
- `e3v3se/klipper-router.service`

Create the LED Klipper service at `/etc/systemd/system/klipper-led.service`:

```ini
[Unit]
Description=Klipper LED Instance
After=network.target

[Service]
Type=simple
User=pi
RemainAfterExit=yes
ExecStart=/home/pi/klippy-env/bin/python /home/pi/klipper/klippy/klippy.py /home/pi/printer_data/config/router/instances/led/printer.cfg -a /tmp/klippy_led_uds -l /home/pi/printer_data/logs/klippy_led.log
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Create the router service at `/etc/systemd/system/klipper-router.service`:

```ini
[Unit]
Description=Klipper Router
After=klipper.service klipper-led.service

[Service]
Type=simple
User=pi
ExecStart=/home/pi/klippy-env/bin/python /home/pi/klipper-router/src/klipper_router.py -c /home/pi/printer_data/config/router/router.cfg
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Install from this repo on the host:

```bash
sudo cp /home/pi/printer_data/config/klipper-led.service /etc/systemd/system/klipper-led.service
sudo cp /home/pi/printer_data/config/klipper-router.service /etc/systemd/system/klipper-router.service
```

If your config path is not `/home/pi/printer_data/config`, adjust those `cp` source paths accordingly.

Enable/start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable klipper-led.service klipper-router.service
sudo systemctl restart klipper-led.service klipper-router.service
```

Check status/logs:

```bash
systemctl status klipper-led.service klipper-router.service
tail -f /home/pi/printer_data/logs/klippy_led.log
journalctl -u klipper-router.service -f
```

## Validation Checklist

1. Manual bridge:
- Run `DISCO_PARTY` on main, LED should start party effect.

2. Startup baseline:
- LED should settle to `DISCO_IDLE` after startup hook.

3. Print lifecycle:
- During print: `STATUS_RUNNING`.
- Pause: `STATUS_PAUSED`.
- Heating: pulsing heating effect.
- Mesh/probing: leveling/homing effect.
- After complete: completion then ready/idle transitions.

4. Idle:
- After idle timeout: `DISCO_IDLE`.

## Troubleshooting

### Duplicate callbacks / repeated actions
Cause:
- Multiple active subscriptions in router memory.
Fix:
1. Restart `klipper-router`.
2. Restart both Klipper instances.

### LED stays initial red after LED restart
Cause:
- No immediate state apply.
Fix:
- Keep `LED_SYNC_MAIN_STATUS_ONCE` query and `last_status="unknown"` startup behavior.

### Heating flashes briefly then exits
Cause:
- One-shot effect or immediate state override.
Fix:
- Use looped `STATUS_HEATING`.
- Ensure arbitration updates only on desired state change.

### Breathing appears while printer is still on
Cause:
- Disconnect transient interpreted as shutdown.
Fix:
- Use delayed disconnect fallback and cancel on reconnect.

### Blue (`STATUS_READY`) dominates probing/mesh
Cause:
- No persistent homing/leveling loop or missing probe signal mapping.
Fix:
- Subscribe `probe.last_z_result`.
- Keep persistent `STATUS_LEVELING` / `STATUS_HOMING`.

## Design Rules (Recommended)

1. Keep `main` side simple:
- router wrappers + manual bridge buttons.

2. Put automation logic on LED side:
- object subscriptions + state machine.

3. Avoid owning core main macros for this feature:
- prefer router subscriptions over invasive macro rewrites.

4. Keep manual testing macros:
- they are invaluable for bring-up and troubleshooting.

## Porting to Another Printer

Minimum required changes:
1. Update router socket paths in `router/router.cfg`.
2. Update LED MCU serial/pin in `router/instances/led/printer.cfg`.
3. Include router wrappers + bridge macros in main config.
4. Include LED macro files in LED instance.
5. Restart in the recommended order and validate.

Once those are correct, the same pattern should work across stock Klipper, vendor forks, and mixed environments.

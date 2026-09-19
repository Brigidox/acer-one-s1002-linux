# Acer One S1002 (N15P2) on Linux

Fixes for three hardware quirks on the **Acer One S1002** — a Bay Trail Atom
(Z36xx/Z37xx) detachable tablet, chassis code **N15P2** — running Linux.

Developed and tested on [Omarchy](https://omarchy.org/) (Arch + Hyprland),
but the root causes are generic (kernel module load order, udev, an
accelerometer via `iio-sensor-proxy`) and apply to this hardware on **any**
Linux distro. The Hyprland-specific bits are clearly marked so you can adapt
them to your own compositor/WM.

If you own this exact tablet and Linux "mostly works" except for these three
things, this is probably why.

## Contents

1. [Screen brightness does nothing](#1-screen-brightness-does-nothing) — kernel module load order
2. [No on-screen keyboard when the dock keyboard is detached](#2-no-on-screen-keyboard-when-detached) — squeekboard + udev
3. [Screen doesn't rotate with the device](#3-screen-doesnt-auto-rotate) — iio-sensor-proxy + Hyprland

---

## 1. Screen brightness does nothing

Brightness keys / `brightnessctl` have no effect on the actual screen. The
only backlight device present is `acpi_video0` (generic ACPI backlight), and
writing to it does nothing visible.

**Cause:** the panel is DSI-connected. Its backlight is driven by the Intel
LPSS PWM controller (`pwm_lpss_platform`, ACPI id `80860F09`), but that kernel
module isn't in the default initramfs. It loads too late — *after* `i915`
already tried to grab it once during early KMS and failed permanently (it
doesn't retry). Confirm with:

```
journalctl -k -b | grep -i "pwm\|i915"
# -> i915 0000:00:02.0: [drm] *ERROR* [CONNECTOR:...:DSI-1] Failed to get the SoC PWM chip
```

**Fix:** get `pwm_lpss_platform` into the initramfs, *before* `i915` runs.

On mkinitcpio-based distros (Arch and derivatives):

```bash
sudo install -m644 mkinitcpio/zz-pwm-backlight.conf /etc/mkinitcpio.conf.d/
sudo mkinitcpio -P
sudo reboot
```

The `zz-` prefix matters if your `mkinitcpio.conf.d/` has anything else in it
that sets `MODULES=(...)` (full overwrite) rather than `MODULES+=(...)`
(append) — drop-ins are applied in filename order, so this one needs to sort
last to survive. Using `MODULES+=` inside it is what makes it additive rather
than replacing whatever ran before it.

On dracut/other initramfs tools: just make sure `pwm_lpss_platform` (and its
dependency `pwm_lpss`) are force-included, however your tool does that
(e.g. dracut's `add_drivers+=` in a `/etc/dracut.conf.d/*.conf` file).

**Verify after reboot:**

```bash
ls /sys/class/backlight/          # should now show intel_backlight
journalctl -k -b | grep -i "pwm\|i915"   # the PWM error should be gone
brightnessctl -d intel_backlight set 50%  # should visibly dim the panel
```

---

## 2. No on-screen keyboard when detached

Detach the screen from the keyboard dock and there's no way to type — no
on-screen keyboard shows up automatically.

**Fix:** [`squeekboard`](https://gitlab.gnome.org/World/Phosh/squeekboard) (the
Phosh/GNOME on-screen keyboard, works fine outside GNOME) kept running hidden
in the background, shown/hidden automatically by a **udev rule** that watches
the dock's USB connector, plus a manual toggle keybind as a fallback.

### Install

```bash
sudo pacman -S squeekboard          # or your distro's package
sudo install -m755 scripts/omarchy-osk-set scripts/omarchy-osk-toggle scripts/omarchy-osk-dock-event /usr/local/bin/
sudo install -m644 udev/99-omarchy-osk-dock.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
```

Then autostart `squeekboard` hidden, and bind a manual toggle key — see
[`hypr/autostart.lua.snippet`](hypr/autostart.lua.snippet) and
[`hypr/bindings.lua.snippet`](hypr/bindings.lua.snippet) for the Hyprland way;
for any other WM/DE, just autostart `squeekboard` and bind a key to run
`omarchy-osk-toggle`.

### How it works / non-obvious gotchas

- **`squeekboard` doesn't show anything just by running.** It only creates its
  on-screen surface in response to a DBus call: `sm.puri.OSK0.SetVisible(bool)`
  on the session bus (property `Visible` is readable the same way). That's
  what `omarchy-osk-set show|hide` does.

- **The USB device identifying your dock's keyboard will be different.**
  Find it with `journalctl -k -b | grep -i "usb.*disconnect\|new usb device"`
  while unplugging/replugging, or `udevadm info -a -p $(udevadm info -q path
  -n /dev/input/by-path/*-event-kbd)`. Edit the `idVendor`/`idProduct` (for
  the `add` rule) and the bus/port `KERNEL` name (for `remove`, see below) in
  `udev/99-omarchy-osk-dock.rules` accordingly. On this specific unit it's the
  internal hub `05e3:0608` ("USB2.0 Hub") at bus/port `1-2`, feeding a
  keyboard+touchpad device `06cb:73f4` ("ITE Tech. Inc. ITE Device(8910)").

- **The single biggest gotcha, and the reason this took a while to get
  right:** matching the `remove` rule on `ENV{ID_VENDOR_ID}`/`ENV{ID_MODEL_ID}`
  (the natural thing to try) *looks* correct — `udevadm test --action=remove
  <devpath>` will even confirm it matches — but it **silently never fires on
  a real unplug**. By the time udevd actually processes a real removal, the
  device's sysfs `descriptors` file is already gone, so udev's `usb_id`
  builtin importer (which derives `ID_VENDOR_ID`/`ID_MODEL_ID`) fails, and
  those variables are simply unset for that event. `udevadm test` is
  misleading here because it reads stale, cached values from the udev
  database instead of genuinely re-deriving them. The fix is to match
  `remove` on `KERNEL=="<bus-port>"` (e.g. `"1-2"`) instead — the fixed
  USB bus/port path, which comes straight from the kernel uevent itself and
  needs no sysfs lookup, so it's reliably present on both `add` and `remove`.

  If you ever need to re-diagnose a similar udev "add works, remove doesn't"
  mystery: `sudo udevadm control --log-level=debug`, reproduce, then
  `journalctl -u systemd-udevd --since ...` — look for `Failed to run
  builtin "usb_id"` lines. Revert with `sudo udevadm control --log-level=info`
  afterwards (debug is very verbose).

- `omarchy-osk-dock-event` runs as root (that's just how udev `RUN+=` works),
  so it has to explicitly `sudo -u <user>` into the desktop session with
  `XDG_RUNTIME_DIR`/`DBUS_SESSION_BUS_ADDRESS` set by hand — udev gives it
  none of that environment otherwise. It auto-detects the logged-in user via
  `loginctl`, assuming a single-user tablet setup.

---

## 3. Screen doesn't auto-rotate

No automatic rotation when you physically turn the tablet.

**Good news:** the accelerometer (`kxcjk_1013` driver, shows up as IIO device
`i2c-SMO8500:00`, ACPI id `SMO8500`) works out of the box with the mainline
kernel driver — no mount-matrix quirk needed on this unit.

### Install

```bash
sudo pacman -S iio-sensor-proxy     # or your distro's package
sudo systemctl enable --now iio-sensor-proxy.service
sudo install -m755 scripts/omarchy-auto-rotate /usr/local/bin/
```

Then autostart `omarchy-auto-rotate` — see
[`hypr/autostart.lua.snippet`](hypr/autostart.lua.snippet). It's a plain
shell script driving `hyprctl`, so it needs Hyprland; for other
compositors/WMs you'd swap the two `apply_transform()` lines in
[`scripts/omarchy-auto-rotate`](scripts/omarchy-auto-rotate) for whatever your
WM uses to set output/input-device transforms (`xrandr --rotate` +
`xinput --map-to-output` on X11, `wlr-randr` + a libinput calibration matrix
on other wlroots compositors, etc.) — the sensor-reading half stays the same.

### How it works / non-obvious gotchas

- `iio-sensor-proxy`'s `AccelerometerOrientation` DBus property stays
  `"undefined"` until *something* claims the accelerometer (to save power).
  `monitor-sensor --accel` (ships with iio-sensor-proxy) does that claiming
  for you and prints `orientation changed: <state>` on every change — that's
  what the script parses instead of talking DBus directly.

- Orientation-state → rotation mapping (the standard mutter /
  gnome-settings-daemon convention, using Wayland's `wl_output_transform`
  values: 0=normal, 1=90°, 2=180°, 3=270°):

  ```
  normal    -> 0
  left-up   -> 1
  bottom-up -> 2
  right-up  -> 3
  ```

  This was **empirically confirmed correct** on this unit by physically
  rotating through all four orientations and checking the panel content
  stayed upright each time — worth re-checking on your own unit in case your
  accelerometer's physical mounting differs (unlikely for the same model, but
  cheap to verify).

- Rotating the **output** without also rotating the **touchscreen input
  device** leaves you with an upright picture and touch coordinates that no
  longer match it. The script sets both in the same `apply_transform()` call.
  Find your touchscreen's device name with `hyprctl devices` (look under
  `Touch:`) — on this unit it's `ftsc9999:00-2808:5012`.

- **This Hyprland fork (Omarchy) configures via Lua and doesn't support the
  classic `hyprctl keyword ...`** — it errors with "keyword can't work with
  non-legacy parsers. Use eval." The live-reconfiguration mechanism here is
  `hyprctl eval '<lua>'`, calling the `hl.*` API (`hl.monitor{...}`,
  `hl.device{...}`, etc.) documented in
  `/usr/share/hypr/stubs/hl.meta.lua` — check that file first if you're
  adapting this and something errors with a Lua parse message.

---

## Hardware reference (this exact unit)

| Component | Identifier |
|---|---|
| SoC | Intel Atom Bay Trail (Z36xx/Z37xx), `valleyview` gen |
| Display | DSI panel, `1280x800`, output name `DSI-1` |
| Backlight PWM | Intel LPSS, ACPI `80860F09`, module `pwm_lpss_platform` |
| Accelerometer | Kionix, driver `kxcjk_1013`, ACPI `SMO8500` |
| Dock keyboard hub | USB `05e3:0608` ("USB2.0 Hub") at bus/port `1-2` |
| Dock keyboard+touchpad | USB `06cb:73f4` ("ITE Tech. Inc. ITE Device(8910)") |
| Touchscreen | `ftsc9999:00-2808:5012` (FocalTech) |

## License

MIT — do whatever you want with this, no warranty. See [LICENSE](LICENSE).

## Contributing

If you've got this same tablet (or a close sibling) and something here needs
tweaking for your unit, PRs / issues welcome.

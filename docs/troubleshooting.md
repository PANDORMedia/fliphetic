# Troubleshooting

This page collects the most common problems and how to diagnose them.

## Where to look first

| Source | Command |
|--------|---------|
| Dashboard service log | `sudo journalctl -u fliphetic.service -b` |
| A kiosk service log | `journalctl --user -u fliphetic-kiosk@playfield.service -b` |
| A specific load or stop | Open the app's page in the dashboard and read the run log. |
| Cabinet status | `fliphetic status` |

Every load and stop writes a full run log, visible on the app page. That is the
first place to look when a load fails.

## The dashboard is not reachable

* **Check the service.** `systemctl status fliphetic.service`. If it is
  restarting, read the journal.
* **Check the Tailscale address.** The dashboard binds to the address in
  `bind_host`. Run `tailscale ip -4` on the cabinet and confirm it matches
  `/etc/fliphetic/config.toml`.
* **Boot timing.** The service waits up to 30 seconds for the Tailscale
  interface. If Tailscale takes longer, the service fails its start condition
  and retries. It should recover on its own within a minute.
* **You are not on the tailnet.** The dashboard is only reachable from the
  cabinet's tailnet. Being on the same local network is not enough.

## A kiosk shows the wrong screen, has borders, or is misplaced

* Confirm the screen is registered with the correct geometry on the System
  page, and that the geometry matches `xrandr --query`.
* Confirm a kiosk service is enabled for that exact role
  (`systemctl --user is-enabled fliphetic-kiosk@<role>.service`).
* Restart the kiosk: `systemctl --user restart fliphetic-kiosk@<role>.service`.
* If a window is on the wrong monitor, the screen's position in the database is
  probably wrong. Re-read `xrandr` and correct it.

## Chromium shows a translate prompt or other popups

Chromium installed as a snap may ignore command line flags for some features. A
managed policy file disables them reliably. As root, create
`/var/snap/chromium/current/policies/managed/fliphetic.json`:

```json
{
  "TranslateEnabled": false,
  "DefaultBrowserSettingEnabled": false,
  "PromotionalTabsEnabled": false,
  "BrowserSignin": 0,
  "MetricsReportingEnabled": false
}
```

Restart the kiosk services. To check whether a policy took effect, point a
kiosk briefly at `chrome://policy`.

## A load fails

Open the app page and read the run log. Common causes:

* **`docker compose up failed`.** An image will not pull, a healthcheck never
  passes, or a published port collides. Try the app's Compose file by hand on
  the cabinet.
* **`cab has no esp32 device named X`.** The app targets a device name that is
  not registered. Register it on the System page, or the app can set
  `required = false`.
* **`ESP32 flash failed`.** Wrong chip, wrong binary, a bad USB cable, or the
  device is busy. See [ESP32 firmware](esp32-firmware.md).
* **`manifest invalid` or `references missing files`.** The `fliphetic.toml`
  is malformed or points at files that are not in the repository. Run
  `fliphetic validate .` in the repository.

## An ESP32 is not detected

* Confirm the board appears: `ls -l /dev/serial/by-id/`. If it does not, the
  cable may be charge only, or the board needs a driver.
* Confirm `flipper` is in the `dialout` group: `groups flipper`. If not,
  `sudo usermod -aG dialout flipper` and log in again.
* Register the device using its `/dev/serial/by-id/...` path, not
  `/dev/ttyUSB0`, so it survives reboots.

## The display layout is lost after a reboot

GNOME can detect outputs in a different order at boot. Use the
`restore-displays` autostart script described in
[Display and power](cab-display-power.md). It reapplies the layout with
`xrandr` at every login.

## The cabinet sleeps, blanks, or shows a lock screen

Re-apply the steps in [Display and power](cab-display-power.md). The most
important settings are `idle-delay 0`, the `logind` drop-in, and
`disable-lock-screen true`. If the cabinet ever locks and there is no keyboard,
you can recover over SSH.

## After a reboot the cabinet loads the wrong app

The cabinet auto loads the boot default if one is set, otherwise the last loaded
app. Check and set the boot default:

```sh
fliphetic default show
fliphetic default set <app-id>
```

## A renamed app still shows its old name

The stored display name is refreshed when an app is pulled or loaded. Click
**Pull latest** on the app page, or load the app, to pick up a changed
`app.name`.

## Resetting screens to the configuration file

Screens live in the database after first boot. To re-seed them from a legacy
`[[screens]]` block in `config.toml`, delete the rows from the `screens` table
in the SQLite database and restart the dashboard.

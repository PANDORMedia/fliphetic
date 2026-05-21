# Cabinet setup: display and power

A pinball cabinet is an appliance. It should never blank a screen, never sleep,
never show a lock screen, and it should keep its multi monitor layout across
reboots. This page configures all of that.

Run these steps in the `flipper` graphical session (or over SSH with the
display environment set, as shown).

## 1. Confirm the display layout

Log in to the desktop, arrange the three screens the way you want in GNOME
Settings, then read the resulting geometry:

```sh
xrandr --query | grep " connected"
```

You will see lines like:

```
DP-0 connected primary 1920x1080+240+0 ...
HDMI-0 connected 1920x1080+240+1080 ...
HDMI-1 connected 2160x3840+0+2160 left ...
```

Each line gives the output name, the resolution, the position (`+x+y`), and the
rotation. Record these. You register them in the dashboard later, on the
[Configuration](configuration.md) page.

## 2. Make the layout survive reboots

GNOME sometimes loses a multi monitor arrangement when outputs are detected in a
different order at boot. To make the layout deterministic, add a small script
that reapplies it at login.

Create `~/.local/bin/restore-displays.sh`, owned by `flipper`, executable:

```sh
#!/bin/bash
# Reapply the cabinet display layout. Edit the geometry to match your screens.
export DISPLAY="${DISPLAY:-:0}"
for i in $(seq 1 15); do
    xrandr --query 2>/dev/null | grep -q "DP-0 connected" && break
    sleep 1
done
xrandr \
  --output DP-0   --primary --mode 1920x1080 --rate 60 --pos 240x0    --rotate normal \
  --output HDMI-0           --mode 1920x1080 --rate 60 --pos 240x1080 --rotate normal \
  --output HDMI-1           --mode 3840x2160 --rate 60 --pos 0x2160   --rotate left \
  --output DP-1   --off
```

Then add an autostart entry at
`~/.config/autostart/restore-displays.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=Restore display layout
Exec=/home/flipper/.local/bin/restore-displays.sh
X-GNOME-Autostart-enabled=true
NoDisplay=true
```

Adjust the outputs, modes, positions, and rotations to your own cabinet.

## 3. Power button and lid behavior

By default `systemd-logind` and GNOME may suspend the machine or show an
interactive menu when the power button is pressed. For a cabinet you want the
power button to perform a clean shutdown, and you want the lid (if the machine
has one) to do nothing.

Create a drop-in file. As root, write
`/etc/systemd/logind.conf.d/99-no-energy.conf`:

```ini
[Login]
HandlePowerKey=poweroff
HandlePowerKeyLongPress=poweroff
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
HandleLidSwitchExternalPower=ignore
IdleAction=ignore
IdleActionSec=0
```

Reload logind without ending sessions:

```sh
sudo systemctl kill -s HUP systemd-logind.service
```

Verify the effective values:

```sh
busctl get-property org.freedesktop.login1 /org/freedesktop/login1 \
    org.freedesktop.login1.Manager HandlePowerKey
```

It should print `s "poweroff"`.

!!! warning "Write config files with care"
    When writing a config file through `sudo tee` while also piping a password
    into `sudo`, the password can end up inside the file. Prime sudo first with
    `sudo -v` or a separate `sudo true`, then write the file with a normal
    redirect. Always read the file back to confirm its contents.

## 4. Disable screen blanking, dimming, and sleep

These are GNOME settings, applied per user with `gsettings`. Run them in the
`flipper` desktop session, or over SSH with the session bus address exported.

```sh
gsettings set org.gnome.desktop.session idle-delay 0
gsettings set org.gnome.desktop.screensaver idle-activation-enabled false
gsettings set org.gnome.desktop.settings-daemon.plugins.power idle-dim false 2>/dev/null || \
gsettings set org.gnome.settings-daemon.plugins.power idle-dim false
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-timeout 0
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-timeout 0
gsettings set org.gnome.settings-daemon.plugins.power lid-close-ac-action 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power lid-close-battery-action 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power power-button-action 'nothing'
```

Setting `power-button-action` to `nothing` stops GNOME from intercepting the
power key, so the `logind` rule from step 3 takes effect.

## 5. Disable the lock screen entirely

This is the most important step for a cabinet. If a screen ever locks, an
operator has to find a keyboard and type the `flipper` password to recover the
cabinet. Disable locking completely.

```sh
gsettings set org.gnome.desktop.screensaver lock-enabled false
gsettings set org.gnome.desktop.lockdown disable-lock-screen true
gsettings set org.gnome.desktop.lockdown disable-log-out true
```

* `lock-enabled false` stops the screensaver from locking.
* `disable-lock-screen true` makes the lock screen impossible to invoke at all,
  including the `Super+L` shortcut and any application that asks to lock.
* `disable-log-out true` removes the log out item so an operator cannot land on
  the GDM picker by accident.

The `flipper` account keeps its password, so `sudo` still requires it. Only the
on screen lock is removed.

## Result

After these steps the cabinet:

* boots straight to the desktop with the correct three screen layout,
* never blanks, dims, sleeps, or locks,
* shuts down cleanly on a power button press.

## Next

Continue with [Boot branding](cab-branding.md), or skip to
[Install Fliphetic](install.md) if you do not need custom boot graphics.

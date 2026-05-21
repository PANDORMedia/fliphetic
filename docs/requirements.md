# Requirements

This page lists everything you need before provisioning a Fliphetic cabinet.

## Hardware

| Item | Notes |
|------|-------|
| Cabinet PC | A 64 bit x86 machine. Reference cabinet runs Ubuntu 24.04 on a multi-core CPU with 8 GB RAM or more. A discrete GPU helps drive the 4K portrait screen. |
| Three displays | The reference layout is one 4K portrait playfield, one 1080p backglass, and one 1080p DMD. Any three outputs work; you configure their geometry later. |
| ESP32 board(s) | Optional. One or more ESP32 boards wired to the cabinet buttons. Each connects over USB. Apps that do not use buttons can run without any board. |
| USB cabling | A reliable data USB cable per ESP32. Charge-only cables will not work for flashing. |
| Network | Wired or wireless connectivity for the cabinet, plus internet access for Tailscale, Docker image pulls, and git. |

### Reference display layout

The reference cabinet uses these three outputs. Your geometry will differ, but
this shows the shape of a typical setup.

| Role | Output | Resolution | Position | Orientation |
|------|--------|------------|----------|-------------|
| playfield | HDMI-1 | 3840x2160 | 0,2160 | rotated left (portrait) |
| backglass | DP-0 | 1920x1080 | 240,0 | normal (primary) |
| dmd | HDMI-0 | 1920x1080 | 240,1080 | normal |

You read these values with `xrandr` during setup and register them in the
dashboard.

## Software

| Software | Version | Notes |
|----------|---------|-------|
| Ubuntu | 24.04 LTS | The reference platform. Other modern Linux distributions can work but are not documented here. |
| GNOME on Xorg | shipped with 24.04 | The kiosk window placement uses X11 tooling (`xrandr`, `wmctrl`). Use the Xorg session, not Wayland. |
| Docker Engine | 24 or newer | With the Compose v2 plugin. |
| Chromium | current | Installed as the Ubuntu snap. Google Chrome also works. |
| Python | 3.12 | Ships with Ubuntu 24.04. |
| Tailscale | current | For private remote access to the dashboard. |

Fliphetic itself is a Python application installed with `uv`. The full package
list is installed during the [bootstrap step](install.md).

## Accounts and access

* **A `flipper` user on the cabinet** with sudo rights. All Fliphetic services
  run as this user. The name `flipper` is used throughout this documentation.
* **A Tailscale account** and the cabinet joined to your tailnet. Students and
  operators who manage the cabinet also need to be on the tailnet, or you can
  expose the dashboard another way.
* **A GitHub account or organization** to host student app repositories. Public
  repositories are simplest because the cabinet clones them over HTTPS without
  credentials.
* **A workstation** (macOS or Linux) that can reach the cabinet over SSH. You
  run the deployment script from this workstation.

## Network model

The dashboard binds to the cabinet's Tailscale IP address only. This has two
consequences:

* Anyone on your tailnet can reach the dashboard. Anyone not on the tailnet
  cannot, even on the same local network.
* The cabinet must finish bringing up its Tailscale interface before the
  dashboard can start. The Fliphetic service handles this with a startup wait,
  described in [Install Fliphetic](install.md).

Student app repositories are cloned over the public internet, so the cabinet
needs a normal internet connection in addition to Tailscale.

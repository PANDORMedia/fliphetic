# Fliphetic

Fliphetic is a deployment and management system for a virtual pinball cabinet.
It is built for an educational context: students develop pinball "apps" in their
own git repositories, and Fliphetic lets you load any of them onto the physical
cabinet from a single web dashboard.

## What it does

A Fliphetic cabinet has three Chromium kiosk screens and one or more ESP32
boards wired to the cabinet buttons. Fliphetic runs a small dashboard on the
cabinet that lets you:

* **Register** a student app by its git URL.
* **Watch** registered repositories and surface new releases automatically.
* **Load** any registered app, which stops the previous app, flashes the ESP32
  firmware, starts the app's Docker services, and points each Chromium kiosk at
  the right URL.
* **Manage** the cabinet: screens, ESP32 devices, users, and the app that loads
  automatically on power up.

## Architecture at a glance

```
+--------------------- cabinet (flipper@cab) -----------------------+
|                                                                   |
|  +--------------+    +----------------+   +-------------------+    |
|  |  dashboard   |<-->|  orchestrator  |-->|  3x chromium      |    |
|  |  (FastAPI +  |    |  - load/stop   |   |  --kiosk windows  |    |
|  |   HTMX)      |    |  - app state   |   |  (systemd user    |    |
|  |              |    |  - run logs    |   |   services)       |    |
|  +--------------+    +-------+--------+   +-------------------+    |
|         ^                    |                                   |
|         |                    +--> docker compose -p <app>        |
|  +------+-------+             |                                   |
|  |  watchdog    |             +--> esptool flash -> ESP32(s)      |
|  |  (git poll)  |                                                 |
|  +--------------+                                                 |
|         |                                                         |
|         v                                                         |
|   /var/lib/fliphetic/apps/<app-id>/   (cloned repositories)        |
|                                                                   |
+-------------------------------------------------------------------+
        ^
        |  Tailscale (you and students reach the dashboard)
        |
   browser -> dashboard
```

The dashboard binds to the cabinet's Tailscale address only, so it is reachable
from your tailnet without exposing anything to the public internet.

## The moving parts

| Component | Role |
|-----------|------|
| Dashboard | Web UI and HTTP API, runs as a systemd service on the cabinet. |
| Orchestrator | Performs load, stop, and status operations. Serialized so two loads cannot interleave. |
| Watchdog | Polls each registered git repository for new releases. |
| Kiosk services | One systemd user service per screen, each running a Chromium window. |
| Storage | A SQLite database for the registry, run history, users, screens, and ESP32 devices. |

## How to read this documentation

If you are setting up a cabinet from scratch, read the pages in order:

1. [Requirements](requirements.md) covers the hardware, software, and accounts.
2. The **Cabinet setup** section provisions the operating system, network,
   displays, power behavior, and boot branding.
3. [Install Fliphetic](install.md) installs the dashboard itself.
4. [Configuration](configuration.md) registers screens, ESP32 devices, and users.
5. [Operations](operations.md) is the day to day guide.

If you are a student building an app, go straight to
[Building apps](apps.md) and [ESP32 firmware](esp32-firmware.md).

The documentation is available in English and French. Use the language selector
in the top bar to switch.

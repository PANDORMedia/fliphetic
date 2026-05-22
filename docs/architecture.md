# Architecture

This page describes how Fliphetic works internally. It is useful when extending
the system or diagnosing unusual behavior.

## Components

Fliphetic is a single Python application. Inside it:

| Module | Responsibility |
|--------|----------------|
| Dashboard | A FastAPI application: the web UI and HTTP API. |
| Orchestrator | Performs load, stop, and status. The only place that drives Docker, esptool, and the kiosks. |
| Watchdog | An async task that polls registered repositories for new references. |
| Kiosk | Writes per screen URL files and controls the Chromium kiosk services. |
| Storage | A thin SQLite layer for the registry, run history, users, screens, and ESP32 devices. |
| Manifest | Parses and validates `fliphetic.toml`. |

The dashboard runs as the system service `fliphetic.service`. Each screen runs
a Chromium window as the user service `fliphetic-kiosk@<role>.service`.

## The load lifecycle

Loading an app is the central operation. In order:

1. **Splash.** Every screen is pointed at an internal splash page so the user
   sees immediate feedback.
2. **Fetch.** The app's repository is fetched and checked out to the target
   reference (the branch head or newest matching tag).
3. **Manifest.** `fliphetic.toml` is read. The stored display name is synced
   from `app.name`.
4. **Stop previous.** The previously loaded app's Docker services are brought
   down.
5. **Flash.** Each `[esp32.<name>]` block is flashed with esptool. A missing
   required device aborts the load.
6. **Compose up.** Any dashboard-set environment variables are written to a
   generated Compose override file, then the app's Docker services are started
   and given time to pass their healthchecks.
7. **Resolve screens.** Each screen target is turned into a host reachable URL.
   `service` targets are resolved through `docker compose port`; each distinct
   service and port is queried once.
8. **Switch kiosks.** The resolved URLs are written to the per screen files and
   the kiosk services are restarted.
9. **Record.** The run is saved with its full log, and the app becomes the
   current app.

If any step fails, the run is marked failed, the half started app is brought
down, and the previously selected app remains.

## Serialization

`load()` and `stop()` are serialized with a lock. Concurrent callers (the boot
auto load racing a dashboard click, or two operators clicking Load at once)
would otherwise interleave Docker and kiosk operations and leave the cabinet in
an inconsistent state. With the lock, a second request waits for the first to
finish, then runs.

## Storage

State lives in one SQLite database, by default
`/var/lib/fliphetic/state.sqlite`. The main tables are:

| Table | Contents |
|-------|----------|
| `apps` | Registered apps: id, name, git URL, deploy strategy, current and latest SHA, owner. |
| `runs` | Load, stop, and flash history with full logs. |
| `users` | Accounts: username, scrypt password hash, role. |
| `screens` | Screen role, output, and geometry. |
| `esp32_devices` | Device name, port, chip, baud. |
| `app_env` | Per-app environment variables: app, container, name, value. |
| `cab_state` | Key and value pairs: the current app, the boot default, the signup setting, the session secret. |

Cloned repositories live under `/var/lib/fliphetic/apps/<app-id>/`. A `current`
symlink points at the loaded app. Generated Compose override files, one per app
that has environment variables, live under `apps/.env-overrides/`.

## The kiosk approach

Each screen runs an independent Chromium window. The kiosk service for a role
reads that role's URL file and launches Chromium with `--app=<url>`, which
removes all browser chrome.

The window is then placed and made full screen by the window manager through
`wmctrl`, rather than by Chromium's own kiosk mode. This is deliberate:

* Chromium's `--kiosk` does not reliably honor a target monitor on a multi
  monitor GNOME desktop, so windows can collapse onto the primary screen.
* Letting the window manager full screen the window avoids Chromium's "press a
  key to exit full screen" notification, because Chromium itself never enters
  its full screen state.

A short background routine waits for the window to appear, then positions and
full screens it on the correct output.

## The watchdog

The watchdog is an async task in the dashboard process. About once a minute, for
each registered app, it fetches the repository and resolves the target
reference. If that reference differs from what was last seen, the app is flagged
with an update badge. The watchdog never loads anything; an operator decides
when to apply an update.

## Network and boot

The dashboard binds only to the cabinet's Tailscale address. The service waits
for that address to exist before starting. On boot, after the dashboard comes
up, it auto loads an app: the boot default if one is set, otherwise the app that
was loaded before shutdown.

## Authentication

The dashboard requires a login. Sessions are signed cookies; the signing secret
is generated once per cabinet and stored in `cab_state`. Passwords are hashed
with scrypt. There are two roles, `admin` and `user`, plus a per app owner
relationship. See [Configuration](configuration.md) for what each can do.

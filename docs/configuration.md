# Configuration

After installation, the cabinet needs to know its screens, its ESP32 devices,
and its users. Most of this is done from the dashboard **System** page by an
administrator. This page explains each setting.

## Cabinet configuration file

`/etc/fliphetic/config.toml` holds the small set of settings that rarely change:

```toml
cab_id    = "cab-0"
bind_host = "100.x.y.z"
bind_port = 8080

apps_dir     = "/var/lib/fliphetic/apps"
state_db     = "/var/lib/fliphetic/state.sqlite"
screens_dir  = "/var/lib/fliphetic/screens"
current_link = "/var/lib/fliphetic/current"
```

Everything else (screens, ESP32 devices, users, the boot default app, the
signup setting) lives in the SQLite state database and is managed at runtime.

## Screens

A screen has a **role** (the name students target in their app manifests, for
example `playfield`), an xrandr **output**, and a geometry (position and size).

### Adding screens

Open the dashboard, go to **System**, and find the **Screens** section. There
are two ways to add a screen:

* **Detect from xrandr.** The "Detected" table lists every connected output
  that is not yet registered, with its current geometry already filled in. Type
  a role name and click Add.
* **Add manually.** Use the "Add screen" form to enter the role, output,
  position, and size by hand.

The first time the dashboard starts, if the configuration file contained a
legacy `[[screens]]` block, those screens are copied into the database
automatically. After that, the database is the single source of truth.

### Screen roles

Role names are lowercase with hyphens. The reference cabinet uses `playfield`,
`backglass`, and `dmd`. Whatever roles you choose, they must match:

* the kiosk service instances you enabled (`fliphetic-kiosk@<role>.service`),
* the `[screens]` keys students use in their app manifests.

## ESP32 devices

Each ESP32 wired to the cabinet is registered with a **name**. Students target
that name in their app manifest. The administrator decides which physical USB
port each name maps to.

### Adding devices

In **System**, find the **ESP32 devices** section.

* **Detected.** The "Detected" table lists every USB serial device at
  `/dev/serial/by-id/` that is not yet registered. Each row has a suggested
  name derived from the USB descriptor. Edit it if you like and click Add, or
  click **Auto-register all** to add every detected device at once.
* **Add manually.** Use the "Add device" form for full control over name,
  port, chip type, and baud rate.

Always prefer a `/dev/serial/by-id/...` path over `/dev/ttyUSB0`. The by-id path
is stable across reboots and across plugging order.

### Device fields

| Field | Meaning |
|-------|---------|
| Name | Lowercase hyphenated identifier. Students reference it as `[esp32.<name>]`. |
| Port | Device path, ideally under `/dev/serial/by-id/`. |
| Chip | `esp32`, `esp32s2`, `esp32s3`, `esp32c3`, or `esp32c6`. |
| Baud | Flashing baud rate. `921600` is a good default. |

### Command line

The same operations are available from the cabinet command line:

```sh
fliphetic esp32 list
fliphetic esp32 detect
fliphetic esp32 add buttons /dev/serial/by-id/usb-...  --chip esp32
fliphetic esp32 remove buttons
```

## Users and roles

Fliphetic has two roles.

| Role | Can do |
|------|--------|
| `admin` | Everything: register and unregister apps, pull updates, manage screens, ESP32 devices, users, the signup setting, and the boot default. |
| `user` | Sign in, view apps and system info, load and stop any app. |

In addition, the **owner** of an app (the user who registered it) can pull
updates for and unregister that specific app, even without the admin role.

### Managing users from the command line

```sh
fliphetic users list
fliphetic users add alice --admin       # create an administrator
fliphetic users add bob                 # create a normal user
fliphetic users passwd bob              # change a password
fliphetic users role bob admin          # change a role
fliphetic users delete bob
```

Each `add` and `passwd` prompts for the password unless you pass `--password`.

### Self-service signup

Administrators can open or close self-service registration from the **System**
page.

* **Open.** A `/signup` page accepts new accounts. Every account created this
  way is a normal `user`, never an admin. The login page shows a link to it.
* **Closed.** The `/signup` page reports that registration is closed. This is
  the default.

Open signup when a class starts so students can create their own accounts, then
close it again.

## Boot default app

The cabinet can auto load a chosen app on power up.

* If a **boot default** is set, the cabinet loads it on every boot, regardless
  of what was loaded before shutdown. This is the right choice for a public
  cabinet: a power cycle always returns to a known idle or attract app.
* If no boot default is set, the cabinet resumes whatever was loaded before it
  shut down.

Set it from an app's page in the dashboard ("Set as boot default"), or from the
command line:

```sh
fliphetic default show
fliphetic default set <app-id>
fliphetic default clear
```

The chosen app shows a "boot default" badge in the dashboard.

## Next

Continue with [Operations](operations.md) for the day to day guide.

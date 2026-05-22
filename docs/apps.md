# Building apps

A Fliphetic app is a git repository the cabinet can clone, validate, and run on
its three screens, with optional ESP32 firmware for the buttons. This page is
the contract: follow it and the cabinet will load your app.

## In short

1. Put a `fliphetic.toml` file at the root of your repository.
2. Provide a Docker Compose file that serves your screen content.
3. Optionally commit a prebuilt ESP32 firmware binary and point the manifest at
   it.
4. Push to a public git repository. An administrator registers its URL on the
   cabinet.

## The three screens

The cabinet has three screens. Your app refers to them by **role**, not by
physical port. The reference cabinet uses these roles:

| Role | Typical screen |
|------|----------------|
| `playfield` | The large portrait screen |
| `backglass` | The upper landscape screen |
| `dmd` | The lower landscape screen |

You do not have to use all three. Declare only the roles your app needs. Roles
you omit are blanked. The exact roles available on a given cabinet are visible
on its dashboard System page.

## `fliphetic.toml`

```toml
[app]
id          = "moby-pinball"        # lowercase, hyphens, 3 to 64 characters, unique
name        = "Moby Pinball"        # display name shown in the dashboard
authors     = ["Student Name"]
description = "A whale of a pinball game"

[deploy]
strategy = "branch"                 # "branch" or "tag"
ref      = "main"                   # branch name, or a tag glob like "v*"

[screens]
playfield = { service = "game",  port = 8080 }
backglass = { service = "art",   port = 80 }
dmd       = { service = "score", port = 80 }

[compose]
file          = "deploy/docker-compose.yml"
ready_timeout = 30

[esp32.buttons]
firmware = "firmware/build/buttons.bin"
chip     = "esp32"
```

### `[app]`

`id` is the canonical key for your app on the cabinet. Choose it once and do not
change it: the cabinet uses it for directory names and Docker project names.
`name` is the human readable label, and it can change freely (the dashboard
re-reads it when the app is pulled or loaded).

### `[deploy]`

Controls which reference the watchdog tracks and the cabinet loads.

* `strategy = "branch"` follows the head of `ref` (a branch name).
* `strategy = "tag"` follows the newest tag matching `ref` used as a glob, for
  example `v*`.

### `[screens]`

A map of screen role to a target. Each target sets exactly one of:

* `url` for an absolute URL, or
* `service` for a Docker Compose service name, with an optional `port` (the
  container port, default 80) and an optional `path` appended to the resolved
  URL.

### `[compose]`

`file` is the path to your Compose file, relative to the repository root.
`ready_timeout` is how many seconds the cabinet waits for the services to be
healthy before declaring the load complete.

### `[esp32.<name>]`

Optional, one block per cabinet ESP32 device you target. See
[ESP32 firmware](esp32-firmware.md) for the full details.

## The Docker Compose file

Each service that backs a screen publishes its container port to the host so the
cabinet's Chromium can reach it. Use `"0:80"` so Docker assigns an ephemeral host
port; the cabinet discovers it after the services start.

```yaml
services:
  game:
    image: nginx:alpine
    volumes: ["../site:/usr/share/nginx/html:ro"]
    ports: ["0:80"]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost/"]
      interval: 5s
      timeout: 2s
      retries: 5
```

Conventions:

* Healthchecks are strongly recommended. The cabinet waits for services to be
  healthy before switching the screens to your app.
* Do not bind to fixed host ports. Use `0:<container_port>`.
* Relative volume paths are resolved against the Compose file's directory.

## Environment variables

Your app's containers can receive environment variables that are **not** part
of your repository. They are set per app on the cabinet dashboard by whoever
deploys the app — see [Operations](operations.md).

This is where anything you must not commit belongs — API keys and other
secrets — along with values that differ from one cabinet to another. Do not put
those in `fliphetic.toml` or in your Compose file: your repository is public.

Inside a container, read the variables the usual way: `process.env` in Node,
`os.environ` in Python, and so on. Document which variables your app expects so
whoever deploys it knows what to set, and fall back to a sensible default when
one is absent.

## Three ways to wire Docker to screens

The cabinet supports all three topologies.

### One service per screen

Three services, three screens:

```toml
[screens]
playfield = { service = "playfield", port = 80 }
backglass = { service = "backglass", port = 80 }
dmd       = { service = "dmd",       port = 80 }
```

### One service for all screens

A single service serves different content per screen, distinguished by `path`:

```toml
[screens]
playfield = { service = "web", port = 80, path = "/playfield" }
backglass = { service = "web", port = 80, path = "/backglass" }
dmd       = { service = "web", port = 80, path = "/dmd" }
```

`path` may also be a query string, for example `path = "/?screen=playfield"`.

### Mixed

Several services, with screens mapped freely. Two screens can share one service
through different paths or ports while a third uses its own service. The cabinet
queries each distinct service and port once.

## What "Load" does

1. Fetch the latest code for your `[deploy]` reference.
2. Stop the previous app's Docker services.
3. Flash any ESP32 firmware your manifest declares.
4. Start your Docker services and wait for healthchecks.
5. Resolve each screen target to a URL and point the kiosks at it.

If any step fails, the load is marked failed and the previous app stays
selected. Open the app page on the dashboard and read the run log.

## Validating locally

Check your manifest before you push:

```sh
fliphetic validate .
```

This reports schema errors and missing files.

## Common errors

| Error | Cause |
|-------|-------|
| `app.id must be lowercase` | The `id` has uppercase letters, underscores, or a bad length. |
| `manifest references missing files` | A path in `[compose]` or `[esp32]` does not exist in the repository. |
| `docker compose up failed` | An image will not pull, a healthcheck never passes, or a port collides. |
| `cab has no esp32 device named X` | Your `[esp32.X]` targets a device the cabinet has not registered. |
| `screen role X must be lowercase` | A screen role name is not lowercase with hyphens. |

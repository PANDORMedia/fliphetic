# Install Fliphetic

This page installs the Fliphetic application on a cabinet that has already been
provisioned (see the Cabinet setup section). At the end the dashboard is running
and reachable over Tailscale.

## 1. Bootstrap the cabinet packages

Fliphetic needs Chromium, Docker, git, and a few Python tools. Install them and
adjust group membership.

```sh
sudo apt-get update
sudo apt-get install -y --no-install-recommends \
    chromium-browser \
    docker.io docker-compose-v2 docker-buildx \
    git pipx rsync python3-venv wmctrl xdotool

# Python tools for the flipper user, isolated with pipx
pipx ensurepath
pipx install esptool
pipx install uv

# flipper must run Docker and open serial devices without sudo
sudo usermod -aG docker flipper
sudo usermod -aG dialout flipper

# user services must keep running without an active graphical login
sudo loginctl enable-linger flipper

# make sure the Docker daemon is enabled
sudo systemctl enable --now docker
```

Group membership changes take effect on the next login. Log out and back in, or
reboot, before continuing.

!!! warning "The docker group is root equivalent"
    Adding `flipper` to the `docker` group lets that user gain root through the
    Docker socket. For a single purpose cabinet on a private tailnet this is
    accepted practice, but it is worth knowing.

## 2. Directory layout

Fliphetic uses three locations on the cabinet.

| Path | Owner | Contents |
|------|-------|----------|
| `/opt/fliphetic` | `flipper` | The application source and its `uv` virtual environment. |
| `/etc/fliphetic/config.toml` | `root` | Cabinet configuration (identity, bind address, paths). |
| `/var/lib/fliphetic` | `flipper` | SQLite state, cloned app repositories, per screen URL files. |

Create them:

```sh
sudo install -d -o flipper -g flipper /opt/fliphetic
sudo install -d -o root    -g root    /etc/fliphetic
sudo install -d -o flipper -g flipper \
    /var/lib/fliphetic /var/lib/fliphetic/apps /var/lib/fliphetic/screens
```

## 3. Install the application

Copy the Fliphetic source to `/opt/fliphetic` and build its virtual environment
with `uv`:

```sh
cd /opt/fliphetic
uv sync
```

The reference setup uses a deployment script, `bin/deploy`, run from your
workstation. It does the following in one step:

1. `rsync` the source to the cabinet under `/opt/fliphetic`.
2. Run `uv sync` to build the virtual environment.
3. Install the systemd units.
4. Restart the dashboard and the kiosk services.

Using a script keeps redeployments fast and repeatable. The manual steps on this
page are the equivalent if you prefer not to use it.

## 4. Cabinet configuration file

Create `/etc/fliphetic/config.toml`. The minimum is the cabinet identity and
the bind address:

```toml
cab_id    = "cab-0"
bind_host = "100.x.y.z"   # the cabinet Tailscale IPv4 address
bind_port = 8080

apps_dir     = "/var/lib/fliphetic/apps"
state_db     = "/var/lib/fliphetic/state.sqlite"
screens_dir  = "/var/lib/fliphetic/screens"
current_link = "/var/lib/fliphetic/current"
```

Screens and ESP32 devices are not configured here. They live in the SQLite
state database and are managed from the dashboard. See
[Configuration](configuration.md).

## 5. systemd units

### Dashboard service

Install `/etc/systemd/system/fliphetic.service`:

```ini
[Unit]
Description=Fliphetic dashboard and orchestrator
After=network-online.target docker.service tailscaled.service
Wants=network-online.target

[Service]
Type=simple
User=flipper
Group=flipper
Environment=FLIPHETIC_CONFIG=/etc/fliphetic/config.toml
Environment=XDG_RUNTIME_DIR=/run/user/1000
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus
Environment=PATH=/home/flipper/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStartPre=/usr/bin/bash -c 'for i in $(seq 1 30); do ip -4 addr show tailscale0 2>/dev/null | grep -q "inet " && exit 0; sleep 1; done; echo "tailscale0 has no IPv4" >&2; exit 1'
ExecStart=/opt/fliphetic/.venv/bin/fliphetic serve
WorkingDirectory=/opt/fliphetic
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

The `ExecStartPre` line waits up to 30 seconds for the Tailscale interface to
receive its address. Without it, the dashboard can start before the bind
address exists and fail.

Enable and start it:

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now fliphetic.service
```

### Kiosk services

Each screen runs its own Chromium window as a systemd **user** service. Install
the template unit at
`~flipper/.config/systemd/user/fliphetic-kiosk@.service`:

```ini
[Unit]
Description=Fliphetic kiosk chromium for screen role "%i"
After=graphical-session.target
PartOf=graphical-session.target

[Service]
Type=simple
Environment=DISPLAY=:0
Environment=XAUTHORITY=%h/.Xauthority
ExecStart=/opt/fliphetic/.venv/bin/fliphetic kiosk launch %i
Restart=on-failure
RestartSec=2

[Install]
WantedBy=graphical-session.target
```

Enable one instance per screen role. The role names must match the screens you
register later:

```sh
systemctl --user daemon-reload
systemctl --user enable --now fliphetic-kiosk@playfield.service
systemctl --user enable --now fliphetic-kiosk@backglass.service
systemctl --user enable --now fliphetic-kiosk@dmd.service
```

## 6. Create the first administrator

The dashboard requires a login. Create the first administrator account from the
command line on the cabinet:

```sh
/opt/fliphetic/.venv/bin/fliphetic users add admin --admin
```

You are prompted for a password. This account can register apps, manage screens
and ESP32 devices, manage other users, and set the boot default app.

## 7. Verify

Check the dashboard is up:

```sh
curl -s http://100.x.y.z:8080/healthz
```

It should return `{"ok":true}`. Then open `http://100.x.y.z:8080` in a browser
on your tailnet and sign in as `admin`.

## Next

Continue with [Configuration](configuration.md) to register screens, ESP32
devices, and users.

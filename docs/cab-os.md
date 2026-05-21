# Cabinet setup: OS and network

This page provisions the operating system and remote access. At the end you
have an Ubuntu cabinet that boots straight to a desktop and is reachable over
Tailscale by SSH.

## 1. Install Ubuntu 24.04

Install Ubuntu 24.04 LTS on the cabinet PC. During or after installation:

* Create a user named `flipper`.
* Give `flipper` administrator (sudo) rights.
* Choose the normal desktop install so GNOME and GDM are present.

After first boot, make sure the graphical session is **Xorg**, not Wayland.
At the GDM login screen, click the gear icon and pick "Ubuntu on Xorg".
Fliphetic places kiosk windows with X11 tools and needs the Xorg session.

## 2. Enable automatic login

A cabinet should boot to its desktop with no keyboard interaction. Enable GDM
automatic login for `flipper`.

Edit `/etc/gdm3/custom.conf` and set, under `[daemon]`:

```ini
AutomaticLoginEnable=true
AutomaticLogin=flipper
```

Reboot and confirm the cabinet lands on the desktop without a password prompt.

## 3. Install Tailscale

Tailscale gives the cabinet a stable private address that you and your students
can reach from anywhere, without opening ports.

```sh
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the printed URL to authenticate the cabinet into your tailnet. Then read
the cabinet's Tailscale IPv4 address:

```sh
tailscale ip -4
```

Write this address down. It is used everywhere: the dashboard binds to it, and
the deployment script targets it. In this documentation the placeholder
`100.x.y.z` stands for your cabinet's Tailscale address.

!!! note "Tailscale addresses are stable"
    A Tailscale node keeps its IP for the life of the node, so you rarely need
    to change it. If it ever does change, update `bind_host` in
    `/etc/fliphetic/config.toml` and the target in the deployment script.

## 4. Set up SSH access from your workstation

You manage and deploy the cabinet from a separate workstation over SSH. Install
a key so you are not typing passwords.

From your workstation, with an existing SSH key (or after `ssh-keygen`):

```sh
ssh-copy-id flipper@100.x.y.z
```

Confirm key based login works:

```sh
ssh flipper@100.x.y.z 'echo connected as $(whoami)'
```

Optionally add a host alias to `~/.ssh/config` on your workstation:

```
Host flipper
    HostName 100.x.y.z
    User flipper
```

After that, `ssh flipper` is enough.

## 5. Keep the cabinet on the network

The cabinet needs:

* Internet access, for Docker image pulls and cloning student repositories.
* Tailscale connectivity, for the dashboard.

If the cabinet uses Wi-Fi, make sure the connection is set to connect
automatically and is available to the `flipper` session at boot.

## Next

Continue with [Display and power](cab-display-power.md) to configure the three
screens and the power behavior.

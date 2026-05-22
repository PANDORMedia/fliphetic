# Operations

This page is the day to day guide to running a Fliphetic cabinet once it is set
up.

## The dashboard

Open `http://100.x.y.z:8080` on a device on your tailnet and sign in. The top
bar has three sections:

* **Apps** lists every registered app.
* **System** shows cabinet information, screens, and ESP32 devices.
* **+ Register** (administrators only) registers a new app.

The current user and role are shown on the right, with a log out link.

## Registering an app

An app is a git repository that contains a valid `fliphetic.toml` manifest at
its root. To register one, an administrator opens **+ Register**, pastes the
git URL, and submits.

Fliphetic clones the repository, validates the manifest, and adds the app to the
list. If the manifest is missing or invalid, registration fails with the reason
shown, and nothing is added.

Public repositories are simplest because the cabinet clones over HTTPS with no
credentials.

## Environment variables

An app's Docker containers can be given environment variables — API URLs,
feature flags, credentials — without putting them in the app's repository. They
are managed here on the cabinet and stored in its database, never in
`fliphetic.toml`, which keeps secrets out of the (usually public) app
repository.

An administrator or the app's owner sets them in the **Environment variables**
section of the app's page. Each variable has:

* a **container** — the Compose service it is injected into, or *all
  containers*;
* a **name** — letters, digits, and underscores;
* a **value**.

The dashboard reads the app's container list from its Compose file, so the
container selector lists the real services. Submitting a variable with the same
container and name again updates its value; the **Remove** button deletes one.

Variables can also be set when the app is first registered: the **+ Register**
form has an environment-variables box where you enter one `KEY=VALUE` per line.
Those apply to all containers and can be re-scoped per container afterwards.

Environment variables take effect on the app's next **Load**. Changing them
does not affect a currently running app until it is loaded again.

## Loading and stopping an app

From the Apps list or an app's own page, click **Load**. Loading performs these
steps in order, and is serialized so two loads can never overlap:

1. Fetch the latest code for the app's configured branch or tag.
2. Stop the Docker services of the previously loaded app.
3. Flash any ESP32 firmware the app declares.
4. Start the app's Docker services with Compose.
5. Resolve each screen to a URL and point the kiosks at it.

While a load runs, the screens show a splash that reflects the current step.
When it finishes, the screens switch to the app.

Click **Stop** to bring the current app down and blank the screens.

## Following a load

Each load is recorded as a run on the app's page, with a status and a full log.
If a load fails, open the app page and read the run log. Every step writes to
it, so the failing step and its error are visible there.

## The watchdog

Fliphetic polls every registered repository about once a minute. For each app it
fetches the repository and resolves the target reference (the branch head, or
the newest tag matching a glob, depending on the app's deploy strategy).

When a newer reference is found, the app shows an **update** badge in the Apps
list. The watchdog never loads anything by itself. An operator decides when to
apply an update by clicking **Load**, which always fetches and checks out the
latest target before running.

## Pulling without loading

The **Pull latest** button on an app page fetches and checks out the latest
target reference without starting the app. This is useful to inspect the new
manifest, or to refresh the app's stored display name after its `app.name`
changed. Pulling is available to administrators and to the app's owner.

## Switching apps

Loading a different app automatically stops the current one first. You do not
need to stop before you load. Because loads are serialized, clicking Load on two
apps in quick succession runs them one after the other, and the cabinet ends on
the second.

## Reboot behavior

On power up the cabinet:

1. Brings up the network and Tailscale.
2. Starts the Fliphetic dashboard service once the Tailscale address exists.
3. Auto loads an app: the boot default if one is set, otherwise the app that
   was loaded before shutdown.
4. Brings up the three kiosk windows and points them at the loaded app.

See [Configuration](configuration.md) for the boot default setting.

## Unregistering an app

**Unregister** on an app page stops the app if it is running, removes it from
the registry, and deletes its cloned repository from the cabinet. If the app was
the boot default, that setting is cleared too. Unregistering is available to
administrators and to the app's owner.

## Useful command line operations

Run these on the cabinet, as `flipper`:

```sh
fliphetic list                  # registered apps
fliphetic status                # what is running, per screen state
fliphetic load <app-id>         # load an app
fliphetic stop                  # stop the current app
fliphetic validate <path>       # check a manifest without registering
```

# Installer Fliphetic

Cette page installe l'application Fliphetic sur une borne de flipper déjà provisionnée (voir la section Configuration de la borne). À la fin, le tableau de bord fonctionne et est accessible via Tailscale.

## 1. Amorcer les paquets de la borne

Fliphetic nécessite Chromium, Docker, git, et quelques outils Python. Installez-les et ajustez l'appartenance aux groupes.

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

Les changements d'appartenance aux groupes prennent effet à la connexion suivante. Déconnectez-vous puis reconnectez-vous, ou redémarrez, avant de continuer.

!!! warning "Le groupe docker équivaut à root"
    Ajouter `flipper` au groupe `docker` permet à cet utilisateur d'obtenir les droits root via le socket Docker. Pour une borne à usage unique sur un tailnet privé, c'est une pratique admise, mais il est utile de le savoir.

## 2. Organisation des répertoires

Fliphetic utilise trois emplacements sur la borne.

| Chemin | Propriétaire | Contenu |
|------|-------|----------|
| `/opt/fliphetic` | `flipper` | Le code source de l'application et son environnement virtuel `uv`. |
| `/etc/fliphetic/config.toml` | `root` | Configuration de la borne (identité, adresse d'écoute, chemins). |
| `/var/lib/fliphetic` | `flipper` | État SQLite, dépôts d'applications clonés, fichiers d'URL par écran. |

Créez-les :

```sh
sudo install -d -o flipper -g flipper /opt/fliphetic
sudo install -d -o root    -g root    /etc/fliphetic
sudo install -d -o flipper -g flipper \
    /var/lib/fliphetic /var/lib/fliphetic/apps /var/lib/fliphetic/screens
```

## 3. Installer l'application

Copiez le code source de Fliphetic vers `/opt/fliphetic` et construisez son environnement virtuel avec `uv` :

```sh
cd /opt/fliphetic
uv sync
```

La configuration de référence utilise un script de déploiement, `bin/deploy`, exécuté depuis votre poste de travail. Il effectue les opérations suivantes en une seule étape :

1. `rsync` du code source vers la borne sous `/opt/fliphetic`.
2. Exécution de `uv sync` pour construire l'environnement virtuel.
3. Installation des unités systemd.
4. Redémarrage des services du tableau de bord et du kiosque.

L'utilisation d'un script rend les redéploiements rapides et reproductibles. Les étapes manuelles de cette page sont l'équivalent si vous préférez ne pas l'utiliser.

## 4. Fichier de configuration de la borne

Créez `/etc/fliphetic/config.toml`. Le minimum est l'identité de la borne et l'adresse d'écoute :

```toml
cab_id    = "cab-0"
bind_host = "100.x.y.z"   # the cabinet Tailscale IPv4 address
bind_port = 8080

apps_dir     = "/var/lib/fliphetic/apps"
state_db     = "/var/lib/fliphetic/state.sqlite"
screens_dir  = "/var/lib/fliphetic/screens"
current_link = "/var/lib/fliphetic/current"
```

Les écrans et les périphériques ESP32 ne se configurent pas ici. Ils résident dans la base de données d'état SQLite et sont gérés depuis le tableau de bord. Voir [Configuration](configuration.md).

## 5. Unités systemd

### Service du tableau de bord

Installez `/etc/systemd/system/fliphetic.service` :

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

La ligne `ExecStartPre` attend jusqu'à 30 secondes que l'interface Tailscale reçoive son adresse. Sans elle, le tableau de bord peut démarrer avant que l'adresse d'écoute n'existe et échouer.

Activez-le et démarrez-le :

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now fliphetic.service
```

### Services de kiosque

Chaque écran exécute sa propre fenêtre Chromium en tant que service systemd **utilisateur**. Installez l'unité modèle à l'emplacement `~flipper/.config/systemd/user/fliphetic-kiosk@.service` :

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

Activez une instance par rôle d'écran. Les noms de rôle doivent correspondre aux écrans que vous enregistrerez plus tard :

```sh
systemctl --user daemon-reload
systemctl --user enable --now fliphetic-kiosk@playfield.service
systemctl --user enable --now fliphetic-kiosk@backglass.service
systemctl --user enable --now fliphetic-kiosk@dmd.service
```

## 6. Créer le premier administrateur

Le tableau de bord nécessite une connexion. Créez le premier compte administrateur depuis la ligne de commande sur la borne :

```sh
/opt/fliphetic/.venv/bin/fliphetic users add admin --admin
```

Un mot de passe vous est demandé. Ce compte peut enregistrer des applications, gérer les écrans et les périphériques ESP32, gérer les autres utilisateurs, et définir l'application par défaut au démarrage.

## 7. Vérifier

Vérifiez que le tableau de bord est opérationnel :

```sh
curl -s http://100.x.y.z:8080/healthz
```

Il devrait renvoyer `{"ok":true}`. Ouvrez ensuite `http://100.x.y.z:8080` dans un navigateur sur votre tailnet et connectez-vous en tant qu'`admin`.

## Suite

Poursuivez avec [Configuration](configuration.md) pour enregistrer les écrans, les périphériques ESP32 et les utilisateurs.

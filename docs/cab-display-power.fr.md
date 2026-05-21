# Configuration de la borne : affichage et alimentation

Une borne de flipper est un appareil dédié. Elle ne doit jamais éteindre un
écran, jamais se mettre en veille, jamais afficher un écran de verrouillage, et
elle doit conserver sa disposition multi-écrans d'un redémarrage à l'autre.
Cette page configure tout cela.

Exécutez ces étapes dans la session graphique `flipper` (ou par SSH avec
l'environnement d'affichage défini, comme indiqué).

## 1. Confirmer la disposition d'affichage

Connectez-vous au bureau, organisez les trois écrans comme vous le souhaitez
dans les Paramètres GNOME, puis lisez la géométrie obtenue :

```sh
xrandr --query | grep " connected"
```

Vous verrez des lignes comme :

```
DP-0 connected primary 1920x1080+240+0 ...
HDMI-0 connected 1920x1080+240+1080 ...
HDMI-1 connected 2160x3840+0+2160 left ...
```

Chaque ligne indique le nom de la sortie, la résolution, la position (`+x+y`)
et la rotation. Notez-les. Vous les enregistrerez dans le tableau de bord plus
tard, sur la page [Configuration](configuration.md).

## 2. Faire en sorte que la disposition survive aux redémarrages

GNOME perd parfois une disposition multi-écrans lorsque les sorties sont
détectées dans un ordre différent au démarrage. Pour rendre la disposition
déterministe, ajoutez un petit script qui la réapplique à la connexion.

Créez `~/.local/bin/restore-displays.sh`, appartenant à `flipper`, exécutable :

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

Ajoutez ensuite une entrée de démarrage automatique à
`~/.config/autostart/restore-displays.desktop` :

```ini
[Desktop Entry]
Type=Application
Name=Restore display layout
Exec=/home/flipper/.local/bin/restore-displays.sh
X-GNOME-Autostart-enabled=true
NoDisplay=true
```

Adaptez les sorties, les modes, les positions et les rotations à votre propre
borne.

## 3. Comportement du bouton d'alimentation et du capot

Par défaut, `systemd-logind` et GNOME peuvent mettre la machine en veille ou
afficher un menu interactif lorsque le bouton d'alimentation est pressé. Pour
une borne, vous voulez que le bouton d'alimentation effectue un arrêt propre, et
vous voulez que le capot (si la machine en a un) ne fasse rien.

Créez un fichier complémentaire (drop-in). En tant que root, écrivez
`/etc/systemd/logind.conf.d/99-no-energy.conf` :

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

Rechargez logind sans mettre fin aux sessions :

```sh
sudo systemctl kill -s HUP systemd-logind.service
```

Vérifiez les valeurs effectives :

```sh
busctl get-property org.freedesktop.login1 /org/freedesktop/login1 \
    org.freedesktop.login1.Manager HandlePowerKey
```

Cela devrait afficher `s "poweroff"`.

!!! warning "Écrivez les fichiers de configuration avec soin"
    Lorsque vous écrivez un fichier de configuration via `sudo tee` tout en
    transmettant aussi un mot de passe à `sudo` par un tube, le mot de passe
    peut se retrouver à l'intérieur du fichier. Amorcez d'abord sudo avec
    `sudo -v` ou un `sudo true` séparé, puis écrivez le fichier avec une
    redirection normale. Relisez toujours le fichier pour confirmer son
    contenu.

## 4. Désactiver l'extinction, l'atténuation et la mise en veille de l'écran

Ce sont des réglages GNOME, appliqués par utilisateur avec `gsettings`.
Exécutez-les dans la session de bureau `flipper`, ou par SSH avec l'adresse du
bus de session exportée.

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

Définir `power-button-action` sur `nothing` empêche GNOME d'intercepter la
touche d'alimentation, de sorte que la règle `logind` de l'étape 3 prenne
effet.

## 5. Désactiver entièrement l'écran de verrouillage

C'est l'étape la plus importante pour une borne. Si un écran venait à se
verrouiller, un opérateur devrait trouver un clavier et taper le mot de passe
`flipper` pour récupérer la borne. Désactivez complètement le verrouillage.

```sh
gsettings set org.gnome.desktop.screensaver lock-enabled false
gsettings set org.gnome.desktop.lockdown disable-lock-screen true
gsettings set org.gnome.desktop.lockdown disable-log-out true
```

* `lock-enabled false` empêche l'économiseur d'écran de verrouiller.
* `disable-lock-screen true` rend l'écran de verrouillage impossible à invoquer,
  y compris le raccourci `Super+L` et toute application qui demande un
  verrouillage.
* `disable-log-out true` retire l'élément de déconnexion afin qu'un opérateur ne
  puisse pas se retrouver par accident sur le sélecteur GDM.

Le compte `flipper` conserve son mot de passe, donc `sudo` le requiert toujours.
Seul le verrouillage à l'écran est supprimé.

## Résultat

Après ces étapes, la borne :

* démarre directement sur le bureau avec la disposition correcte des trois
  écrans,
* ne s'éteint jamais, ne s'atténue jamais, ne se met jamais en veille et ne se
  verrouille jamais,
* s'arrête proprement lorsque le bouton d'alimentation est pressé.

## Suite

Continuez avec [Habillage de démarrage](cab-branding.md), ou passez directement
à [Installer Fliphetic](install.md) si vous n'avez pas besoin de graphismes de
démarrage personnalisés.

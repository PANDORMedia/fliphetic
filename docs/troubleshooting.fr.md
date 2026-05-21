# Dépannage

Cette page rassemble les problèmes les plus courants et la façon de les
diagnostiquer.

## Où regarder en premier

| Source | Commande |
|--------|---------|
| Journal du service du tableau de bord | `sudo journalctl -u fliphetic.service -b` |
| Journal d'un service de kiosque | `journalctl --user -u fliphetic-kiosk@playfield.service -b` |
| Un chargement ou un arrêt spécifique | Ouvrez la page de l'application dans le tableau de bord et lisez le journal d'exécution. |
| Statut de la borne | `fliphetic status` |

Chaque chargement et chaque arrêt écrit un journal d'exécution complet, visible
sur la page de l'application. C'est le premier endroit où regarder lorsqu'un
chargement échoue.

## Le tableau de bord est inaccessible

* **Vérifiez le service.** `systemctl status fliphetic.service`. S'il redémarre
  en boucle, lisez le journal.
* **Vérifiez l'adresse Tailscale.** Le tableau de bord se lie à l'adresse
  indiquée dans `bind_host`. Exécutez `tailscale ip -4` sur la borne et
  confirmez qu'elle correspond à `/etc/fliphetic/config.toml`.
* **Délai au démarrage.** Le service attend jusqu'à 30 secondes l'interface
  Tailscale. Si Tailscale prend plus de temps, le service échoue à sa condition
  de démarrage et réessaie. Il devrait se rétablir de lui-même en moins d'une
  minute.
* **Vous n'êtes pas sur le tailnet.** Le tableau de bord n'est accessible que
  depuis le tailnet de la borne. Être sur le même réseau local ne suffit pas.

## Un kiosque affiche le mauvais écran, a des bordures, ou est mal placé

* Confirmez que l'écran est enregistré avec la bonne géométrie sur la page
  System, et que la géométrie correspond à `xrandr --query`.
* Confirmez qu'un service de kiosque est activé pour ce rôle exact
  (`systemctl --user is-enabled fliphetic-kiosk@<role>.service`).
* Redémarrez le kiosque :
  `systemctl --user restart fliphetic-kiosk@<role>.service`.
* Si une fenêtre est sur le mauvais moniteur, la position de l'écran dans la
  base de données est probablement incorrecte. Relisez `xrandr` et corrigez-la.

## Chromium affiche une invite de traduction ou d'autres fenêtres surgissantes

Chromium installé en tant que snap peut ignorer les options de ligne de commande
pour certaines fonctionnalités. Un fichier de stratégie gérée les désactive de
manière fiable. En tant que root, créez
`/var/snap/chromium/current/policies/managed/fliphetic.json` :

```json
{
  "TranslateEnabled": false,
  "DefaultBrowserSettingEnabled": false,
  "PromotionalTabsEnabled": false,
  "BrowserSignin": 0,
  "MetricsReportingEnabled": false
}
```

Redémarrez les services de kiosque. Pour vérifier si une stratégie a pris effet,
dirigez brièvement un kiosque vers `chrome://policy`.

## Un chargement échoue

Ouvrez la page de l'application et lisez le journal d'exécution. Causes
courantes :

* **`docker compose up failed`.** Une image ne se télécharge pas, un test de
  santé ne passe jamais, ou un port publié entre en collision. Essayez le
  fichier Compose de l'application à la main sur la borne.
* **`cab has no esp32 device named X`.** L'application cible un nom de
  périphérique qui n'est pas enregistré. Enregistrez-le sur la page System, ou
  l'application peut définir `required = false`.
* **`ESP32 flash failed`.** Mauvaise puce, mauvais binaire, un câble USB
  défectueux, ou le périphérique est occupé. Voir
  [Firmware ESP32](esp32-firmware.md).
* **`manifest invalid` ou `references missing files`.** Le `fliphetic.toml` est
  mal formé ou pointe vers des fichiers qui ne sont pas dans le dépôt. Exécutez
  `fliphetic validate .` dans le dépôt.

## Un ESP32 n'est pas détecté

* Confirmez que la carte apparaît : `ls -l /dev/serial/by-id/`. Si ce n'est pas
  le cas, le câble est peut-être destiné uniquement à la charge, ou la carte a
  besoin d'un pilote.
* Confirmez que `flipper` fait partie du groupe `dialout` : `groups flipper`. Si
  ce n'est pas le cas, `sudo usermod -aG dialout flipper` et reconnectez-vous.
* Enregistrez le périphérique en utilisant son chemin
  `/dev/serial/by-id/...`, et non `/dev/ttyUSB0`, afin qu'il survive aux
  redémarrages.

## La disposition de l'affichage est perdue après un redémarrage

GNOME peut détecter les sorties dans un ordre différent au démarrage. Utilisez le
script de démarrage automatique `restore-displays` décrit dans
[Affichage et alimentation](cab-display-power.md). Il réapplique la disposition
avec `xrandr` à chaque connexion.

## La borne se met en veille, s'éteint, ou affiche un écran de verrouillage

Réappliquez les étapes de
[Affichage et alimentation](cab-display-power.md). Les paramètres les plus
importants sont `idle-delay 0`, le drop-in `logind`, et
`disable-lock-screen true`. Si la borne se verrouille un jour et qu'il n'y a pas
de clavier, vous pouvez la récupérer via SSH.

## Après un redémarrage, la borne charge la mauvaise application

La borne charge automatiquement la valeur par défaut au démarrage si elle est
définie, sinon la dernière application chargée. Vérifiez et définissez la valeur
par défaut au démarrage :

```sh
fliphetic default show
fliphetic default set <app-id>
```

## Une application renommée affiche toujours son ancien nom

Le nom d'affichage stocké est rafraîchi lorsqu'une application est mise à jour
ou chargée. Cliquez sur **Pull latest** sur la page de l'application, ou chargez
l'application, pour prendre en compte un `app.name` modifié.

## Réinitialiser les écrans à partir du fichier de configuration

Les écrans résident dans la base de données après le premier démarrage. Pour les
réinitialiser à partir d'un bloc `[[screens]]` hérité dans `config.toml`,
supprimez les lignes de la table `screens` dans la base de données SQLite et
redémarrez le tableau de bord.

# Configuration

Après l'installation, la borne de flipper doit connaître ses écrans, ses périphériques ESP32, et ses utilisateurs. La plupart de cela se fait depuis la page **System** du tableau de bord par un administrateur. Cette page explique chaque paramètre.

## Fichier de configuration de la borne

`/etc/fliphetic/config.toml` contient le petit ensemble de paramètres qui changent rarement :

```toml
cab_id    = "cab-0"
bind_host = "100.x.y.z"
bind_port = 8080

apps_dir     = "/var/lib/fliphetic/apps"
state_db     = "/var/lib/fliphetic/state.sqlite"
screens_dir  = "/var/lib/fliphetic/screens"
current_link = "/var/lib/fliphetic/current"
```

Tout le reste (écrans, périphériques ESP32, utilisateurs, application par défaut au démarrage, paramètre d'inscription) réside dans la base de données d'état SQLite et est géré au moment de l'exécution.

## Écrans

Un écran a un **rôle** (le nom que les étudiants ciblent dans leurs manifestes d'application, par exemple `playfield`), une **sortie** xrandr, et une géométrie (position et taille).

### Ajouter des écrans

Ouvrez le tableau de bord, allez dans **System**, et trouvez la section **Screens**. Il y a deux façons d'ajouter un écran :

* **Détection depuis xrandr.** Le tableau « Detected » répertorie chaque sortie connectée qui n'est pas encore enregistrée, avec sa géométrie actuelle déjà renseignée. Saisissez un nom de rôle et cliquez sur Add.
* **Ajout manuel.** Utilisez le formulaire « Add screen » pour saisir le rôle, la sortie, la position et la taille à la main.

Au premier démarrage du tableau de bord, si le fichier de configuration contenait un ancien bloc `[[screens]]`, ces écrans sont copiés automatiquement dans la base de données. Ensuite, la base de données est l'unique source de vérité.

### Rôles d'écran

Les noms de rôle sont en minuscules avec des tirets. La borne de référence utilise `playfield`, `backglass`, et `dmd`. Quels que soient les rôles que vous choisissez, ils doivent correspondre :

* aux instances du service de kiosque que vous avez activées (`fliphetic-kiosk@<role>.service`),
* aux clés `[screens]` que les étudiants utilisent dans leurs manifestes d'application.

## Périphériques ESP32

Chaque ESP32 câblé à la borne est enregistré avec un **nom**. Les étudiants ciblent ce nom dans leur manifeste d'application. L'administrateur décide à quel port USB physique chaque nom correspond.

### Ajouter des périphériques

Dans **System**, trouvez la section **ESP32 devices**.

* **Detected.** Le tableau « Detected » répertorie chaque périphérique série USB dans `/dev/serial/by-id/` qui n'est pas encore enregistré. Chaque ligne propose un nom dérivé du descripteur USB. Modifiez-le si vous le souhaitez puis cliquez sur Add, ou cliquez sur **Auto-register all** pour ajouter tous les périphériques détectés d'un coup.
* **Ajout manuel.** Utilisez le formulaire « Add device » pour un contrôle complet sur le nom, le port, le type de puce, et le débit en bauds.

Préférez toujours un chemin `/dev/serial/by-id/...` à `/dev/ttyUSB0`. Le chemin by-id est stable entre les redémarrages et entre les ordres de branchement.

### Champs du périphérique

| Champ | Signification |
|-------|---------|
| Name | Identifiant en minuscules avec tirets. Les étudiants y font référence sous la forme `[esp32.<name>]`. |
| Port | Chemin du périphérique, idéalement sous `/dev/serial/by-id/`. |
| Chip | `esp32`, `esp32s2`, `esp32s3`, `esp32c3`, ou `esp32c6`. |
| Baud | Débit en bauds pour le flashage. `921600` est une bonne valeur par défaut. |

### Ligne de commande

Les mêmes opérations sont disponibles depuis la ligne de commande de la borne :

```sh
fliphetic esp32 list
fliphetic esp32 detect
fliphetic esp32 add buttons /dev/serial/by-id/usb-...  --chip esp32
fliphetic esp32 remove buttons
```

## Utilisateurs et rôles

Fliphetic a deux rôles.

| Rôle | Peut faire |
|------|--------|
| `admin` | Tout : enregistrer et désenregistrer des applications, récupérer les mises à jour, gérer les écrans, les périphériques ESP32, les utilisateurs, le paramètre d'inscription, et l'application par défaut au démarrage. |
| `user` | Se connecter, consulter les applications et les informations système, charger et arrêter n'importe quelle application. |

De plus, le **propriétaire** d'une application (l'utilisateur qui l'a enregistrée) peut récupérer les mises à jour et désenregistrer cette application précise, même sans le rôle admin.

### Gérer les utilisateurs depuis la ligne de commande

```sh
fliphetic users list
fliphetic users add alice --admin       # create an administrator
fliphetic users add bob                 # create a normal user
fliphetic users passwd bob              # change a password
fliphetic users role bob admin          # change a role
fliphetic users delete bob
```

Chaque commande `add` et `passwd` demande le mot de passe sauf si vous passez `--password`.

### Inscription en libre-service

Les administrateurs peuvent ouvrir ou fermer l'inscription en libre-service depuis la page **System**.

* **Ouverte.** Une page `/signup` accepte les nouveaux comptes. Chaque compte créé de cette manière est un `user` normal, jamais un admin. La page de connexion affiche un lien vers celle-ci.
* **Fermée.** La page `/signup` indique que l'inscription est fermée. C'est la valeur par défaut.

Ouvrez l'inscription au début d'un cours pour que les étudiants puissent créer leurs propres comptes, puis refermez-la ensuite.

## Application par défaut au démarrage

La borne peut charger automatiquement une application choisie à la mise sous tension.

* Si une **application par défaut au démarrage** est définie, la borne la charge à chaque démarrage, quel que soit ce qui était chargé avant l'arrêt. C'est le bon choix pour une borne publique : un cycle d'alimentation revient toujours à une application connue, en veille ou en mode attractif.
* Si aucune application par défaut au démarrage n'est définie, la borne reprend ce qui était chargé avant son arrêt.

Définissez-la depuis la page d'une application dans le tableau de bord (« Set as boot default »), ou depuis la ligne de commande :

```sh
fliphetic default show
fliphetic default set <app-id>
fliphetic default clear
```

L'application choisie affiche un badge « boot default » dans le tableau de bord.

## Suite

Poursuivez avec [Exploitation](operations.md) pour le guide au quotidien.

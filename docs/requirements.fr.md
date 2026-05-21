# Prérequis

Cette page liste tout ce dont vous avez besoin avant de provisionner une borne
de flipper Fliphetic.

## Matériel

| Élément | Notes |
|------|-------|
| PC de la borne | Une machine x86 64 bits. La borne de référence exécute Ubuntu 24.04 sur un processeur multicœur avec 8 Go de RAM ou plus. Un GPU dédié aide à piloter l'écran 4K en mode portrait. |
| Trois écrans | La disposition de référence comprend un playfield 4K en mode portrait, un backglass 1080p et un dmd 1080p. N'importe quelles trois sorties conviennent ; vous configurez leur géométrie plus tard. |
| Carte(s) ESP32 | Optionnel. Une ou plusieurs cartes ESP32 reliées aux boutons de la borne. Chacune se connecte en USB. Les applications qui n'utilisent pas de boutons peuvent fonctionner sans aucune carte. |
| Câblage USB | Un câble USB de données fiable par ESP32. Les câbles de charge seule ne fonctionneront pas pour le flashage. |
| Réseau | Une connectivité filaire ou sans fil pour la borne, ainsi qu'un accès internet pour Tailscale, les téléchargements d'images Docker et git. |

### Disposition d'affichage de référence

La borne de référence utilise ces trois sorties. Votre géométrie sera
différente, mais ceci montre la forme d'une configuration typique.

| Rôle | Sortie | Résolution | Position | Orientation |
|------|--------|------------|----------|-------------|
| playfield | HDMI-1 | 3840x2160 | 0,2160 | pivoté à gauche (portrait) |
| backglass | DP-0 | 1920x1080 | 240,0 | normale (principal) |
| dmd | HDMI-0 | 1920x1080 | 240,1080 | normale |

Vous lisez ces valeurs avec `xrandr` pendant la configuration et vous les
enregistrez dans le tableau de bord.

## Logiciels

| Logiciel | Version | Notes |
|----------|---------|-------|
| Ubuntu | 24.04 LTS | La plateforme de référence. D'autres distributions Linux modernes peuvent fonctionner mais ne sont pas documentées ici. |
| GNOME sur Xorg | livré avec 24.04 | Le placement des fenêtres de kiosque utilise des outils X11 (`xrandr`, `wmctrl`). Utilisez la session Xorg, pas Wayland. |
| Docker Engine | 24 ou plus récent | Avec le plugin Compose v2. |
| Chromium | version actuelle | Installé en tant que snap Ubuntu. Google Chrome fonctionne également. |
| Python | 3.12 | Livré avec Ubuntu 24.04. |
| Tailscale | version actuelle | Pour un accès distant privé au tableau de bord. |

Fliphetic lui-même est une application Python installée avec `uv`. La liste
complète des paquets est installée pendant l'[étape d'amorçage](install.md).

## Comptes et accès

* **Un utilisateur `flipper` sur la borne** avec les droits sudo. Tous les
  services Fliphetic s'exécutent sous cet utilisateur. Le nom `flipper` est
  utilisé tout au long de cette documentation.
* **Un compte Tailscale** et la borne rattachée à votre tailnet. Les étudiants
  et les opérateurs qui gèrent la borne doivent également être sur le tailnet,
  ou bien vous pouvez exposer le tableau de bord d'une autre manière.
* **Un compte ou une organisation GitHub** pour héberger les dépôts des
  applications étudiantes. Les dépôts publics sont les plus simples car la borne
  les clone via HTTPS sans identifiants.
* **Un poste de travail** (macOS ou Linux) qui peut atteindre la borne par SSH.
  Vous exécutez le script de déploiement depuis ce poste de travail.

## Modèle de réseau

Le tableau de bord écoute uniquement sur l'adresse IP Tailscale de la borne.
Cela a deux conséquences :

* Toute personne sur votre tailnet peut atteindre le tableau de bord. Toute
  personne qui n'est pas sur le tailnet ne le peut pas, même sur le même réseau
  local.
* La borne doit terminer la mise en service de son interface Tailscale avant
  que le tableau de bord puisse démarrer. Le service Fliphetic gère cela avec
  une attente au démarrage, décrite dans [Installer Fliphetic](install.md).

Les dépôts des applications étudiantes sont clonés via l'internet public, donc
la borne a besoin d'une connexion internet normale en plus de Tailscale.

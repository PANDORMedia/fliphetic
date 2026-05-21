# Fliphetic

Fliphetic est un système de déploiement et de gestion pour une borne de flipper
virtuelle. Il est conçu pour un contexte pédagogique : les étudiants développent
des « applications » de flipper dans leurs propres dépôts git, et Fliphetic vous
permet de charger n'importe laquelle d'entre elles sur la borne physique depuis
un tableau de bord web unique.

## Ce qu'il fait

Une borne de flipper Fliphetic comporte trois écrans en mode kiosque Chromium et
une ou plusieurs cartes ESP32 reliées aux boutons de la borne. Fliphetic exécute
un petit tableau de bord sur la borne qui vous permet de :

* **Enregistrer** une application étudiante à partir de son URL git.
* **Surveiller** les dépôts enregistrés et faire remonter automatiquement les
  nouvelles versions.
* **Charger** n'importe quelle application enregistrée, ce qui arrête
  l'application précédente, flashe le firmware de l'ESP32, démarre les services
  Docker de l'application et pointe chaque kiosque Chromium vers la bonne URL.
* **Gérer** la borne : les écrans, les périphériques ESP32, les utilisateurs et
  l'application qui se charge automatiquement à la mise sous tension.

## L'architecture en un coup d'œil

```
+--------------------- cabinet (flipper@cab) -----------------------+
|                                                                   |
|  +--------------+    +----------------+   +-------------------+    |
|  |  dashboard   |<-->|  orchestrator  |-->|  3x chromium      |    |
|  |  (FastAPI +  |    |  - load/stop   |   |  --kiosk windows  |    |
|  |   HTMX)      |    |  - app state   |   |  (systemd user    |    |
|  |              |    |  - run logs    |   |   services)       |    |
|  +--------------+    +-------+--------+   +-------------------+    |
|         ^                    |                                   |
|         |                    +--> docker compose -p <app>        |
|  +------+-------+             |                                   |
|  |  watchdog    |             +--> esptool flash -> ESP32(s)      |
|  |  (git poll)  |                                                 |
|  +--------------+                                                 |
|         |                                                         |
|         v                                                         |
|   /var/lib/fliphetic/apps/<app-id>/   (cloned repositories)        |
|                                                                   |
+-------------------------------------------------------------------+
        ^
        |  Tailscale (you and students reach the dashboard)
        |
   browser -> dashboard
```

Le tableau de bord écoute uniquement sur l'adresse Tailscale de la borne, de
sorte qu'il est accessible depuis votre tailnet sans rien exposer à l'internet
public.

## Les composants

| Composant | Rôle |
|-----------|------|
| Tableau de bord | Interface web et API HTTP, s'exécute comme un service systemd sur la borne. |
| Orchestrateur | Effectue les opérations de chargement, d'arrêt et de statut. Sérialisé pour que deux chargements ne puissent pas s'entremêler. |
| Service de surveillance | Interroge chaque dépôt git enregistré à la recherche de nouvelles versions. |
| Services de kiosque | Un service utilisateur systemd par écran, exécutant chacun une fenêtre Chromium. |
| Stockage | Une base de données SQLite pour le registre, l'historique des exécutions, les utilisateurs, les écrans et les périphériques ESP32. |

## Comment lire cette documentation

Si vous configurez une borne de zéro, lisez les pages dans l'ordre :

1. [Prérequis](requirements.md) couvre le matériel, les logiciels et les comptes.
2. La section **Configuration de la borne** met en place le système
   d'exploitation, le réseau, les écrans, le comportement à la mise sous tension
   et l'habillage de démarrage.
3. [Installer Fliphetic](install.md) installe le tableau de bord lui-même.
4. [Configuration](configuration.md) enregistre les écrans, les périphériques
   ESP32 et les utilisateurs.
5. [Exploitation](operations.md) est le guide quotidien.

Si vous êtes un étudiant qui développe une application, allez directement à
[Développer des applications](apps.md) et [Firmware ESP32](esp32-firmware.md).

La documentation est disponible en anglais et en français. Utilisez le sélecteur
de langue dans la barre supérieure pour changer.

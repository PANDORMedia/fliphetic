# Architecture

Cette page décrit le fonctionnement interne de Fliphetic. Elle est utile pour
étendre le système ou diagnostiquer un comportement inhabituel.

## Composants

Fliphetic est une seule application Python. À l'intérieur :

| Module | Responsabilité |
|--------|----------------|
| Dashboard | Une application FastAPI : l'interface web et l'API HTTP. |
| Orchestrator | Effectue le chargement, l'arrêt et le statut. Le seul endroit qui pilote Docker, esptool et les kiosques. |
| Watchdog | Une tâche asynchrone qui interroge les dépôts enregistrés pour détecter de nouvelles références. |
| Kiosk | Écrit les fichiers d'URL par écran et contrôle les services de kiosque Chromium. |
| Storage | Une fine couche SQLite pour le registre, l'historique d'exécution, les utilisateurs, les écrans et les périphériques ESP32. |
| Manifest | Analyse et valide `fliphetic.toml`. |

Le tableau de bord s'exécute en tant que service système `fliphetic.service`.
Chaque écran exécute une fenêtre Chromium en tant que service utilisateur
`fliphetic-kiosk@<role>.service`.

## Le cycle de vie du chargement

Charger une application est l'opération centrale. Dans l'ordre :

1. **Écran d'accueil.** Chaque écran est dirigé vers une page d'accueil interne
   afin que l'utilisateur ait un retour immédiat.
2. **Récupération.** Le dépôt de l'application est récupéré et basculé sur la
   référence cible (la tête de branche ou le tag correspondant le plus récent).
3. **Manifeste.** `fliphetic.toml` est lu. Le nom d'affichage stocké est
   synchronisé depuis `app.name`.
4. **Arrêt du précédent.** Les services Docker de l'application précédemment
   chargée sont arrêtés.
5. **Flashage.** Chaque bloc `[esp32.<name>]` est flashé avec esptool. Un
   périphérique requis manquant interrompt le chargement.
6. **Compose up.** Les services Docker de l'application sont démarrés et
   disposent d'un délai pour passer leurs tests de santé.
7. **Résolution des écrans.** Chaque cible d'écran est transformée en une URL
   atteignable depuis l'hôte. Les cibles `service` sont résolues via
   `docker compose port` ; chaque service et port distinct est interrogé une
   seule fois.
8. **Bascule des kiosques.** Les URL résolues sont écrites dans les fichiers par
   écran et les services de kiosque sont redémarrés.
9. **Enregistrement.** L'exécution est sauvegardée avec son journal complet, et
   l'application devient l'application courante.

Si une étape échoue, l'exécution est marquée comme échouée, l'application
partiellement démarrée est arrêtée, et l'application précédemment sélectionnée
demeure.

## Sérialisation

`load()` et `stop()` sont sérialisés avec un verrou. Des appelants concurrents
(le chargement automatique au démarrage qui entre en concurrence avec un clic
sur le tableau de bord, ou deux opérateurs qui cliquent sur Charger en même
temps) entrelaceraient sinon les opérations Docker et de kiosque et laisseraient
la borne dans un état incohérent. Avec le verrou, une seconde requête attend que
la première se termine, puis s'exécute.

## Stockage

L'état réside dans une seule base de données SQLite, par défaut
`/var/lib/fliphetic/state.sqlite`. Les principales tables sont :

| Table | Contenu |
|-------|----------|
| `apps` | Applications enregistrées : id, nom, URL git, stratégie de déploiement, SHA courant et le plus récent, propriétaire. |
| `runs` | Historique des chargements, arrêts et flashages avec les journaux complets. |
| `users` | Comptes : nom d'utilisateur, empreinte scrypt du mot de passe, rôle. |
| `screens` | Rôle d'écran, sortie et géométrie. |
| `esp32_devices` | Nom du périphérique, port, puce, débit en bauds. |
| `cab_state` | Paires clé / valeur : l'application courante, la valeur par défaut au démarrage, le paramètre d'inscription, le secret de session. |

Les dépôts clonés résident sous `/var/lib/fliphetic/apps/<app-id>/`. Un lien
symbolique `current` pointe vers l'application chargée.

## L'approche kiosque

Chaque écran exécute une fenêtre Chromium indépendante. Le service de kiosque
d'un rôle lit le fichier d'URL de ce rôle et lance Chromium avec `--app=<url>`,
ce qui supprime tout l'habillage du navigateur.

La fenêtre est ensuite placée et passée en plein écran par le gestionnaire de
fenêtres via `wmctrl`, plutôt que par le mode kiosque propre à Chromium. C'est
délibéré :

* L'option `--kiosk` de Chromium ne respecte pas de manière fiable un moniteur
  cible sur un bureau GNOME multi-écrans, si bien que les fenêtres peuvent se
  rabattre sur l'écran principal.
* Laisser le gestionnaire de fenêtres passer la fenêtre en plein écran évite la
  notification de Chromium « appuyez sur une touche pour quitter le plein
  écran », car Chromium lui-même n'entre jamais dans son état plein écran.

Une courte routine d'arrière-plan attend l'apparition de la fenêtre, puis la
positionne et la passe en plein écran sur la bonne sortie.

## Le service de surveillance

Le service de surveillance est une tâche asynchrone dans le processus du tableau
de bord. Environ une fois par minute, pour chaque application enregistrée, il
récupère le dépôt et résout la référence cible. Si cette référence diffère de ce
qui a été vu la dernière fois, l'application est signalée par un badge de mise à
jour. Le service de surveillance ne charge jamais rien ; c'est un opérateur qui
décide quand appliquer une mise à jour.

## Réseau et démarrage

Le tableau de bord ne se lie qu'à l'adresse Tailscale de la borne. Le service
attend que cette adresse existe avant de démarrer. Au démarrage, une fois le
tableau de bord disponible, il charge automatiquement une application : la
valeur par défaut au démarrage si elle est définie, sinon l'application qui
était chargée avant l'arrêt.

## Authentification

Le tableau de bord exige une connexion. Les sessions sont des cookies signés ;
le secret de signature est généré une fois par borne et stocké dans
`cab_state`. Les mots de passe sont hachés avec scrypt. Il existe deux rôles,
`admin` et `user`, plus une relation de propriétaire par application. Voir
[Configuration](configuration.md) pour savoir ce que chacun peut faire.

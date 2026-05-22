# Exploitation

Cette page est le guide au quotidien pour exploiter une borne de flipper Fliphetic une fois qu'elle est configurée.

## Le tableau de bord

Ouvrez `http://100.x.y.z:8080` sur un appareil de votre tailnet et connectez-vous. La barre supérieure comporte trois sections :

* **Apps** répertorie chaque application enregistrée.
* **System** affiche les informations de la borne, les écrans, et les périphériques ESP32.
* **+ Register** (administrateurs uniquement) enregistre une nouvelle application.

L'utilisateur courant et son rôle sont affichés à droite, avec un lien de déconnexion.

## Enregistrer une application

Une application est un dépôt git qui contient un manifeste `fliphetic.toml` valide à sa racine. Pour en enregistrer une, un administrateur ouvre **+ Register**, colle l'URL git, et valide.

Fliphetic clone le dépôt, valide le manifeste, et ajoute l'application à la liste. Si le manifeste est absent ou invalide, l'enregistrement échoue avec la raison affichée, et rien n'est ajouté.

Les dépôts publics sont les plus simples car la borne clone via HTTPS sans identifiants.

## Variables d'environnement

Les conteneurs Docker d'une application peuvent recevoir des variables d'environnement — URL d'API, indicateurs de fonctionnalité, identifiants — sans les placer dans le dépôt de l'application. Elles sont gérées ici, sur la borne, et stockées dans sa base de données, jamais dans `fliphetic.toml`, ce qui garde les secrets hors du dépôt (généralement public) de l'application.

Un administrateur ou le propriétaire de l'application les définit dans la section **Environment variables** de la page de l'application. Chaque variable comporte :

* un **conteneur** — le service Compose dans lequel elle est injectée, ou *tous les conteneurs* ;
* un **nom** — lettres, chiffres et tirets bas ;
* une **valeur**.

Le tableau de bord lit la liste des conteneurs de l'application depuis son fichier Compose, donc le sélecteur de conteneur liste les services réels. Soumettre à nouveau une variable avec le même conteneur et le même nom met à jour sa valeur ; le bouton **Remove** en supprime une.

Les variables peuvent aussi être définies lors du premier enregistrement de l'application : le formulaire **+ Register** comporte une zone de variables d'environnement où vous saisissez une paire `KEY=VALUE` par ligne. Celles-ci s'appliquent à tous les conteneurs et peuvent être re-ciblées par conteneur ensuite.

Les variables d'environnement prennent effet au prochain **Load** de l'application. Les modifier n'affecte pas une application en cours d'exécution tant qu'elle n'est pas rechargée.

## Charger et arrêter une application

Depuis la liste Apps ou la page d'une application, cliquez sur **Load**. Le chargement effectue ces étapes dans l'ordre, et est sérialisé afin que deux chargements ne puissent jamais se chevaucher :

1. Récupérer le dernier code de la branche ou du tag configuré pour l'application.
2. Arrêter les services Docker de l'application précédemment chargée.
3. Flasher tout firmware ESP32 que l'application déclare.
4. Démarrer les services Docker de l'application avec Compose.
5. Résoudre chaque écran vers une URL et y faire pointer les kiosques.

Pendant l'exécution d'un chargement, les écrans affichent un écran de présentation qui reflète l'étape en cours. Quand il se termine, les écrans basculent sur l'application.

Cliquez sur **Stop** pour arrêter l'application courante et éteindre les écrans.

## Suivre un chargement

Chaque chargement est enregistré comme une exécution sur la page de l'application, avec un statut et un journal complet. Si un chargement échoue, ouvrez la page de l'application et lisez le journal d'exécution. Chaque étape y écrit, donc l'étape en échec et son erreur y sont visibles.

## Le service de surveillance

Fliphetic interroge chaque dépôt enregistré environ une fois par minute. Pour chaque application, il récupère le dépôt et résout la référence cible (le sommet de la branche, ou le tag le plus récent correspondant à un motif glob, selon la stratégie de déploiement de l'application).

Quand une référence plus récente est trouvée, l'application affiche un badge **update** dans la liste Apps. Le service de surveillance ne charge jamais rien de lui-même. Un opérateur décide quand appliquer une mise à jour en cliquant sur **Load**, qui récupère et extrait toujours la dernière cible avant de s'exécuter.

## Récupérer sans charger

Le bouton **Pull latest** sur la page d'une application récupère et extrait la dernière référence cible sans démarrer l'application. C'est utile pour inspecter le nouveau manifeste, ou pour rafraîchir le nom d'affichage stocké de l'application après que son `app.name` a changé. La récupération est disponible pour les administrateurs et pour le propriétaire de l'application.

## Changer d'application

Charger une application différente arrête automatiquement la courante d'abord. Vous n'avez pas besoin d'arrêter avant de charger. Comme les chargements sont sérialisés, cliquer sur Load pour deux applications en succession rapide les exécute l'une après l'autre, et la borne se termine sur la seconde.

## Comportement au redémarrage

À la mise sous tension, la borne :

1. Active le réseau et Tailscale.
2. Démarre le service du tableau de bord Fliphetic une fois que l'adresse Tailscale existe.
3. Charge automatiquement une application : l'application par défaut au démarrage si elle est définie, sinon l'application qui était chargée avant l'arrêt.
4. Active les trois fenêtres de kiosque et les fait pointer vers l'application chargée.

Voir [Configuration](configuration.md) pour le paramètre de l'application par défaut au démarrage.

## Désenregistrer une application

**Unregister** sur la page d'une application arrête l'application si elle est en cours d'exécution, la retire du registre, et supprime son dépôt cloné de la borne. Si l'application était l'application par défaut au démarrage, ce paramètre est également effacé. Le désenregistrement est disponible pour les administrateurs et pour le propriétaire de l'application.

## Opérations utiles en ligne de commande

Exécutez celles-ci sur la borne, en tant que `flipper` :

```sh
fliphetic list                  # registered apps
fliphetic status                # what is running, per screen state
fliphetic load <app-id>         # load an app
fliphetic stop                  # stop the current app
fliphetic validate <path>       # check a manifest without registering
```

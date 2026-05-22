# Créer des applications

Une application Fliphetic est un dépôt git que la borne de flipper peut cloner,
valider et exécuter sur ses trois écrans, avec un firmware ESP32 facultatif pour
les boutons. Cette page constitue le contrat : suivez-la et la borne chargera
votre application.

## En bref

1. Placez un fichier `fliphetic.toml` à la racine de votre dépôt.
2. Fournissez un fichier Docker Compose qui sert le contenu de vos écrans.
3. Vous pouvez aussi committer un binaire de firmware ESP32 précompilé et faire
   pointer le manifeste vers celui-ci.
4. Poussez vers un dépôt git public. Un administrateur enregistre son URL sur la
   borne.

## Les trois écrans

La borne comporte trois écrans. Votre application les désigne par leur **rôle**,
et non par leur port physique. La borne de référence utilise ces rôles :

| Rôle | Écran typique |
|------|----------------|
| `playfield` | Le grand écran en mode portrait |
| `backglass` | L'écran supérieur en mode paysage |
| `dmd` | L'écran inférieur en mode paysage |

Vous n'êtes pas obligé d'utiliser les trois. Déclarez uniquement les rôles dont
votre application a besoin. Les rôles que vous omettez sont laissés vides. Les
rôles exacts disponibles sur une borne donnée sont visibles sur la page System
de son tableau de bord.

## `fliphetic.toml`

```toml
[app]
id          = "moby-pinball"        # lowercase, hyphens, 3 to 64 characters, unique
name        = "Moby Pinball"        # display name shown in the dashboard
authors     = ["Student Name"]
description = "A whale of a pinball game"

[deploy]
strategy = "branch"                 # "branch" or "tag"
ref      = "main"                   # branch name, or a tag glob like "v*"

[screens]
playfield = { service = "game",  port = 8080 }
backglass = { service = "art",   port = 80 }
dmd       = { service = "score", port = 80 }

[compose]
file          = "deploy/docker-compose.yml"
ready_timeout = 30

[esp32.buttons]
firmware = "firmware/build/buttons.bin"
chip     = "esp32"
```

### `[app]`

`id` est la clé canonique de votre application sur la borne. Choisissez-la une
fois et ne la changez pas : la borne l'utilise pour les noms de répertoires et
les noms de projet Docker. `name` est le libellé lisible par un humain, et il
peut changer librement (le tableau de bord le relit lorsque l'application est
mise à jour ou chargée).

### `[deploy]`

Contrôle quelle référence le service de surveillance suit et que la borne
charge.

* `strategy = "branch"` suit la tête de `ref` (un nom de branche).
* `strategy = "tag"` suit le tag le plus récent correspondant à `ref` utilisé
  comme glob, par exemple `v*`.

### `[screens]`

Une table associant un rôle d'écran à une cible. Chaque cible définit
exactement l'un des éléments suivants :

* `url` pour une URL absolue, ou
* `service` pour un nom de service Docker Compose, avec un `port` facultatif
  (le port du conteneur, 80 par défaut) et un `path` facultatif ajouté à l'URL
  résolue.

### `[compose]`

`file` est le chemin de votre fichier Compose, relatif à la racine du dépôt.
`ready_timeout` est le nombre de secondes pendant lesquelles la borne attend
que les services soient sains avant de déclarer le chargement terminé.

### `[esp32.<name>]`

Facultatif, un bloc par périphérique ESP32 de la borne que vous ciblez. Voir
[Firmware ESP32](esp32-firmware.md) pour tous les détails.

## Le fichier Docker Compose

Chaque service qui alimente un écran publie son port de conteneur vers l'hôte
afin que le Chromium de la borne puisse l'atteindre. Utilisez `"0:80"` pour que
Docker attribue un port hôte éphémère ; la borne le découvre après le démarrage
des services.

```yaml
services:
  game:
    image: nginx:alpine
    volumes: ["../site:/usr/share/nginx/html:ro"]
    ports: ["0:80"]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost/"]
      interval: 5s
      timeout: 2s
      retries: 5
```

Conventions :

* Les tests de santé sont fortement recommandés. La borne attend que les
  services soient sains avant de basculer les écrans vers votre application.
* Ne vous liez pas à des ports hôtes fixes. Utilisez `0:<container_port>`.
* Les chemins de volume relatifs sont résolus par rapport au répertoire du
  fichier Compose.

## Variables d'environnement

Les conteneurs de votre application peuvent recevoir des variables
d'environnement qui ne font **pas** partie de votre dépôt. Elles sont définies
par application sur le tableau de bord de la borne, par la personne qui déploie
l'application — voir [Exploitation](operations.md).

C'est là que doit aller tout ce que vous ne devez pas committer — clés d'API et
autres secrets — ainsi que les valeurs qui diffèrent d'une borne à l'autre. Ne
les mettez pas dans `fliphetic.toml` ni dans votre fichier Compose : votre dépôt
est public.

Dans un conteneur, lisez les variables de la manière habituelle : `process.env`
en Node, `os.environ` en Python, et ainsi de suite. Documentez les variables que
votre application attend pour que la personne qui la déploie sache quoi définir,
et prévoyez une valeur par défaut raisonnable lorsqu'une variable est absente.

## Trois façons de relier Docker aux écrans

La borne prend en charge les trois topologies.

### Un service par écran

Trois services, trois écrans :

```toml
[screens]
playfield = { service = "playfield", port = 80 }
backglass = { service = "backglass", port = 80 }
dmd       = { service = "dmd",       port = 80 }
```

### Un seul service pour tous les écrans

Un service unique sert un contenu différent par écran, distingué par `path` :

```toml
[screens]
playfield = { service = "web", port = 80, path = "/playfield" }
backglass = { service = "web", port = 80, path = "/backglass" }
dmd       = { service = "web", port = 80, path = "/dmd" }
```

`path` peut aussi être une chaîne de requête, par exemple
`path = "/?screen=playfield"`.

### Mixte

Plusieurs services, avec des écrans associés librement. Deux écrans peuvent
partager un même service via des chemins ou des ports différents tandis qu'un
troisième utilise son propre service. La borne interroge chaque service et port
distinct une seule fois.

## Ce que fait « Charger »

1. Récupérer le code le plus récent pour votre référence `[deploy]`.
2. Arrêter les services Docker de l'application précédente.
3. Flasher tout firmware ESP32 que votre manifeste déclare.
4. Démarrer vos services Docker et attendre les tests de santé.
5. Résoudre chaque cible d'écran en une URL et y faire pointer les kiosques.

Si une étape échoue, le chargement est marqué comme échoué et l'application
précédente reste sélectionnée. Ouvrez la page de l'application sur le tableau de
bord et lisez le journal d'exécution.

## Validation en local

Vérifiez votre manifeste avant de pousser :

```sh
fliphetic validate .
```

Cela signale les erreurs de schéma et les fichiers manquants.

## Erreurs courantes

| Erreur | Cause |
|-------|-------|
| `app.id must be lowercase` | L'`id` contient des lettres majuscules, des tirets bas, ou une longueur incorrecte. |
| `manifest references missing files` | Un chemin dans `[compose]` ou `[esp32]` n'existe pas dans le dépôt. |
| `docker compose up failed` | Une image ne se télécharge pas, un test de santé ne passe jamais, ou un port entre en collision. |
| `cab has no esp32 device named X` | Votre `[esp32.X]` cible un périphérique que la borne n'a pas enregistré. |
| `screen role X must be lowercase` | Un nom de rôle d'écran n'est pas en minuscules avec des traits d'union. |

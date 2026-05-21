# Exemples et recettes

Cette page présente des façons concrètes de structurer une application Fliphetic,
depuis un dépôt unique jusqu'à un projet assemblé à partir de plusieurs dépôts
distincts.

## Un dépôt Fliphetic est petit

Un dépôt Fliphetic n'a besoin que de deux choses à sa racine :

* `fliphetic.toml`, le manifeste.
* un fichier Docker Compose (par défaut `deploy/docker-compose.yml`).

Tout ce que les écrans affichent atteint la borne de l'une de ces deux manières :

* **Construit à partir d'un Dockerfile** présent dans le dépôt. Convient à un
  petit projet autonome.
* **Récupéré sous forme d'image préconstruite** depuis un registre comme Docker
  Hub ou GHCR. Convient lorsque le vrai code vit dans son propre dépôt, ou
  lorsque plusieurs dépôts sont combinés en une seule application.

Les recettes ci-dessous couvrent les deux cas.

## Recette 1 : Monorepo

Tout dans un seul dépôt. Le dépôt porte le manifeste, le fichier Compose, le
contenu des écrans et tous les Dockerfiles.

```
my-pinball/
  fliphetic.toml
  deploy/docker-compose.yml
  site/
    index.html
    ...
```

`deploy/docker-compose.yml` construit ou sert le contenu directement :

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

C'est la disposition la plus simple et un bon point de départ. L'inconvénient :
la borne construit ou assemble le contenu au moment du chargement.

## Recette 2 : Un Dockerfile dans le dépôt

Lorsque votre application a besoin de plus que de fichiers statiques, ajoutez un
Dockerfile et laissez Compose le construire.

`Dockerfile` :

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
```

`deploy/docker-compose.yml` :

```yaml
services:
  game:
    build:
      context: ..
      dockerfile: Dockerfile
    ports: ["0:8080"]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/health"]
      interval: 5s
      timeout: 3s
      retries: 10
```

Le `context: ..` pointe vers la racine du dépôt, car le fichier Compose se
trouve dans `deploy/`.

## Recette 3 : Code dans un dépôt distinct, image sur Docker Hub

C'est le modèle recommandé pour les projets réels. Le code de votre application
reste dans son propre dépôt. Un workflow d'intégration continue construit une
image Docker et la pousse vers un registre. Le dépôt Fliphetic est alors minime :
un manifeste et un fichier Compose qui récupère cette image.

### Étape 1 : le dépôt de code

Dans le dépôt de votre application (par exemple `github.com/student/space-game`),
ajoutez un Dockerfile :

```dockerfile
FROM nginx:alpine
COPY site/ /usr/share/nginx/html/
```

### Étape 2 : construire et pousser l'image automatiquement

Ajoutez un workflow GitHub Actions dans `.github/workflows/image.yml` qui
construit l'image et la pousse vers Docker Hub à chaque push sur `main` :

```yaml
name: Build and push image

on:
  push:
    branches: [main]

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: student/space-game:latest
```

Créez un compte Docker Hub, générez un jeton d'accès dans Account Settings,
puis ajoutez deux secrets de dépôt dans GitHub sous
**Settings > Secrets and variables > Actions** :

* `DOCKERHUB_USERNAME`, votre nom d'utilisateur Docker Hub.
* `DOCKERHUB_TOKEN`, le jeton d'accès.

Chaque push sur `main` publie désormais `student/space-game:latest` sur Docker
Hub.

### Étape 3 : le dépôt Fliphetic

Créez un second dépôt, petit, par exemple
`github.com/student/space-game-cab`. Il ne contient que le manifeste et le
fichier Compose.

`fliphetic.toml` :

```toml
[app]
id   = "space-game"
name = "Space Game"

[screens]
playfield = { service = "game", port = 80 }

[compose]
file = "deploy/docker-compose.yml"
```

`deploy/docker-compose.yml` :

```yaml
services:
  game:
    image: student/space-game:latest
    pull_policy: always
    ports: ["0:80"]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost/"]
      interval: 5s
      timeout: 2s
      retries: 5
```

Enregistrez `space-game-cab` sur la borne. Lorsque vous le chargez, la borne
récupère la dernière image et la démarre.

!!! note "pull_policy: always"
    `pull_policy: always` indique à Compose de récupérer l'image à chaque
    chargement de l'application. Sans cela, la borne continue d'utiliser la
    première image récupérée même après que vous en ayez poussé une nouvelle.
    Utilisez-le chaque fois qu'un service référence une image de registre par
    une étiquette mouvante comme `latest`.

## Recette 4 : Combiner plusieurs dépôts

Une équipe répartit souvent le travail entre plusieurs dépôts : un pour le
playfield, un pour le backglass, un pour le DMD. Chaque dépôt construit et pousse
sa propre image, exactement comme dans la Recette 3. Le dépôt Fliphetic récupère
ensuite chacune d'elles.

`deploy/docker-compose.yml` dans le dépôt Fliphetic :

```yaml
services:
  playfield:
    image: team/sp-playfield:latest
    pull_policy: always
    ports: ["0:80"]
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost/"]
      interval: 5s
      timeout: 2s
      retries: 5
  backglass:
    image: team/sp-backglass:latest
    pull_policy: always
    ports: ["0:80"]
  dmd:
    image: team/sp-dmd:latest
    pull_policy: always
    ports: ["0:80"]
```

`fliphetic.toml` :

```toml
[app]
id   = "space-pinball"
name = "Space Pinball"

[screens]
playfield = { service = "playfield", port = 80 }
backglass = { service = "backglass", port = 80 }
dmd       = { service = "dmd",       port = 80 }

[compose]
file = "deploy/docker-compose.yml"
```

Chaque équipe de composant travaille dans son propre dépôt, à son propre rythme.
Le dépôt Fliphetic est de la pure orchestration : il ne contient jamais leur
code, seulement le manifeste et le fichier Compose qui relie les images publiées
aux écrans.

## Utiliser GitHub Container Registry au lieu de Docker Hub

GHCR fonctionne de la même manière et ne nécessite aucun compte distinct. Poussez
vers `ghcr.io/<owner>/<image>` et référencez ce chemin dans le fichier Compose.
Le workflow se connecte avec le jeton intégré :

```yaml
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/student/space-game:latest
```

Rendez le paquet GHCR public afin que la borne puisse le récupérer sans
identifiants.

## Une note sur les sous-modules git

Vous pouvez techniquement ajouter d'autres dépôts comme sous-modules git, mais la
borne clone une application avec un simple `git clone` et ne récupère pas les
sous-modules. Le contenu derrière un sous-module sera manquant. Préférez
l'approche par image publiée des Recettes 3 et 4 : elle est plus rapide sur la
borne (un chargement ne fait que récupérer des images, il ne construit pas), et
elle est reproductible.

## Choisir une recette

| Situation | Recette |
|-----------|--------|
| Petit, statique, une personne | Recette 1, monorepo |
| Une application avec un backend | Recette 2, Dockerfile dans le dépôt |
| Le code vit dans son propre dépôt | Recette 3, dépôt distinct plus image |
| Plusieurs équipes, plusieurs dépôts | Recette 4, une image par composant |

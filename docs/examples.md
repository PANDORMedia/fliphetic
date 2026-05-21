# Examples and recipes

This page shows concrete ways to structure a Fliphetic app, from a single
repository to a project assembled from several separate repositories.

## A Fliphetic repository is small

A Fliphetic repository only needs two things at its root:

* `fliphetic.toml`, the manifest.
* a Docker Compose file (by default `deploy/docker-compose.yml`).

Everything the screens display reaches the cabinet in one of two ways:

* **Built from a Dockerfile** that lives in the repository. Good for a small,
  self contained project.
* **Pulled as a prebuilt image** from a registry such as Docker Hub or GHCR.
  Good when the real code lives in its own repository, or when several
  repositories are combined into one app.

The recipes below cover both.

## Recipe 1: Monorepo

Everything in one repository. The repository carries the manifest, the Compose
file, the screen content, and any Dockerfiles.

```
my-pinball/
  fliphetic.toml
  deploy/docker-compose.yml
  site/
    index.html
    ...
```

`deploy/docker-compose.yml` builds or serves the content directly:

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

This is the simplest layout and a good starting point. The downside: the cabinet
builds or assembles the content at load time.

## Recipe 2: A Dockerfile in the repository

When your app needs more than static files, add a Dockerfile and let Compose
build it.

`Dockerfile`:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
```

`deploy/docker-compose.yml`:

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

The `context: ..` points at the repository root, because the Compose file is in
`deploy/`.

## Recipe 3: Code in a separate repository, image on Docker Hub

This is the recommended pattern for real projects. Your application code stays
in its own repository. A continuous integration workflow builds a Docker image
and pushes it to a registry. The Fliphetic repository is then tiny: a manifest
and a Compose file that pulls that image.

### Step 1: the code repository

In your application repository (for example `github.com/student/space-game`),
add a Dockerfile:

```dockerfile
FROM nginx:alpine
COPY site/ /usr/share/nginx/html/
```

### Step 2: build and push the image automatically

Add a GitHub Actions workflow at `.github/workflows/image.yml` that builds the
image and pushes it to Docker Hub on every push to `main`:

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

Create a Docker Hub account, generate an access token under Account Settings,
then add two repository secrets in GitHub under
**Settings > Secrets and variables > Actions**:

* `DOCKERHUB_USERNAME`, your Docker Hub username.
* `DOCKERHUB_TOKEN`, the access token.

Every push to `main` now publishes `student/space-game:latest` to Docker Hub.

### Step 3: the Fliphetic repository

Create a second, small repository, for example
`github.com/student/space-game-cab`. It contains only the manifest and the
Compose file.

`fliphetic.toml`:

```toml
[app]
id   = "space-game"
name = "Space Game"

[screens]
playfield = { service = "game", port = 80 }

[compose]
file = "deploy/docker-compose.yml"
```

`deploy/docker-compose.yml`:

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

Register `space-game-cab` on the cabinet. When you load it, the cabinet pulls
the latest image and starts it.

!!! note "pull_policy: always"
    `pull_policy: always` tells Compose to pull the image every time the app
    loads. Without it, the cabinet keeps using the first image it pulled even
    after you push a new one. Use it whenever a service references a registry
    image by a moving tag such as `latest`.

## Recipe 4: Combining several repositories

A team often splits work across repositories: one for the playfield, one for the
backglass, one for the DMD. Each repository builds and pushes its own image,
exactly as in Recipe 3. The Fliphetic repository then pulls all of them.

`deploy/docker-compose.yml` in the Fliphetic repository:

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

`fliphetic.toml`:

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

Each component team works in its own repository at its own pace. The Fliphetic
repository is pure orchestration: it never holds their code, only the manifest
and the Compose file that wires the published images to screens.

## Using GitHub Container Registry instead of Docker Hub

GHCR works the same way and needs no separate account. Push to
`ghcr.io/<owner>/<image>` and reference that path in the Compose file. The
workflow logs in with the built in token:

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

Make the GHCR package public so the cabinet can pull it without credentials.

## A note on git submodules

You can technically add other repositories as git submodules, but the cabinet
clones an app with a plain `git clone` and does not fetch submodules. Content
behind a submodule will be missing. Prefer the published image approach in
Recipes 3 and 4: it is faster on the cabinet (a load only pulls images, it does
not build), and it is reproducible.

## Choosing a recipe

| Situation | Recipe |
|-----------|--------|
| Small, static, one person | Recipe 1, monorepo |
| One app with a backend | Recipe 2, Dockerfile in the repo |
| Code lives in its own repository | Recipe 3, separate repo plus image |
| Several teams, several repositories | Recipe 4, one image per component |

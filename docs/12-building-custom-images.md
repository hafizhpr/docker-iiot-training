# 12 — Bonus: Building Custom Images

In Lesson 07 you installed `node-red-contrib-influxdb` by hand through the
palette manager. It worked. It also has three problems:

1. Every student repeats it, and some mistype the name
2. It vanishes when the volume is removed
3. It is a manual step, so it cannot be automated or version-controlled

The fix is to build your own image with the node already inside.

## Dockerfile basics

A `Dockerfile` is a recipe. Each instruction adds a layer on top of the last.

```dockerfile
FROM nodered/node-red:4.1.0-22

RUN npm install --no-audit --no-fund --no-update-notifier \
    node-red-contrib-influxdb
```

| Instruction | Meaning |
|---|---|
| `FROM` | The base image to start from — always the first instruction |
| `RUN` | Execute a command **at build time** and keep the result in a layer |

Note *build time*. `RUN` happens once, when the image is built. It is not
executed when a container starts.

This file already exists in the repository at `nodered/Dockerfile`.

### Why `/usr/src/node-red` matters

The base image sets its working directory to `/usr/src/node-red`, where
Node-RED's own `package.json` lives. Installing there puts the node **inside
the image**.

Installing into `/data` instead would put it in the volume — which is exactly
the situation we are trying to escape.

## Common instructions

| Instruction | Purpose |
|---|---|
| `FROM` | Base image |
| `RUN` | Run a command during build |
| `COPY` | Copy files from your machine into the image |
| `WORKDIR` | Set the working directory for later instructions |
| `ENV` | Set an environment variable |
| `EXPOSE` | Document which port the app listens on |
| `CMD` | Default command when a container starts |

`EXPOSE` documents; it does not publish. Publishing is still `-p` or the
compose `ports:` key. This trips people up constantly.

## Build it

```powershell
docker build -t my-nodered:1.0 ./nodered
```

| Part | Meaning |
|---|---|
| `-t my-nodered:1.0` | Tag: name and version |
| `./nodered` | Build context — the directory sent to the builder |

Watch the output. Each instruction becomes a step, and each step becomes a
layer.

Confirm it exists:

```powershell
docker images | Select-String my-nodered
```

## Use it in Compose

Compose can build the image for you. Replace the `image:` line in the
`nodered` service:

```yaml
  nodered:
    build: ./nodered          # instead of: image: nodered/node-red:4.1.0-22
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    environment:
      TZ: ${TZ}
    volumes:
      - nodered_data:/data
    networks:
      - iiot-net
```

Then:

```powershell
docker compose up -d --build
```

`--build` forces a rebuild. Without it, Compose reuses whatever it built last
time and your Dockerfile edits are ignored — a frequent source of confusion.

Verify the payoff by wiping the volumes and starting fresh, then opening
<http://localhost:1880>. The InfluxDB nodes are in the palette on a completely
empty volume. Nobody installed anything.

## Layer caching

Docker caches each layer. On a rebuild it reuses every layer up to the first
instruction whose inputs changed — then rebuilds everything after it.

This makes instruction order a performance decision:

```dockerfile
# Slow: any source change re-runs npm install
COPY . /app
RUN npm install

# Fast: npm install only re-runs when package.json changes
COPY package.json package-lock.json /app/
RUN npm install
COPY . /app
```

Put what rarely changes first, and what changes constantly last.

## Keeping images small

```powershell
docker images
```

Compare `nodered/node-red` with `my-nodered`. The difference is your added
layer — small here, but the habits matter as images grow:

| Technique | Why |
|---|---|
| Pick a slim base (`-alpine`, `-slim`) | Often 10x smaller |
| Chain related `RUN` commands with `&&` | One layer instead of several |
| Clean package caches in the same `RUN` | A separate cleanup step does not shrink the earlier layer |
| Use `.dockerignore` | Keeps `node_modules` and `.git` out of the build context |

That third point surprises people: **deleting a file in a later layer does not
remove it from the image**, it only hides it. The bytes are still shipped.

Clean up inside the same `RUN` that created the mess, chained with `&&`, so the
cache never becomes a layer of its own.

## When to build vs when to configure

| Belongs in the image | Belongs in the volume or environment |
|---|---|
| Installed packages and nodes | Flows you edit |
| Application code | Dashboards |
| System dependencies | Passwords and tokens |
| Default configuration | Per-deployment settings |

The dividing line: **anything identical for every deployment goes in the image;
anything that differs stays outside it.** Bake a password into an image and you
have to rebuild to rotate it — and it stays in the image history forever.

## Exercise

Extend `nodered/Dockerfile` to also install `node-red-dashboard`, rebuild, and
confirm the dashboard nodes appear on a fresh volume.

<details>
<summary>Solution</summary>

```dockerfile
FROM nodered/node-red:4.1.0-22

RUN npm install --no-audit --no-fund --no-update-notifier \
    node-red-contrib-influxdb \
    node-red-dashboard
```

Then rebuild with `docker compose up -d --build`.

</details>

## Checkpoint

1. What is the difference between `RUN` and `CMD`?
2. Why does `EXPOSE 1880` not make the port reachable from your browser?
3. Why does deleting a file in a later layer fail to shrink the image?

---

**Previous:** [11 — Cleanup](11-cleanup.md) · **Next:** [13 — Bonus: Backup, Restore, and Moving to Another Machine](13-backup-migration.md)

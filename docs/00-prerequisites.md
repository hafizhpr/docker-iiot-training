# 00 — Prerequisites

## Install Docker

| OS | What to install |
|---|---|
| Windows 10/11 | [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/) — enable the WSL 2 backend when prompted |
| macOS | [Docker Desktop](https://docs.docker.com/desktop/install/mac-install/) — pick the Apple Silicon or Intel build to match your machine |
| Linux | [Docker Engine](https://docs.docker.com/engine/install/) plus the `docker-compose-plugin` package |

## Verify the installation

```powershell
docker --version
docker compose version
```

You should see two version numbers. If `docker compose version` fails but
`docker-compose --version` works, you have the old standalone tool: every
command in this course written as `docker compose` becomes `docker-compose`.

## Confirm the engine is actually running

A version number only proves the *client* is installed. This proves the engine
behind it is alive:

```powershell
docker run --rm hello-world
```

Expected: a short greeting, then the container exits. `--rm` deletes the
container the moment it finishes, so it leaves nothing behind.

If you instead see `Cannot connect to the Docker daemon`, Docker Desktop is not
running yet. Start it and wait for the whale icon to stop animating.

## Three words you need before Lesson 01

| Word | Meaning | Everyday analogy |
|---|---|---|
| **Image** | A read-only template containing an application and everything it needs | A recipe |
| **Container** | A running instance of an image | The meal cooked from that recipe |
| **Registry** | A server that stores and distributes images | The cookbook shelf (Docker Hub is the public one) |

You can cook the same recipe many times. You can run many containers from one
image — that is the whole idea.

## Disk space

The four images in this course occupy roughly **2.5 GB** on disk:

| Image | Size |
|---|---|
| `eclipse-mosquitto:2` | 36 MB |
| `influxdb:2.7` | 553 MB |
| `nodered/node-red:4.1.0-22` | 905 MB |
| `grafana/grafana:12.3.3` | 995 MB |

Check your own with `docker images` after Lesson 03. Make sure you have 8 GB
free; Docker also needs room for volumes and its build cache.

## Optional but recommended

- **Visual Studio Code** with the *Docker* extension — makes browsing
  containers, logs, and volumes far less painful than the CLI alone
- A terminal you are comfortable in — Windows users can use PowerShell,
  Git Bash, or a WSL shell; all three work

---

**Next:** [01 — Docker Fundamentals](01-docker-fundamentals.md)

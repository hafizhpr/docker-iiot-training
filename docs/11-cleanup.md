# 11 — Cleanup

Knowing how to remove things completely is what makes it safe to experiment.

## The three levels

```powershell
docker compose stop     # 1. stop containers, keep everything
docker compose down     # 2. remove containers + network, KEEP volumes
docker compose down -v  # 3. remove containers + network + VOLUMES
```

| Command | Containers | Network | Volumes | Images |
|---|---|---|---|---|
| `stop` | stopped | kept | kept | kept |
| `down` | removed | removed | **kept** | kept |
| `down -v` | removed | removed | **DELETED** | kept |

### `stop`

Frees CPU and memory, keeps everything else. `docker compose start` resumes
exactly where you were. Use it at the end of a training day.

### `down`

Containers and the network are removed. Your data, dashboards, and flows are
untouched because they live in volumes.

This is the normal way to shut down. `up -d` recreates the containers and
everything looks exactly as you left it.

### `down -v`

Deletes the volumes too. All measurements, every dashboard, every Node-RED
flow, and the InfluxDB setup itself are gone permanently.

**There is no undo. No confirmation prompt. No recycle bin.**

Use it deliberately: to reset the lab for a new group, or to force InfluxDB to
re-run `setup` after changing `.env`.

## Reclaiming disk space

Check what Docker is holding:

```powershell
docker system df
docker system df -v    # per-image and per-volume detail
```

Remove unused resources:

```powershell
docker container prune   # stopped containers
docker image prune       # dangling images
docker volume prune      # volumes not used by ANY container
docker network prune     # unused networks
```

The blunt instrument:

```powershell
docker system prune -a --volumes
```

That removes every stopped container, every unused image, every unused network,
**and every unused volume on the machine** — including work from projects that
have nothing to do with this course.

⚠️ `docker volume prune` counts a volume as unused if no container currently
exists for it. Run `docker compose down` first and your training data qualifies
as unused. The two commands are harmless alone and destructive in sequence.

Read the summary it prints before confirming. Every time.

## Remove only this course

```powershell
cd path/to/docker-iiot-training
docker compose down -v --rmi all
```

| Flag | Effect |
|---|---|
| `-v` | Delete this project's volumes |
| `--rmi all` | Delete the images this project used |

Verify nothing is left:

```powershell
docker ps -a | Select-String "mosquitto|influxdb|nodered|grafana"
docker volume ls | Select-String iiot-training
docker network ls | Select-String iiot-training
```

Three empty results means a clean uninstall. Compare that with removing four
natively-installed services, and the appeal of containers is hard to miss.

## Backing up before you delete

A named volume is not a folder you can copy. Extract it with a throwaway
container:

```powershell
docker run --rm  -v iiot-training_influxdb_data:/source:ro  -v "${PWD}:/backup"  alpine tar czf /backup/influxdb-backup.tar.gz -C /source .
```

Reading it, since this pattern is worth keeping:

| Part | Purpose |
|---|---|
| `--rm` | Delete the helper container when it finishes |
| `-v iiot-training_influxdb_data:/source:ro` | Mount the volume, read-only |
| `-v "${PWD}:/backup"` | Mount the current directory to write into. `${PWD}` is PowerShell's current path |
| `alpine tar czf ...` | A 5 MB image whose only job is to run `tar` |

Restore into a fresh volume:

```powershell
docker run --rm  -v iiot-training_influxdb_data:/target  -v "${PWD}:/backup"  alpine sh -c "cd /target && tar xzf /backup/influxdb-backup.tar.gz"
```

Stop the stack before backing up a database, or you capture a half-written
file.

## End-of-session checklist

| Situation | Command |
|---|---|
| Continuing tomorrow | `docker compose stop` |
| Done for the week, keep data | `docker compose down` |
| Resetting for a new group | `docker compose down -v` |
| Removing the course entirely | `docker compose down -v --rmi all` |

## Checkpoint

1. What is the exact difference between `down` and `down -v`?
2. Why is `docker volume prune` dangerous right after `docker compose down`?
3. Why can you not simply copy a named volume with a file manager?

---

**Previous:** [10 — Troubleshooting](10-troubleshooting.md) · **Next:** [12 — Bonus: Building Custom Images](12-building-custom-images.md)

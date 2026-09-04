# 13 — Bonus: Backup, Restore, and Moving to Another Machine

A stack that only runs on the laptop that built it is a demo. This lesson turns
it into something you can hand to somebody else.

Moving it is three separate jobs. People get burned because they only think
about one of them.

## Three things have to travel

| What | Where it lives | How it moves |
|---|---|---|
| Configuration | this repo folder — `docker-compose.yml`, `.env`, `mosquitto/config/`, `nodered/Dockerfile` | copy the folder, or `git clone` |
| Images | Docker's own store, roughly 2.5 GB on disk | `docker pull` on the new machine, or `docker save` / `docker load` if it has no internet |
| Data | six named volumes | `tar` through a throwaway container, as in Lesson 11 |

Copy the folder and nothing else and the new machine starts a perfectly healthy
stack — with no measurements, no flows, and no dashboards. That is the usual
mistake, and it looks like success right up until someone opens Grafana.

## Why the volume names still match

Line 10 of `docker-compose.yml`:

```yaml
name: iiot-training
```

That is what makes the volume `iiot-training_influxdb_data` on every machine
that runs this project. Without it, Compose takes the project name from the
folder name. Unzip the course into `Downloads\docker-course` on the new PC and
you get `docker-course_influxdb_data` — six brand new empty volumes, and a
restore that appears to work while the stack ignores it.

Pinning the project name is the one line that makes the rest of this lesson
possible.

## Step 1 — Stop the stack

```powershell
docker compose stop
```

InfluxDB is a database with open files. Archive it while it is running and you
capture a half-written state. `stop` is not `down` — nothing is deleted.

## Step 2 — Back up all six volumes

Lesson 11 backed up one volume. Here is the same command in a loop:

```powershell
mkdir backup
$volumes = "mosquitto_data","mosquitto_log","influxdb_data","influxdb_config","nodered_data","grafana_data"
foreach ($v in $volumes) {
  docker run --rm -v "iiot-training_${v}:/source:ro" -v "${PWD}/backup:/backup" alpine tar czf "/backup/$v.tar.gz" -C /source .
}
```

Check what you got:

```powershell
Get-ChildItem backup | Select-Object Name, Length
```

```
Name                     Length
----                     ------
grafana_data.tar.gz    20969470
influxdb_config.tar.gz      313
influxdb_data.tar.gz      44579
mosquitto_data.tar.gz       448
mosquitto_log.tar.gz       3371
nodered_data.tar.gz    16594864
```

Six files, none of them zero bytes. The sizes vary wildly and that is normal —
`influxdb_config` is a few hundred bytes of JSON, `grafana_data` carries an
entire SQLite database plus the plugin folder.

What each volume is holding for you:

| Volume | What you lose by skipping it |
|---|---|
| `mosquitto_data` | Retained messages |
| `mosquitto_log` | Broker log history — the one you can safely skip |
| `influxdb_data` | Every measurement you ever wrote |
| `influxdb_config` | The org, bucket, and token InfluxDB created during first-run setup |
| `nodered_data` | Every flow, plus the palette nodes you installed by hand |
| `grafana_data` | Every dashboard, the datasource, and the admin password |

⚠️ **`influxdb_data` and `influxdb_config` are a pair.** One is the database,
the other is the identity that unlocks it. Carry one without the other and
InfluxDB comes up a stranger to its own data.

## Step 3 — Move the images, if the new machine has no internet

Skip this step entirely when the target PC can reach the internet: `docker
compose up -d` will pull the four images itself.

A factory PC usually cannot. Pack them into a file:

```powershell
docker compose config --images
docker save -o iiot-images.tar eclipse-mosquitto:2 influxdb:2.7 nodered/node-red:4.1.0-22 grafana/grafana:12.3.3
```

`docker compose config --images` prints the exact four tags this project uses,
so you never have to guess. `docker save` preserves those tags, which is why
Lesson 01 insisted on pinning versions instead of `:latest` — an image saved as
`:latest` arrives on the new machine meaning something else entirely.

Check the file you produced:

```powershell
Get-Item iiot-images.tar | Select-Object Name, Length
```

About **580 MB** — noticeably less than the 2.5 GB those same four images
occupy according to `docker image ls`. Layers travel compressed inside the
archive and expand when Docker unpacks them, and layers shared between images
are stored once. `docker image ls` reports the unpacked size; plan your USB
drive around the archive, and the target machine's free space around the
2.5 GB.

## Step 4 — What you actually carry across

```
transfer/
├── docker-iiot-training/     the repo folder
├── backup/
│   ├── mosquitto_data.tar.gz
│   ├── mosquitto_log.tar.gz
│   ├── influxdb_data.tar.gz
│   ├── influxdb_config.tar.gz
│   ├── nodered_data.tar.gz
│   └── grafana_data.tar.gz
└── iiot-images.tar          only if the new machine is offline
```

USB drive, network share, however you like. Docker is not involved in this part.

## Step 5 — Restore on the new machine, in this order

The order is the whole lesson. Get it wrong and the symptoms are confusing.

**1. Put the repo folder wherever you want it.** The folder name does not
matter — `name: iiot-training` decides the volume names, not the folder.

**2. Load the images** — offline case only:

```powershell
docker load -i iiot-images.tar
```

**3. Restore the volumes.** They do not exist yet on this machine; the `-v`
flag creates each one on demand:

```powershell
$volumes = "mosquitto_data","mosquitto_log","influxdb_data","influxdb_config","nodered_data","grafana_data"
foreach ($v in $volumes) {
  docker run --rm -v "iiot-training_${v}:/target" -v "${PWD}/backup:/backup" alpine sh -c "cd /target && tar xzf /backup/$v.tar.gz"
}
```

**4. Only now start the stack:**

```powershell
docker compose up -d
```

Compose will print one warning per volume:

```
volume "iiot-training_grafana_data" already exists but was not created by
Docker Compose. Use `external: true` to use an existing volume
```

Six of those, and every one is expected. You created those volumes in step 3,
before Compose had a chance to. Nothing is wrong and nothing needs changing.

⚠️ **Do not run `up -d` before the restore.** `tar xzf` extracts *over*
whatever is already in the volume — it merges, it does not replace. Start the
stack first and InfluxDB runs its `setup` (Lesson 05), initialising both of its
volumes; restoring on top of that leaves two databases mixed into one
directory. If you have already made this mistake, `docker compose down -v` and
start Step 5 again from a clean slate.

## Step 6 — Verify the move actually worked

Four `Up` rows is not proof. The data is the point:

| Check | How | Pass |
|---|---|---|
| Containers | `docker compose ps` | four rows, all `Up` |
| Measurements | the Influx query below | rows carrying your old timestamps |
| Flows | http://localhost:1880 | your tabs, not an empty editor |
| Dashboards | http://localhost:3000 | your dashboard, datasource `working` |
| Retained MQTT | `docker compose exec mosquitto mosquitto_sub -t "factory/#" -C 1` | an old payload appears instantly |

```powershell
docker compose exec influxdb influx query --org training --token training-token-do-not-use-in-production 'from(bucket:\"sensors\") |> range(start:-30d) |> count()'
```

A 30-day window, not the 1-hour window used elsewhere in this course — you are
looking for readings made before the move.

The clearest tell of a failed restore: **if Grafana asks you to change the
admin password, `grafana_data` did not restore.** That prompt is Grafana's
first-run wizard, and a restored Grafana never shows it.

⚠️ **Log in to the restored Grafana with the password from the old machine,
not the one in `.env`.** `GF_SECURITY_ADMIN_PASSWORD` only seeds the admin user
the first time Grafana creates its database. After a restore that database
already exists, so the variable is ignored and the old password still rules —
the same trap as InfluxDB's `setup` in Lesson 05. A `401 Unauthorized` here is
evidence the restore *worked*: a fresh Grafana would have accepted `admin`.

## What does not travel

| Thing | Why |
|---|---|
| Container IP addresses | Assigned fresh on the new machine's bridge. This is exactly why Lesson 04 told you never to write one down |
| `host.docker.internal` targets | It resolves to the **new** host. A PLC reachable from the old laptop's network may be unreachable from this one |
| Published ports | If something on the new machine already owns `:3000`, Compose fails at start — Lesson 10 has the fix |
| Absolute bind mount paths | `./mosquitto/config` is relative, so it travels. `C:\Users\you\...` would not |

Volume contents are Linux-side no matter what your host is, so the tarballs
move between Windows and Linux machines without conversion.

## The production answer

Tarring a volume works for any container, which is why it is worth learning
once. For a database specifically the engine's own tool is better, because it
can run while the database is live:

```powershell
docker compose exec influxdb influx backup /tmp/backup --token training-token-do-not-use-in-production
```

Grafana exports dashboards as JSON, and Node-RED exports flows the same way.
Reach for those when the stack is not allowed to stop. Reach for `tar` when you
want one method that covers all six volumes and every container you will ever
meet.

## Checkpoint

1. You copy the folder to a new PC, run `docker compose up -d`, and everything
   starts cleanly — but every dashboard is empty. What did you forget?
2. Why must the volume restore happen *before* the first `up -d`, not after?
3. Which two volumes must always be restored together, and what breaks if you
   restore only one?
4. The new machine has no internet. Which two commands move the images onto it?

---

**Previous:** [12 — Bonus: Building Custom Images](12-building-custom-images.md) · **Back to:** [README](../README.md)

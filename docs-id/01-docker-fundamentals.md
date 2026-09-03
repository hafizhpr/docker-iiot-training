# 01 — Docker Fundamentals

## Masalah yang Diselesaikan Docker

Kamu menginstal sebuah aplikasi. Aplikasi itu membutuhkan Python 3.9, tapi aplikasi
lain di mesin yang sama membutuhkan Python 3.12. Kamu menginstal database, dan
file konfigurasinya tersebar di empat direktori yang tidak akan pernah kamu temukan
lagi. Kamu menyerahkan proyek ke rekan, dan tidak bisa berjalan, karena mesinnya
bukan mesinmu.

Docker mengemas sebuah aplikasi *beserta seluruh environment-nya* ke dalam satu
image. Image tersebut berjalan identik di laptopmu, laptop rekanmu, dan server
produksi.

## Image vs Container

Sebuah **image** adalah tumpukan layer read-only:

```
┌────────────────────────────┐
│ Layer 4: app configuration │
├────────────────────────────┤
│ Layer 3: application code  │   ← read-only
├────────────────────────────┤
│ Layer 2: Node.js runtime   │
├────────────────────────────┤
│ Layer 1: Debian base       │
└────────────────────────────┘
```

Menjalankan sebuah **container** menambahkan satu layer writable tipis di atasnya.
Semua yang ditulis container mendarat di layer itu — dan layer itu dihapus bersama
container. Itulah tepatnya alasan Pelajaran 05 tentang volume ada.

Layer juga dibagi bersama. Jika tiga image semuanya dibangun di atas `debian:bookworm`,
layer base tersebut diunduh dan disimpan **sekali saja**.

## Anatomi dari `docker run`

```powershell
docker run -d --name my-broker -p 1883:1883 eclipse-mosquitto:2
```

| Bagian | Arti |
|---|---|
| `-d` | Detached — jalankan di background |
| `--name my-broker` | Nama yang mudah dibaca, bukan nama acak |
| `-p 1883:1883` | Publish port host 1883 ke port container 1883 |
| `eclipse-mosquitto:2` | Image beserta tag-nya |

Coba:

```powershell
docker run -d --name my-broker -p 1883:1883 eclipse-mosquitto:2
docker ps
docker logs my-broker
docker stop my-broker
docker rm my-broker
```

## Perintah-Perintah Penting

| Perintah | Fungsinya |
|---|---|
| `docker ps` | Tampilkan container yang sedang berjalan |
| `docker ps -a` | Tampilkan semua container, termasuk yang sudah berhenti |
| `docker images` | Tampilkan image yang sudah diunduh |
| `docker logs <name>` | Tampilkan output sebuah container |
| `docker logs -f <name>` | Ikuti output secara langsung (Ctrl-C untuk berhenti) |
| `docker exec -it <name> sh` | Buka shell **di dalam** container yang sedang berjalan |
| `docker stop` / `start` / `rm` | Hentikan, jalankan, hapus sebuah container |

`docker exec` adalah yang paling perlu diingat. Ketika ada yang tidak beres,
kemampuan untuk masuk ke dalam container dan melihat-lihat mengubah tebak-tebakan
menjadi diagnosis.

## Mengapa Compose

Stack kita membutuhkan empat container, satu network, enam volume, dan sekitar
selusin environment variable. Sebagai perintah `docker run`, itu berarti empat
baris panjang yang rapuh yang tidak akan ada yang mau mengetiknya dua kali dengan
benar.

Compose memasukkan semuanya dalam satu file deklaratif:

```powershell
docker compose up -d     # buat dan jalankan semuanya
docker compose ps        # status container proyek ini
docker compose logs -f   # ikuti log dari semua service
docker compose down      # hentikan dan hapus container + network
```

Compose juga memberimu dua hal yang tidak dimiliki `docker run`:

1. **Network privat** yang dibuat otomatis, di mana container menemukan satu
   sama lain berdasarkan nama service
2. **Idempotency** — menjalankan `up` lagi hanya mengubah apa yang benar-benar berbeda

## Tag Itu Penting

```yaml
image: influxdb:2.7      # versi minor tertentu — dapat diprediksi
image: influxdb:latest   # apapun yang terbaru hari ini — target yang bergerak
```

`latest` bukan pointer "selalu terkini" yang ajaib; itu hanya nama tag, dan bisa
berubah tanpa sepengetahuanmu. Kursus ini mematok setiap image — `influxdb:2.7`,
`eclipse-mosquitto:2`, `nodered/node-red:4.1.0-22`, dan `grafana/grafana:12.3.3`
karena kursus yang rusak saat ada rilis upstream adalah kursus yang buruk.

## Checkpoint

Kamu seharusnya sudah bisa menjawab:

1. Di mana data yang ditulis container tersimpan, dan apa yang terjadi padanya saat `docker rm`?
2. Dalam `-p 8080:80`, nomor mana yang merupakan host dan mana yang merupakan container?
3. Apa yang diberikan `docker exec -it <name> sh` kepadamu?

---

**Sebelumnya:** [00 — Prasyarat](00-prerequisites.md) · **Selanjutnya:** [02 — The Stack](02-the-stack.md)

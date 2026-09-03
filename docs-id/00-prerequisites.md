# 00 — Prasyarat

## Instalasi Docker

| OS | Yang perlu diinstal |
|---|---|
| Windows 10/11 | [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/) — aktifkan WSL 2 backend saat diminta |
| macOS | [Docker Desktop](https://docs.docker.com/desktop/install/mac-install/) — pilih build Apple Silicon atau Intel sesuai mesinmu |
| Linux | [Docker Engine](https://docs.docker.com/engine/install/) beserta paket `docker-compose-plugin` |

## Verifikasi Instalasi

```powershell
docker --version
docker compose version
```

Kamu akan melihat dua nomor versi. Jika `docker compose version` gagal tapi
`docker-compose --version` berhasil, kamu masih menggunakan tool standalone yang
lama: setiap perintah dalam kursus ini yang ditulis sebagai `docker compose`
menjadi `docker-compose`.

## Konfirmasi Engine Sudah Berjalan

Nomor versi hanya membuktikan bahwa *client* sudah terinstal. Ini membuktikan bahwa
engine di baliknya sudah aktif:

```powershell
docker run --rm hello-world
```

Yang diharapkan: pesan singkat, kemudian container keluar. `--rm` menghapus
container begitu selesai, sehingga tidak meninggalkan apapun.

Jika kamu melihat `Cannot connect to the Docker daemon`, Docker Desktop belum
berjalan. Jalankan dan tunggu hingga ikon paus berhenti bergerak.

## Tiga Istilah Sebelum Pelajaran 01

| Istilah | Arti | Analogi Sehari-hari |
|---|---|---|
| **Image** | Template read-only yang berisi aplikasi dan semua yang dibutuhkannya | Sebuah resep |
| **Container** | Instance yang sedang berjalan dari sebuah image | Makanan yang dimasak dari resep itu |
| **Registry** | Server yang menyimpan dan mendistribusikan image | Rak buku resep (Docker Hub adalah yang publik) |

Kamu bisa memasak resep yang sama berkali-kali. Kamu bisa menjalankan banyak
container dari satu image — itulah inti dari konsep ini.

## Ruang Disk

Empat image dalam kursus ini menempati sekitar **2,5 GB** di disk:

| Image | Ukuran |
|---|---|
| `eclipse-mosquitto:2` | 36 MB |
| `influxdb:2.7` | 553 MB |
| `nodered/node-red:4.1.0-22` | 905 MB |
| `grafana/grafana:12.3.3` | 995 MB |

Periksa milikmu dengan `docker images` setelah Pelajaran 03. Pastikan kamu memiliki
8 GB ruang bebas; Docker juga memerlukan ruang untuk volume dan build cache-nya.

## Opsional tapi Direkomendasikan

- **Visual Studio Code** dengan ekstensi *Docker* — membuat penelusuran container,
  log, dan volume jauh lebih mudah dibanding hanya CLI
- Terminal yang kamu kuasai — pengguna Windows dapat menggunakan PowerShell,
  Git Bash, atau WSL shell; ketiganya berfungsi

---

**Selanjutnya:** [01 — Docker Fundamentals](01-docker-fundamentals.md)

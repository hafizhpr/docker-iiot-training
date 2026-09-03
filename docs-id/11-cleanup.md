# 11 — Cleanup

Mengetahui cara menghapus semuanya sepenuhnya adalah yang membuat eksperimen
menjadi aman.

## Tiga Tingkatan

```powershell
docker compose stop     # 1. hentikan container, pertahankan semuanya
docker compose down     # 2. hapus container + network, PERTAHANKAN volume
docker compose down -v  # 3. hapus container + network + VOLUME
```

| Perintah | Container | Network | Volume | Image |
|---|---|---|---|---|
| `stop` | dihentikan | dipertahankan | dipertahankan | dipertahankan |
| `down` | dihapus | dihapus | **dipertahankan** | dipertahankan |
| `down -v` | dihapus | dihapus | **DIHAPUS** | dipertahankan |

### `stop`

Membebaskan CPU dan memori, mempertahankan segalanya. `docker compose start`
melanjutkan tepat di mana kamu tinggalkan. Gunakan di akhir hari pelatihan.

### `down`

Container dan network dihapus. Data, dashboard, dan flow-mu tidak berubah karena
mereka berada di volume.

Ini adalah cara normal untuk mematikan. `up -d` membuat ulang container dan
semuanya terlihat persis seperti yang kamu tinggalkan.

### `down -v`

Juga menghapus volume. Semua pengukuran, setiap dashboard, setiap flow Node-RED,
dan setup InfluxDB itu sendiri hilang selamanya.

**Tidak ada undo. Tidak ada konfirmasi prompt. Tidak ada recycle bin.**

Gunakan dengan sengaja: untuk mereset lab bagi kelompok baru, atau untuk memaksa
InfluxDB menjalankan ulang `setup` setelah mengubah `.env`.

## Mengklaim Kembali Ruang Disk

Periksa apa yang ditahan Docker:

```powershell
docker system df
docker system df -v    # detail per-image dan per-volume
```

Hapus resource yang tidak digunakan:

```powershell
docker container prune   # container yang berhenti
docker image prune       # dangling image
docker volume prune      # hanya volume ANONIM yang tidak digunakan
docker volume prune -a   # volume yang tidak digunakan termasuk yang BERNAMA
docker network prune     # network yang tidak digunakan
```

Instrumen tumpul:

```powershell
docker system prune -a --volumes
```

Itu menghapus setiap container yang berhenti, setiap image yang tidak digunakan,
setiap network yang tidak digunakan, dan setiap volume **anonim** yang tidak
digunakan di mesin — termasuk pekerjaan dari proyek yang tidak ada hubungannya
dengan kursus ini.

⚠️ **Named volume memerlukan `-a`.** Docker modern memperlakukan "volume yang
tidak digunakan" sebagai *anonim* secara default, sehingga `docker volume prune`
biasa dan `docker system prune --volumes` membiarkan `iiot-training_influxdb_data`
tidak tersentuh. Menambahkan `-a` adalah yang menjangkau named volume. Periksa
build-mu sendiri:

```powershell
docker volume prune --help    # cari: -a, --all
```

⚠️ **Volume dihitung sebagai tidak digunakan ketika tidak ada container untuknya**
— bukan ketika tidak ada container yang berjalan. Jadi `docker compose down`
diikuti dengan `docker volume prune -a` menghapus data pelatihanmu. Salah satu
perintah itu tidak berbahaya sendirian; urutannya tidak aman.

Baca ringkasan yang dicetaknya sebelum mengkonfirmasi. Setiap saat.

## Hapus Hanya Kursus Ini

```powershell
cd path/to/docker-iiot-training
docker compose down -v --rmi all
```

| Flag | Efek |
|---|---|
| `-v` | Hapus volume proyek ini |
| `--rmi all` | Hapus image yang digunakan proyek ini |

Verifikasi tidak ada yang tersisa:

```powershell
docker ps -a | Select-String "mosquitto|influxdb|nodered|grafana"
docker volume ls | Select-String iiot-training
docker network ls | Select-String iiot-training
```

Tiga hasil kosong berarti uninstall bersih. Bandingkan dengan menghapus empat
service yang terinstal secara native, dan daya tarik container sulit diabaikan.

## Backup Sebelum Menghapus

Named volume bukan folder yang bisa kamu salin. Ekstrak dengan container
sementara:

```powershell
docker run --rm  -v iiot-training_influxdb_data:/source:ro  -v "${PWD}:/backup"  alpine tar czf /backup/influxdb-backup.tar.gz -C /source .
```

Membacanya, karena pola ini layak diingat:

| Bagian | Tujuan |
|---|---|
| `--rm` | Hapus container helper setelah selesai |
| `-v iiot-training_influxdb_data:/source:ro` | Mount volume, read-only |
| `-v "${PWD}:/backup"` | Mount direktori saat ini untuk menulis. `${PWD}` adalah path saat ini di PowerShell |
| `alpine tar czf ...` | Image 5 MB yang tugasnya hanya menjalankan `tar` |

Restore ke volume baru:

```powershell
docker run --rm  -v iiot-training_influxdb_data:/target  -v "${PWD}:/backup"  alpine sh -c "cd /target && tar xzf /backup/influxdb-backup.tar.gz"
```

Hentikan stack sebelum mem-backup database, atau kamu menangkap file yang setengah
tertulis.

## Checklist Akhir Sesi

| Situasi | Perintah |
|---|---|
| Melanjutkan besok | `docker compose stop` |
| Selesai untuk minggu ini, pertahankan data | `docker compose down` |
| Reset untuk kelompok baru | `docker compose down -v` |
| Menghapus kursus sepenuhnya | `docker compose down -v --rmi all` |

## Checkpoint

1. Apa perbedaan tepat antara `down` dan `down -v`?
2. Mengapa `docker volume prune -a` berbahaya tepat setelah `docker compose down`,
   dan mengapa perintah yang sama tanpa `-a` membiarkan datamu tidak tersentuh?
3. Mengapa kamu tidak bisa begitu saja menyalin named volume dengan file manager?

---

**Sebelumnya:** [10 — Troubleshooting](10-troubleshooting.md) · **Selanjutnya:** [12 — Bonus: Building Custom Images](12-building-custom-images.md)

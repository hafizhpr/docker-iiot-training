# 05 — Volumes dan Persistensi

## Mengapa Container Lupa

Writable layer sebuah container dihapus ketika container dihapus. Tanpa volume,
setiap `docker compose down` akan menghapus database, flow Node-RED, dan
dashboard Grafana-mu.

Itu bukan bug. Container memang dirancang untuk bisa dibuang — yang hanya
berfungsi jika data yang berharga berada di tempat lain.

## Dua Cara Menyimpan Data

### Named Volume

```yaml
volumes:
  - influxdb_data:/var/lib/influxdb2
```

Docker membuat dan mengelola penyimpanan. Kamu tidak memilih lokasi di disk, dan
kamu hanya berinteraksi dengannya melalui perintah Docker.

**Gunakan untuk:** database, state aplikasi, apapun yang dimiliki container.

### Bind Mount

```yaml
volumes:
  - ./mosquitto/config:/mosquitto/config
```

Sebuah direktori di **mesinmu** dipetakan ke dalam container. Edit file di editor
biasamu dan container langsung melihat perubahannya.

**Gunakan untuk:** konfigurasi yang ingin kamu edit, source code selama development.

### Memilih di Antara Keduanya

| | Named Volume | Bind Mount |
|---|---|---|
| Dikelola oleh | Docker | Kamu |
| Lokasi | Penyimpanan internal Docker | Path yang kamu pilih |
| Bisa diedit dari host | Rumit | Mudah |
| Masalah permission | Jarang | Umum |
| Portabel antar mesin | Ya | Tergantung path-nya |

Stack kita menggunakan keduanya dengan sengaja, agar kamu bisa merasakan
perbedaannya.

## Apa yang Dipersistensikan Stack Kita

| Service | Volume | Isi |
|---|---|---|
| Mosquitto | `mosquitto_data` | Retained message, pesan antrian QoS 1/2 |
| Mosquitto | `mosquitto_log` | File log broker |
| Mosquitto | `./mosquitto/config` *(bind)* | `mosquitto.conf` |
| InfluxDB | `influxdb_data` | Semua data pengukuran |
| InfluxDB | `influxdb_config` | Setup org, bucket, dan token |
| Node-RED | `nodered_data` | Flow, kredensial, palette node yang terinstal |
| Grafana | `grafana_data` | Dashboard, pengguna, datasource |

Setiap named volume juga harus dideklarasikan di bagian bawah file:

```yaml
volumes:
  mosquitto_data:
  mosquitto_log:
  influxdb_data:
  influxdb_config:
  nodered_data:
  grafana_data:
```

Lupa satu saja dan Compose menolak start dengan pesan `service refers to undefined
volume`.

## Buktikan Persistensi

Tulis sebuah data point:

```powershell
docker compose exec influxdb influx write  --bucket sensors --org training --token training-token-do-not-use-in-production  --precision s "temperature,line=line1 value=42.5"
```

Hapus setiap container:

```powershell
docker compose down
docker compose ps -a
```

Tidak ada yang tersisa. Sekarang kembalikan dan cari datanya:

```powershell
docker compose up -d
Start-Sleep 10
docker compose exec influxdb influx query --org training --token training-token-do-not-use-in-production 'from(bucket:\"sensors\") |> range(start:-1h) |> filter(fn:(r) => r._measurement == \"temperature\")'
```

> **Mengapa escape `\"`?** Flux membutuhkan tanda kutip ganda di sekitar string
> literal, tapi Windows PowerShell 5.1 menghapus tanda kutip ganda yang disematkan
> saat meneruskan argumen ke program native. Tanpa backslash, InfluxDB menerima
> `bucket: sensors` dan menjawab `undefined identifier sensors`.
> Meng-escape-nya adalah bentuk yang reliabel di PowerShell.

Datanya masih ada. Container dihapus dan dibuat ulang; volume tidak tersentuh.

## `down` vs `down -v`

```powershell
docker compose down       # menghapus container + network. VOLUME TETAP ADA.
docker compose down -v    # menghapus container + network + VOLUME. Data hilang.
```

Jalankan eksperimen persistensi lagi dengan `down -v` dan query akan kembali
kosong — dan InfluxDB menjalankan ulang seluruh setup-nya, karena `influxdb_config`
juga sudah dihapus.

`-v` adalah flag yang bisa mengakhiri karir. Tidak ada undo.

## Memeriksa Volume

```powershell
docker volume ls
docker volume inspect iiot-training_influxdb_data
docker system df -v
```

Nama volume diberi prefix dengan nama proyek Compose, persis seperti network.

`docker system df -v` menampilkan berapa banyak disk yang dikonsumsi setiap volume —
berguna setelah database time-series sudah mengumpulkan data cukup lama.

## Konfigurasi Berbasis Environment

Volume mempersistensikan data; environment variable mengkonfigurasi perilaku.
Service InfluxDB kita menginisialisasi dirinya sepenuhnya dari `.env`:

```yaml
environment:
  DOCKER_INFLUXDB_INIT_MODE: setup
  DOCKER_INFLUXDB_INIT_USERNAME: ${INFLUXDB_USERNAME}
  DOCKER_INFLUXDB_INIT_PASSWORD: ${INFLUXDB_PASSWORD}
  DOCKER_INFLUXDB_INIT_ORG: ${INFLUXDB_ORG}
  DOCKER_INFLUXDB_INIT_BUCKET: ${INFLUXDB_BUCKET}
  DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: ${INFLUXDB_TOKEN}
```

Compose membaca `.env` dari direktori yang sama secara otomatis.

Mode `setup` hanya berjalan ketika `influxdb_config` kosong. Pada start kedua,
volume sudah berisi konfigurasi, sehingga variabel-variabel itu diabaikan — itulah
mengapa mengubah password di `.env` tampaknya tidak berpengaruh sampai kamu
melakukan `down -v`.

Konfirmasi apa yang sebenarnya di-resolve Compose sebelum memulai apapun:

```powershell
docker compose config
```

## Checkpoint

1. Volume type mana yang akan kamu gunakan untuk direktori data Postgres, dan mana
   untuk `nginx.conf` yang ingin kamu ubah?
2. Apa perbedaan antara `down` dan `down -v`?
3. Kamu mengubah `INFLUXDB_PASSWORD` di `.env` dan restart. Tidak ada yang terjadi. Mengapa?

---

**Sebelumnya:** [04 — Networking](04-networking.md) · **Selanjutnya:** [06 — Lab: MQTT](06-lab-mqtt.md)

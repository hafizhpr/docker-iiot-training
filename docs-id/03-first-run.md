# 03 — First Run

Kamu sudah menginstal Docker dan memahami apa yang dilakukan stack ini. Sekarang
kamu menjalankannya — dan, yang lebih penting, belajar cara memastikan apakah
benar-benar berhasil.

## Baca File Sebelum Menjalankannya

Buka `docker-compose.yml`. Terlihat panjang, tapi sebenarnya pola pendek yang
sama diulang empat kali.

### Struktur Luar

```yaml
name: iiot-training      # nama proyek — menjadi prefix volume dan network

services:                # container-container
  mosquitto: ...
  influxdb: ...
  nodered: ...
  grafana: ...

volumes:                 # setiap named volume harus dideklarasikan di sini
  mosquitto_data:
  ...

networks:                # network privat yang mereka bagikan
  iiot-net:
    driver: bridge
```

Tiga blok tingkat atas. Selebihnya hanya detail.

### Satu Service, Key demi Key

```yaml
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    restart: unless-stopped
    ports:
      - "8086:8086"
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: ${INFLUXDB_USERNAME}
    volumes:
      - influxdb_data:/var/lib/influxdb2
    networks:
      - iiot-net
```

| Key | Fungsinya |
|---|---|
| `image` | Image mana yang akan dijalankan. Dipatok ke `2.7`, bukan `latest` |
| `container_name` | Nama tetap yang mudah dibaca, bukan nama acak |
| `restart: unless-stopped` | Restart otomatis kecuali kamu yang menghentikannya |
| `ports` | `host:container` — mempublikasikan port ke mesinmu |
| `environment` | Konfigurasi yang diteruskan saat startup; `${...}` dibaca dari `.env` |
| `volumes` | `volume_name:/path/in/container` — apa yang bertahan setelah container dihapus |
| `networks` | Network mana yang diikuti, agar DNS berdasarkan nama service berfungsi |

Dua key lagi muncul di tempat lain dalam file:

| Key | Di mana | Fungsinya |
|---|---|---|
| `depends_on` | `nodered`, `grafana` | Urutan start saja — **bukan** kesiapan service |
| `extra_hosts` | `nodered` | Menambahkan `host.docker.internal` agar flow bisa menjangkau mesin host — Pelajaran 04 |

Itulah seluruh kosakata file compose ini. Empat service, key yang sama.

## Periksa Sebelum Mulai

```powershell
docker compose config
```

Perintah ini mencetak file dengan setiap `${VARIABLE}` yang sudah disubstitusi,
dan gagal dengan jelas jika ada error sintaks. Dua hal yang perlu dicek:

- Password dan token menampilkan nilai nyata, bukan string kosong. Kosong berarti
  `.env` tidak ditemukan — biasanya karena kamu berada di direktori yang salah
- Tidak ada peringatan `variable is not set`

Membiasakan diri menjalankan ini terlebih dahulu akan menghemat sesi debug yang
membingungkan di kemudian hari.

## Menjalankan Stack

```powershell
docker compose up -d
```

`-d` adalah detached — mengembalikan promptmu alih-alih streaming log selamanya.

**Saat pertama kali dijalankan, sekitar 2,5 GB image akan diunduh** dan membutuhkan
beberapa menit pada koneksi normal. Unduhan sebenarnya lebih kecil — layer berjalan
terkompresi dan mengembang di disk — jadi perhatikan prosesnya, bukan jamnya.
Kamu akan melihat progress layer demi layer:

```
mosquitto Pulling
 6e771e15690e Pull complete
 ...
mosquitto Pulled
influxdb Pulling
```

Setiap start berikutnya melewati semua itu, karena image sudah ada di disk.

Kemudian urutan pembuatan:

```
 Volume iiot-training_influxdb_data  Created
 Network iiot-training_iiot-net      Created
 Container mosquitto                 Created
 Container influxdb                  Created
 Container nodered                   Created
 Container grafana                   Created
 Container mosquitto                 Started
 ...
```

Perhatikan apa yang terjadi pertama. Volume dan network dibuat **sebelum** container
manapun, karena container menempel pada mereka.

## Verifikasi

```powershell
docker compose ps
```

```
NAME        STATUS          PORTS
grafana     Up 6 seconds    0.0.0.0:3000->3000/tcp
influxdb    Up 6 seconds    0.0.0.0:8086->8086/tcp
mosquitto   Up 6 seconds    0.0.0.0:1883->1883/tcp, 0.0.0.0:9001->9001/tcp
nodered     Up 6 seconds    0.0.0.0:1880->1880/tcp
```

Empat baris, semua `Up`. Apapun yang menampilkan `Restarting` atau `Exited` berarti
baca log sekarang:

```powershell
docker compose logs <service-name>
```

Pelajaran 10 membahas seperti apa kegagalan umum yang terjadi.

## Buka Setiap Service

| Service | URL | Login |
|---|---|---|
| Node-RED | <http://localhost:1880> | tidak ada |
| InfluxDB | <http://localhost:8086> | `admin` / `admin12345` |
| Grafana | <http://localhost:3000> | `admin` / `admin` |

Buka ketiganya sekarang. Jika halaman terbuka, service tersebut benar-benar
berfungsi — container berjalan, port dipublikasikan, aplikasi mendengarkan. Tiga
tanda centang hijau lebih bernilai daripada membaca log sebanyak apapun.

Mosquitto tidak memiliki web UI. Uji dari command line:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'test' -m 'hello' -r
docker compose exec mosquitto mosquitto_sub -h localhost -t 'test' -C 1 -W 5
```

`hello` muncul kembali. Broker berfungsi.

Publishing dengan `-r` membuat broker menyimpan pesan (retain), sehingga subscriber
menerimanya meskipun baru terhubung setelahnya. Itulah yang memungkinkanmu menguji
broker dari satu jendela PowerShell alih-alih mengelola dua.

### Mengapa InfluxDB Tidak Meminta Setup Apapun

Biasanya InfluxDB menyambutmu dengan wizard setup. Kita melewatinya, karena
`DOCKER_INFLUXDB_INIT_MODE: setup` beserta variabel dari `.env` menjalankan
seluruh inisialisasi saat pertama kali boot — organisasi, bucket, admin user, dan
API token.

Ini patut diperhatikan sebagai pola, bukan sekadar kemudahan: container yang
mengkonfigurasi dirinya sendiri dari environment dapat dibuat dan dihancurkan oleh
skrip, tanpa manusia yang harus mengklik wizard. Itulah perbedaan antara mesin
yang kamu setup dan mesin yang bisa kamu recreate.

## Apa yang Sebenarnya Dibuat

```powershell
docker compose ps       # 4 container
docker volume ls | Select-String iiot-training    # 6 volume
docker network ls | Select-String iiot-training   # 1 network
```

Semuanya membawa prefix `iiot-training` — `name:` di bagian atas file compose.
Prefix itulah yang mencegah proyek ini bertabrakan dengan proyek Docker lain di
mesinmu.

## Menghentikan

```powershell
docker compose stop     # jeda; jalankan lagi dengan: docker compose start
docker compose down     # hapus container + network, pertahankan semua data
```

Keduanya aman. Pelajaran 11 menjelaskan yang ketiga — dan mengapa itu tidak aman.

## Checkpoint

1. Apa yang ditampilkan `docker compose config` yang tidak kamu lihat dengan membaca file langsung?
2. Sebuah service menampilkan `Restarting (1)`. Apa perintahmu berikutnya?
3. Mengapa volume dan network dibuat sebelum container manapun?
4. Mengapa InfluxDB tidak pernah menampilkan wizard setup?

---

**Sebelumnya:** [02 — The Stack](02-the-stack.md) · **Selanjutnya:** [04 — Networking](04-networking.md)

# 10 — Troubleshooting

## Urutan Diagnosis

Ikuti daftar ini dari atas. Hampir setiap masalah teridentifikasi dalam tiga
perintah pertama.

```powershell
docker compose ps          # 1. apakah sedang berjalan sama sekali?
docker compose logs <svc>  # 2. apa yang dikatakannya sebelum rusak?
docker compose exec <svc> sh   # 3. masuk ke dalam dan lihat
```

Menebak lebih lambat daripada melihat. Lihat langsung.

## Membaca Kolom Status

| Status | Arti |
|---|---|
| `Up 5 minutes` | Sehat dan berjalan |
| `Restarting (1)` | Crash saat startup, dalam loop — periksa log segera |
| `Exited (0)` | Berhenti dengan bersih |
| `Exited (1)` | Berhenti dengan error |
| `Exited (143)` | Dibunuh oleh SIGTERM — hasil normal dari `docker compose stop` |
| `Created` | Tidak pernah start |

`Restarting` adalah yang paling mencolok. `restart: unless-stopped` setia me-restart
container yang mati setiap saat.

### Tidak Setiap Exit Code Bukan Nol Adalah Kesalahan

`docker compose stop` mengirimkan SIGTERM. Container yang shutdown pada sinyal
tersebut keluar dengan `0`; yang tidak menanganinya dibunuh olehnya dan keluar
dengan `143` (`128 + 15`, nomor sinyal). Keduanya normal. Stack kita sendiri
beragam:

```
grafana     Exited (0)
influxdb    Exited (2)
mosquitto   Exited (0)
nodered     Exited (143)
```

Tidak ada yang rusak di sana — itu adalah empat aplikasi dengan empat pendapat
berbeda tentang SIGTERM.

Kode hanya penting ketika kamu **tidak** meminta container untuk berhenti. Sebuah
`Exited (1)` atau `Exited (2)` tepat setelah `up -d`, atau dipasangkan dengan
`Restarting`, adalah kegagalan nyata. Kode yang sama setelah `stop`-mu sendiri
bukan. Tanyakan apa yang kamu lakukan terakhir sebelum membaca angkanya.

---

## Masalah: InfluxDB Keluar Segera

**Log:**

```
Error: failed to setup instance: 400 Bad Request:
passwords must be between 8 and 72 characters long
```

**Penyebab:** `DOCKER_INFLUXDB_INIT_PASSWORD` kurang dari 8 karakter. Pilihan
yang jelas yaitu `admin` tepat 5 karakter.

**Solusi:** di `.env`, gunakan `INFLUXDB_PASSWORD=admin12345`.

Ini adalah satu tempat dalam kursus ini di mana kita tidak bisa menghormati aturan
"admin / admin di mana-mana". Image menegakkannya dan tidak ada flag untuk
mematikannya — pengingat bagus bahwa validasi image sendiri mengungguli file
compose-mu.

---

## Masalah: Client MQTT Tidak Bisa Terhubung

**Gejala:** container dalam status `Up`, port 1883 dipublikasikan, dan setiap
client ditolak.

**Penyebab:** Mosquitto 2.x hanya bind ke loopback di dalam container kecuali
ada listener yang dideklarasikan.

**Solusi:** konfirmasi bahwa konfigurasi benar-benar sampai ke container:

```powershell
docker compose exec mosquitto cat /mosquitto/config/mosquitto.conf
```

Kamu harus melihat `listener 1883`. Jika file kosong atau hilang, path bind mount
di `docker-compose.yml` salah.

---

## Masalah: "Connection refused" Antara Container

**Penyebab:** kamu menggunakan `localhost` dalam alamat container-ke-container.
Ini adalah error yang paling sering terjadi dalam seluruh kursus.

**Solusi:**

| Salah | Benar |
|---|---|
| `mqtt://localhost:1883` | `mqtt://mosquitto:1883` |
| `http://localhost:8086` | `http://influxdb:8086` |

**Verifikasi jalur secara langsung:**

```powershell
docker compose exec nodered wget -qO- http://influxdb:8086/health
```

JSON kembali berarti network baik-baik saja dan masalahmu ada di kredensial.
Ditolak berarti kamu menggunakan alamat yang salah atau target tidak aktif.

**Periksa apakah DNS ter-resolve sama sekali:**

```powershell
docker compose exec nodered getent hosts influxdb
```

---

## Masalah: Port Sudah Dialokasikan

**Log:**

```
Error starting userland proxy: listen tcp 0.0.0.0:3000: bind: address already in use
```

**Penyebab:** sesuatu yang lain di mesinmu memiliki port itu. Port 3000 diperebutkan
oleh React, Next.js, dan Rails.

**Cari pelakunya:**

```powershell
Get-NetTCPConnection -LocalPort 3000 -State Listen | Select-Object LocalPort, OwningProcess
Get-Process -Id (Get-NetTCPConnection -LocalPort 3000 -State Listen).OwningProcess
```

Baris pertama memberi tahu kamu port sudah digunakan dan process id mana yang
memilikinya; yang kedua mengubah id itu menjadi nama yang kamu kenali.

**Solusi — pindahkan hanya sisi host:**

```yaml
grafana:
  ports:
    - "3001:3000"     # browser menggunakan 3001; Grafana tetap serving di 3000
```

Kemudian `docker compose up -d`. Tidak ada yang berubah di dalam container, dan
tidak ada container lain yang perlu diperbarui — mereka menjangkau Grafana dengan
nama service, bukan host port.

---

## Masalah: Perubahan Konfigurasi Tidak Berpengaruh (Windows)

**Penyebab:** editormu menyimpan `mosquitto.conf` dengan line ending CRLF dan
parser tersedak, atau file tidak pernah di-mount.

**Periksa apa yang dilihat container:**

```powershell
docker compose exec mosquitto cat -A /mosquitto/config/mosquitto.conf | Select-Object -First 5
```

`^M$` di akhir baris berarti CRLF.

**Solusi:** simpan file sebagai LF di editormu (VS Code menampilkan `CRLF` / `LF`
di status bar — klik untuk menggantinya). `.gitattributes` dalam repositori ini
memaksakan LF untuk file `.conf` saat checkout.

---

## Masalah: Mengubah `.env` Tidak Berpengaruh

**Penyebab:** mode `setup` InfluxDB hanya berjalan ketika `influxdb_config` kosong.
Di setiap start berikutnya, volume sudah berisi konfigurasi dan variabel-variabel
diabaikan.

**Solusi — hanya jika kamu bersedia kehilangan data:**

```powershell
docker compose down -v
docker compose up -d
```

**Periksa apa yang di-resolve Compose sebelum start:**

```powershell
docker compose config
```

Ini mencetak file yang sudah disubstitusi sepenuhnya. Jika sebuah variabel tampil
kosong, `.env` tidak dibaca — biasanya karena kamu menjalankan perintah dari
direktori yang berbeda.

---

## Masalah: Palette Node Node-RED Menghilang

**Penyebab:** `docker compose down -v` menghapus `nodered_data`, yang berisi
`/data/node_modules`.

**Solusi:** instal ulang melalui Manage palette — atau hentikan masalah berulang
dengan memasukkan node ke dalam custom image ([Pelajaran 12](12-building-custom-images.md)).

---

## Masalah: Service Refers to Undefined Volume

**Log:**

```
service "influxdb" refers to undefined volume influxdb_data
```

**Penyebab:** volume digunakan dalam service tapi tidak dideklarasikan di blok
`volumes:` tingkat atas.

**Solusi:** tambahkan:

```yaml
volumes:
  influxdb_data:
```

---

## Masalah: Service Start Sebelum Dependensinya Siap

**Penyebab:** `depends_on` mengatur urutan startup; ia tidak menunggu kesiapan.

**Solusi:** tambahkan healthcheck dan bergantung padanya:

```yaml
influxdb:
  healthcheck:
    test: ["CMD", "influx", "ping"]
    interval: 10s
    timeout: 5s
    retries: 5

nodered:
  depends_on:
    influxdb:
      condition: service_healthy
```

Kita menghilangkan ini dari stack kursus karena Node-RED reconnect dengan sendirinya.
Tambahkan ini ketika sebuah service crash alih-alih retry.

---

## Masalah: Instalasi Palette Gagal dengan EBADENGINE

**Log (dari Manage palette, atau `docker compose logs nodered`):**

```
npm error code EBADENGINE
npm error engine Unsupported engine
npm error notsup Not compatible with your version of node/npm: node-red-contrib-modbus@5.60.2
npm error notsup Required: {"node":">=22"}
npm error notsup Actual:   {"npm":"10.9.8","node":"v20.20.2"}
```

**Penyebab:** palette node memerlukan runtime Node.js yang lebih baru dari yang
ada di dalam image Node-RED-mu. Tidak ada yang salah dengan node atau npm — image-nya
hanya lebih lama dari yang diharapkan package.

Baca dua baris terakhir dari error. Keduanya memberi tahu kamu persis apa yang
diperlukan dan apa yang kamu miliki.

**Periksa apa yang sedang kamu jalankan:**

```powershell
docker compose exec nodered node --version
```

**Solusi:** beralih ke varian image yang dibangun di atas Node major yang diperlukan.
Node-RED mempublikasikan tag dengan versi Node sebagai suffix:

| Tag | Node runtime |
|---|---|
| `nodered/node-red:latest` | Node 20 |
| `nodered/node-red:4.1.0-22` | Node 22 |

Di `docker-compose.yml`:

```yaml
nodered:
  image: nodered/node-red:4.1.0-22
```

Kemudian recreate hanya service itu:

```powershell
docker compose up -d nodered
docker compose exec nodered node --version
```

Flow dan palette node yang sebelumnya terinstal tidak berubah — mereka berada di
volume `nodered_data`, dan hanya image yang berubah. Inilah hasil dari Pelajaran
05: runtime bisa dibuang, datanya tidak.

Coba lagi instalasi dari Manage palette.

> **Mengapa ini adalah argumen untuk mematok versi.** `latest` berpindah ke Node 20
> pada suatu titik tanpa memberi tahu siapapun, dan akan berpindah lagi. Tag yang
> dipatok seperti `4.1.0-22` menyatakan runtime yang kamu uji, sehingga package
> yang berhasil diinstal kemarin masih bisa diinstal besok.

> **Native module.** Sebagian besar palette node adalah pure JavaScript dan bertahan
> setelah perubahan versi Node. Node dengan binding yang dikompilasi mungkin perlu
> diinstal ulang setelah pergantian — uninstall dan instal ulang dari Manage
> palette jika bermasalah.

---

## Masalah: Query Flux Mengatakan "undefined identifier"

**Log:**

```
Error: failed to execute query: 400 Bad Request:
error @1:13-1:20: undefined identifier sensors
```

**Penyebab:** Windows PowerShell 5.1 menghapus tanda kutip ganda saat meneruskan
argumen ke program native. Flux membutuhkannya di sekitar setiap string literal,
sehingga InfluxDB menerima `from(bucket: sensors)` dan memperlakukan `sensors`
sebagai nama variabel yang tidak pernah didefinisikan.

**Solusi:** escape tanda kutip dalam dengan backslash.

| | |
|---|---|
| Gagal | `'from(bucket:"sensors")'` |
| Berhasil | `'from(bucket:\"sensors\")'` |

Flux tidak memiliki bentuk string single-quoted, sehingga menukar gaya kutip
tidak membantu — `from(bucket:'sensors')` adalah syntax error.

Ini hanya mempengaruhi query yang diketik di prompt PowerShell. Query yang
dimasukkan di panel editor Grafana atau UI InfluxDB tidak memerlukan escaping.

---

## Perintah yang Layak Dihafalkan

```powershell
docker compose ps                      # status
docker compose logs -f --tail=50 grafana   # ikuti log terbaru
docker compose exec influxdb sh        # shell di dalam
docker compose config                  # konfigurasi yang sudah di-resolve
docker compose restart mosquitto       # restart satu service
docker compose up -d --force-recreate nodered   # rebuild satu container
docker stats                           # CPU dan memori langsung per container
docker network inspect iiot-training_iiot-net   # siapa yang ada di network
```

---

## Membuat Stack Ini Aman untuk Penggunaan Nyata

Setiap shortcut dalam kursus ini, dan apa yang menggantinya:

| Shortcut | Pengganti untuk produksi |
|---|---|
| `allow_anonymous true` | `password_file` beserta kredensial per-perangkat, dan TLS di port 8883 |
| Token InfluxDB yang hardcoded | Token yang dibuat dengan scope least-privilege, disuntikkan sebagai secret |
| Grafana `admin` / `admin` | Password yang kuat, atau SSO / LDAP |
| `.env` yang di-commit ke git | Tambahkan ke `.gitignore`; gunakan secret manager |
| Tag `latest` | Versi yang dipatok, di-upgrade secara sengaja |
| Port yang dipublikasikan ke `0.0.0.0` | Bind ke `127.0.0.1`, atau letakkan reverse proxy di depannya |

Tidak ada yang sulit. Semuanya dilewati di sini karena satu alasan: pelajar yang
menghabiskan jam pertamanya pada sertifikat TLS tidak pernah sampai ke pelajaran
Docker.

## Checkpoint

1. Sebuah container menampilkan `Restarting (1)`. Apa perintah pertamamu?
2. Grafana tidak bisa menjangkau InfluxDB. Perintah tunggal apa yang membedakan
   masalah network dari masalah kredensial?
3. Kamu mengubah nama bucket InfluxDB di `.env` dan tidak ada yang terjadi. Mengapa?

---

**Sebelumnya:** [09 — Lab: Grafana](09-lab-grafana.md) · **Selanjutnya:** [11 — Cleanup](11-cleanup.md)

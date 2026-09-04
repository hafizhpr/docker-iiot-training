# 13 — Bonus: Backup, Restore, dan Migrasi ke Mesin Lain

Stack yang hanya jalan di laptop pembuatnya adalah demo. Pelajaran ini
mengubahnya menjadi sesuatu yang bisa kamu serahkan ke orang lain.

Memindahkannya adalah tiga pekerjaan terpisah. Orang gagal karena hanya
memikirkan salah satunya.

## Tiga Hal yang Harus Ikut Pindah

| Apa | Tempatnya | Cara Pindahnya |
|---|---|---|
| Konfigurasi | folder repo ini — `docker-compose.yml`, `.env`, `mosquitto/config/`, `nodered/Dockerfile` | salin foldernya, atau `git clone` |
| Image | store milik Docker sendiri, sekitar 2,5 GB di disk | `docker pull` di mesin baru, atau `docker save` / `docker load` kalau tidak ada internet |
| Data | enam named volume | `tar` lewat container sekali pakai, seperti di Pelajaran 11 |

Salin foldernya saja dan mesin baru akan menjalankan stack yang sehat
sempurna — tanpa measurement, tanpa flow, tanpa dashboard. Itulah kesalahan
yang biasa terjadi, dan tampak seperti keberhasilan sampai seseorang membuka
Grafana.

## Mengapa Nama Volume Tetap Cocok

Baris 10 di `docker-compose.yml`:

```yaml
name: iiot-training
```

Itulah yang membuat volumenya bernama `iiot-training_influxdb_data` di setiap
mesin yang menjalankan project ini. Tanpa baris itu, Compose mengambil nama
project dari nama folder. Ekstrak materi ini ke `Downloads\docker-course` di PC
baru dan kamu mendapat `docker-course_influxdb_data` — enam volume kosong yang
baru, dan restore yang tampak berhasil sementara stack-nya mengabaikannya.

Memaku nama project adalah satu baris yang membuat sisa pelajaran ini mungkin.

## Langkah 1 — Hentikan Stack

```powershell
docker compose stop
```

InfluxDB adalah database dengan file yang terbuka. Arsipkan saat ia berjalan
dan kamu menangkap keadaan setengah tertulis. `stop` bukan `down` — tidak ada
yang dihapus.

## Langkah 2 — Backup Keenam Volume

Pelajaran 11 mem-backup satu volume. Ini perintah yang sama dalam loop:

```powershell
mkdir backup
$volumes = "mosquitto_data","mosquitto_log","influxdb_data","influxdb_config","nodered_data","grafana_data"
foreach ($v in $volumes) {
  docker run --rm -v "iiot-training_${v}:/source:ro" -v "${PWD}/backup:/backup" alpine tar czf "/backup/$v.tar.gz" -C /source .
}
```

Periksa hasilnya:

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

Enam file, tidak ada yang nol byte. Ukurannya berbeda jauh dan itu wajar —
`influxdb_config` hanya beberapa ratus byte JSON, sedangkan `grafana_data`
membawa seluruh database SQLite plus folder plugin.

Apa yang disimpan masing-masing volume untukmu:

| Volume | Yang Hilang Kalau Dilewat |
|---|---|
| `mosquitto_data` | Retained message |
| `mosquitto_log` | Riwayat log broker — satu-satunya yang aman dilewat |
| `influxdb_data` | Setiap measurement yang pernah kamu tulis |
| `influxdb_config` | Org, bucket, dan token yang dibuat InfluxDB saat setup pertama |
| `nodered_data` | Semua flow, plus node palette yang kamu pasang manual |
| `grafana_data` | Semua dashboard, datasource-nya, dan password admin |

⚠️ **`influxdb_data` dan `influxdb_config` adalah pasangan.** Yang satu
databasenya, yang lain identitas yang membukanya. Bawa satu tanpa yang lain dan
InfluxDB akan bangun sebagai orang asing bagi datanya sendiri.

## Langkah 3 — Pindahkan Image, Kalau Mesin Baru Tanpa Internet

Lewati langkah ini sepenuhnya kalau PC tujuan bisa menjangkau internet:
`docker compose up -d` akan menarik keempat image itu sendiri.

PC pabrik biasanya tidak bisa. Bungkus image-nya jadi satu file:

```powershell
docker compose config --images
docker save -o iiot-images.tar eclipse-mosquitto:2 influxdb:2.7 nodered/node-red:4.1.0-22 grafana/grafana:12.3.3
```

`docker compose config --images` mencetak keempat tag persis yang dipakai
project ini, jadi kamu tidak perlu menebak. `docker save` mempertahankan tag
tersebut — itulah sebabnya Pelajaran 01 bersikeras memaku versi alih-alih
`:latest`. Image yang disimpan sebagai `:latest` tiba di mesin baru dengan arti
yang sama sekali berbeda.

Periksa file yang kamu hasilkan:

```powershell
Get-Item iiot-images.tar | Select-Object Name, Length
```

Sekitar **580 MB** — jauh lebih kecil dari 2,5 GB yang ditempati keempat image
yang sama menurut `docker image ls`. Layer berjalan dalam keadaan terkompresi
di dalam arsip dan mengembang saat Docker membongkarnya, dan layer yang dipakai
bersama antar-image hanya disimpan sekali. `docker image ls` melaporkan ukuran
setelah dibongkar; rencanakan kapasitas flash disk berdasarkan arsipnya, dan
ruang kosong mesin tujuan berdasarkan angka 2,5 GB.

## Langkah 4 — Apa yang Sebenarnya Kamu Bawa

```
transfer/
├── docker-iiot-training/     folder repo
├── backup/
│   ├── mosquitto_data.tar.gz
│   ├── mosquitto_log.tar.gz
│   ├── influxdb_data.tar.gz
│   ├── influxdb_config.tar.gz
│   ├── nodered_data.tar.gz
│   └── grafana_data.tar.gz
└── iiot-images.tar          hanya kalau mesin baru offline
```

Flash disk, network share, terserah kamu. Docker tidak terlibat di bagian ini.

## Langkah 5 — Restore di Mesin Baru, Dengan Urutan Ini

Urutannya adalah inti pelajaran ini. Salah urutan dan gejalanya membingungkan.

**1. Taruh folder repo di mana pun kamu mau.** Nama foldernya tidak penting —
`name: iiot-training` yang menentukan nama volume, bukan foldernya.

**2. Muat image-nya** — hanya untuk kasus offline:

```powershell
docker load -i iiot-images.tar
```

**3. Restore volume-nya.** Volume itu belum ada di mesin ini; flag `-v` membuat
masing-masing sesuai kebutuhan:

```powershell
$volumes = "mosquitto_data","mosquitto_log","influxdb_data","influxdb_config","nodered_data","grafana_data"
foreach ($v in $volumes) {
  docker run --rm -v "iiot-training_${v}:/target" -v "${PWD}/backup:/backup" alpine sh -c "cd /target && tar xzf /backup/$v.tar.gz"
}
```

**4. Baru sekarang jalankan stack-nya:**

```powershell
docker compose up -d
```

Compose akan mencetak satu peringatan per volume:

```
volume "iiot-training_grafana_data" already exists but was not created by
Docker Compose. Use `external: true` to use an existing volume
```

Enam peringatan seperti itu, dan semuanya memang diharapkan. Kamu yang membuat
volume tersebut di langkah 3, sebelum Compose sempat melakukannya. Tidak ada
yang salah dan tidak ada yang perlu diubah.

⚠️ **Jangan jalankan `up -d` sebelum restore.** `tar xzf` mengekstrak **di
atas** apa pun yang sudah ada di volume — ia menggabung, bukan mengganti.
Jalankan stack lebih dulu dan InfluxDB akan menjalankan `setup` (Pelajaran 05),
menginisialisasi kedua volumenya; restore di atas itu meninggalkan dua database
yang tercampur dalam satu direktori. Kalau kamu sudah terlanjur melakukannya,
jalankan `docker compose down -v` lalu mulai Langkah 5 lagi dari keadaan
bersih.

## Langkah 6 — Verifikasi Migrasinya Benar-benar Berhasil

Empat baris `Up` bukan bukti. Datanya yang jadi tujuan:

| Cek | Caranya | Lolos |
|---|---|---|
| Container | `docker compose ps` | empat baris, semua `Up` |
| Measurement | query Influx di bawah | baris-baris dengan timestamp lamamu |
| Flow | http://localhost:1880 | tab-tabmu, bukan editor kosong |
| Dashboard | http://localhost:3000 | dashboard-mu, datasource `working` |
| Retained MQTT | `docker compose exec mosquitto mosquitto_sub -t "factory/#" -C 1` | payload lama muncul seketika |

```powershell
docker compose exec influxdb influx query --org training --token training-token-do-not-use-in-production 'from(bucket:\"sensors\") |> range(start:-30d) |> count()'
```

Jendela 30 hari, bukan jendela 1 jam yang dipakai di bagian lain materi ini —
kamu sedang mencari pembacaan yang dibuat sebelum perpindahan.

Penanda paling jelas dari restore yang gagal: **kalau Grafana memintamu
mengganti password admin, `grafana_data` tidak ter-restore.** Prompt itu adalah
wizard first-run Grafana, dan Grafana yang ter-restore tidak pernah
menampilkannya.

⚠️ **Masuk ke Grafana hasil restore memakai password dari mesin lama, bukan
yang ada di `.env`.** `GF_SECURITY_ADMIN_PASSWORD` hanya menanam user admin
saat Grafana pertama kali membuat databasenya. Setelah restore, database itu
sudah ada, jadi variabel tersebut diabaikan dan password lama tetap berlaku —
jebakan yang sama dengan `setup` InfluxDB di Pelajaran 05. `401 Unauthorized`
di sini justru bukti restore-nya **berhasil**: Grafana yang masih segar pasti
menerima `admin`.

## Yang Tidak Ikut Pindah

| Hal | Alasan |
|---|---|
| Alamat IP container | Diberikan baru di bridge mesin baru. Persis inilah alasan Pelajaran 04 melarangmu mencatatnya |
| Target `host.docker.internal` | Ia menunjuk ke host **baru**. PLC yang terjangkau dari jaringan laptop lama bisa jadi tak terjangkau dari sini |
| Port yang dipublish | Kalau sesuatu di mesin baru sudah memakai `:3000`, Compose gagal start — Pelajaran 10 punya solusinya |
| Path bind mount absolut | `./mosquitto/config` bersifat relatif, jadi ikut pindah. `C:\Users\kamu\...` tidak |

Isi volume selalu di sisi Linux apa pun host-mu, jadi tarball-nya berpindah
antara mesin Windows dan Linux tanpa konversi.

## Jawaban untuk Produksi

Mem-`tar` volume berlaku untuk container apa pun, dan itulah kenapa layak
dipelajari sekali. Khusus untuk database, tool bawaan engine-nya lebih baik
karena bisa berjalan saat database hidup:

```powershell
docker compose exec influxdb influx backup /tmp/backup --token training-token-do-not-use-in-production
```

Grafana mengekspor dashboard sebagai JSON, dan Node-RED mengekspor flow dengan
cara yang sama. Pakai itu kalau stack tidak boleh berhenti. Pakai `tar` kalau
kamu ingin satu metode yang mencakup keenam volume dan setiap container yang
akan kamu temui.

## Checkpoint

1. Kamu menyalin folder ke PC baru, menjalankan `docker compose up -d`, dan
   semuanya start dengan bersih — tapi setiap dashboard kosong. Apa yang lupa
   kamu bawa?
2. Mengapa restore volume harus dilakukan *sebelum* `up -d` pertama, bukan
   sesudahnya?
3. Dua volume mana yang harus selalu di-restore bersamaan, dan apa yang rusak
   kalau kamu hanya me-restore salah satunya?
4. Mesin baru tidak punya internet. Dua perintah apa yang memindahkan image ke
   sana?

---

**Sebelumnya:** [12 — Bonus: Membangun Custom Image](12-building-custom-images.md) · **Kembali ke:** [README](../README.md)

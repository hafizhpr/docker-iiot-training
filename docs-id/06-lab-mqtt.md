# 06 — Lab: MQTT dengan Mosquitto

**Tujuan:** publish dan subscribe pesan, pelajari konvensi topic yang digunakan
kursus ini, dan pahami mengapa `mosquitto.conf` ada sama sekali.

Semua perintah dalam kursus ini menggunakan **PowerShell**. Buka PowerShell di
folder proyek sebelum memulai.

## Jalankan Stack

```powershell
docker compose up -d
docker compose ps
```

Semua empat container harus menampilkan `Up`.

## Mengapa File Konfigurasi Wajib Ada

Mosquitto 2.x mengubah sebuah default yang mengejutkan hampir semua orang: **tanpa
deklarasi listener, ia hanya bind ke loopback di dalam container**. Container
berjalan, log terlihat bersih, port 1883 dipublikasikan — dan setiap client ditolak.

`mosquitto/config/mosquitto.conf` kita memperbaiki itu:

```conf
listener 1883
allow_anonymous true
```

| Baris | Efek |
|---|---|
| `listener 1883` | Terima koneksi di port 1883 dari interface manapun |
| `allow_anonymous true` | Tidak memerlukan username atau password |

⚠️ `allow_anonymous true` berarti siapapun yang bisa menjangkau port 1883 bisa
publish apapun. Dapat diterima di ruang kelas di mesinmu sendiri, tidak pernah
dapat diterima di jaringan pabrik.

File ini adalah **bind mount**, jadi kamu mengeditnya di host dan restart broker
untuk menerapkan perubahan:

```powershell
docker compose restart mosquitto
docker compose logs mosquitto
```

## Konvensi Topic

Sebelum mempublish apapun, sepakati bentuk topic-mu. Kursus ini menggunakan satu
konvensi di mana-mana:

```
factory / <line> / <device> / telemetry / <measurement>     device  →  system
factory / <line> / <device> / command   / <action>          system  →  device
```

| Segmen | Arti | Contoh |
|---|---|---|
| `factory` | Root site | `factory` |
| `<line>` | Lini produksi atau area | `line1` |
| `<device>` | Peralatannya sendiri | `press01` |
| `telemetry` \| `command` | **Arah perjalanan data** | lihat di bawah |
| `<measurement>` / `<action>` | Apa yang dilaporkan atau diminta | `temperature`, `setpoint` |

### Mengapa Segmen Arah Penting

Ini bagian yang patut diperlambat.

| Cabang | Siapa yang publish | Siapa yang subscribe | Arti |
|---|---|---|---|
| `telemetry` | Perangkat | Historian, dashboard, alarm | "ini yang saya ukur" |
| `command` | Sistem kontrol | Perangkat | "tolong lakukan ini" |

Siapapun yang membaca topic langsung tahu ke mana arah data mengalir. Yang lebih
praktis, ini memungkinkan access control di kemudian hari: sensor mendapat izin
hanya untuk publish di bawah `telemetry`, dan tidak pernah ke `command`. Tanpa
segmen arah, kamu tidak bisa mengungkapkan aturan itu sama sekali.

Jika kamu pernah bekerja dengan database tag SCADA, ini disiplin yang sama dengan
memisahkan read tag dari write tag — ditegakkan oleh nama topic alih-alih oleh
konvensi di kepala seseorang.

### Contoh

```
factory/line1/press01/telemetry/temperature
factory/line1/press01/telemetry/pressure
factory/line2/press02/telemetry/temperature
factory/line1/press01/command/setpoint
factory/line1/press01/command/reset
```

### Subscribe ke Irisan Tertentu

| Subscription | Mendapat |
|---|---|
| `factory/#` | Semuanya |
| `factory/+/+/telemetry/#` | Semua telemetry dari setiap perangkat |
| `factory/line1/+/telemetry/#` | Semua telemetry dari line 1 |
| `factory/+/+/telemetry/temperature` | Setiap temperature di site |
| `factory/line1/press01/command/#` | Setiap command yang ditujukan ke satu press |

Salah di sini mahal untuk diperbaiki nanti, karena setiap perangkat harus
dikonfigurasi ulang. Luangkan waktu di awal.

## Subscribe

Image Mosquitto sudah menyertakan tool client `mosquitto_sub` dan `mosquitto_pub`,
jadi tidak perlu instalasi tambahan.

Subscribe ke semua telemetry:

```powershell
docker compose exec mosquitto mosquitto_sub -h localhost -t 'factory/+/+/telemetry/#' -v
```

Ini memblokir dan menunggu. Biarkan tetap berjalan.

> **Menjalankan lab ini untuk kedua kalinya?** Pesan mungkin muncul sesaat setelah
> kamu subscribe, sebelum kamu mempublish apapun. Itu adalah *retained* message
> yang tersisa dari run sebelumnya — broker menyimpannya di volume `mosquitto_data`,
> sehingga `docker compose restart` maupun `docker compose down` tidak menghapusnya.
> Tidak ada yang rusak. Bagian retained-message di bawah menjelaskan apa itu;
> Pelajaran 11 menunjukkan cara menghapusnya.

> `-h localhost` benar **di sini** karena perintah berjalan di dalam container
> mosquitto. Dari container lain manapun itu akan menjadi `-h mosquitto`.
> Pelajaran 04 menjelaskan alasannya.

> Topic dibungkus dalam **tanda kutip tunggal** di seluruh kursus ini. PowerShell
> tidak merusak `#` di tengah kata, jadi `-t factory/#` yang tidak dikutip
> kebetulan berfungsi — tapi tetap kutip. Tidak ada ruginya, dan itu menyelamatkanmu
> saat topic berisi spasi atau `$`.

## Publish

Buka **jendela PowerShell kedua** di folder yang sama:

```powershell
cd C:\Users\<you>\projects\docker-iiot-training
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/temperature' -m '42.5'
```

Jendela pertama mencetak:

```
factory/line1/press01/telemetry/temperature 42.5
```

Kirim beberapa lagi:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/pressure' -m '3.2'
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line2/press02/telemetry/temperature' -m '38.1'
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/command/setpoint' -m '45'
```

Dua yang pertama datang. **Command** tidak datang — subscription-mu hanya mencakup
cabang `telemetry`. Itulah segmen arah yang bekerja.

Perhatikan cabang command sebagai gantinya, di jendela kedua:

```powershell
docker compose exec mosquitto mosquitto_sub -h localhost -t 'factory/+/+/command/#' -v -C 1 -W 20
```

Kemudian publish setpoint lagi dari jendela pertama. Ia datang.

Hentikan subscriber dengan Ctrl-C.

## Wildcard Topic

| Wildcard | Cocok dengan | Contoh |
|---|---|---|
| `+` | Tepat satu level | `factory/+/+/telemetry/temperature` |
| `#` | Semua level yang tersisa | `factory/line1/#` |

`#` harus menjadi karakter terakhir dari topic. `factory/#/telemetry` tidak valid.

`+` adalah yang memungkinkanmu menulis `factory/+/+/telemetry/#` dan mengambil
setiap perangkat di setiap line tanpa mendaftarkannya.

## Retained Message

Retained message disimpan oleh broker dan langsung dikirimkan ke subscriber masa
depan manapun. Publish satu dengan `-r`:

```powershell
docker compose exec mosquitto mosquitto_pub -h localhost -t 'factory/line1/press01/telemetry/status' -m 'running' -r
```

Sekarang subscribe *setelah* fakta tersebut — satu jendela sudah cukup, tidak perlu
terminal kedua:

```powershell
docker compose exec mosquitto mosquitto_sub -h localhost -t 'factory/line1/press01/telemetry/status' -v -C 1 -W 5
```

Kamu menerima `running` secara instan, meskipun sudah dipublish sebelumnya. Tanpa
`-r` kamu akan menunggu publish berikutnya dan time out.

Ini adalah cara dashboard menampilkan status mesin saat ini begitu ia terhubung —
ide yang sama dengan tag SCADA yang menyimpan nilai terakhir yang diketahui.

Retained message ditulis ke `mosquitto_data`, sehingga bertahan setelah restart:

```powershell
docker compose restart mosquitto
docker compose exec mosquitto mosquitto_sub -h localhost -t 'factory/line1/press01/telemetry/status' -v -C 1 -W 5
```

Masih ada. Volume dan semantik MQTT bertemu dalam satu tempat.

## Flag yang Berguna

| Flag | Arti |
|---|---|
| `-v` | Cetak topic bersama payload |
| `-C 1` | Keluar setelah menerima 1 pesan |
| `-W 10` | Menyerah setelah 10 detik |
| `-r` | Publish sebagai retained |
| `-q 1` | QoS 1 — pengiriman minimal sekali |

`-C 1 -W 5` bersama-sama adalah cara yang reliabel untuk menguji subscription
dari satu jendela PowerShell: ambil satu pesan, kemudian menyerah setelah lima
detik.

## Checkpoint

1. Mengapa Mosquitto 2.x memerlukan `listener 1883` dalam file konfigurasi?
2. Sensor tekanan dan HMI operator keduanya berbicara ke `press01`. Tulis
   topic yang masing-masing publish.
3. Kamu punya satu jendela PowerShell dan ingin membuktikan ada retained message.
   Dua flag mana yang kamu butuhkan?
4. Volume mana yang menjaga retained message tetap ada setelah restart?

---

**Sebelumnya:** [05 — Volumes dan Persistensi](05-volumes.md) · **Selanjutnya:** [07 — Lab: Node-RED](07-lab-nodered.md)

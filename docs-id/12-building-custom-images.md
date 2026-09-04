# 12 — Bonus: Membangun Custom Image

Di Pelajaran 07 kamu menginstal `node-red-contrib-influxdb` secara manual melalui
palette manager. Berhasil. Tapi ada tiga masalahnya:

1. Setiap pelajar mengulanginya, dan ada yang salah ketik namanya
2. Ia hilang ketika volume dihapus
3. Itu adalah langkah manual, sehingga tidak bisa diotomasi atau dikontrol versinya

Solusinya adalah membangun image-mu sendiri dengan node sudah ada di dalamnya.

## Dasar-Dasar Dockerfile

`Dockerfile` adalah resep. Setiap instruksi menambahkan layer di atas yang terakhir.

```dockerfile
FROM nodered/node-red:4.1.0-22

RUN npm install --no-audit --no-fund --no-update-notifier \
    node-red-contrib-influxdb
```

| Instruksi | Arti |
|---|---|
| `FROM` | Image dasar untuk memulai — selalu instruksi pertama |
| `RUN` | Jalankan perintah **saat build** dan simpan hasilnya dalam layer |

Perhatikan *saat build*. `RUN` terjadi sekali, ketika image dibangun. Ia tidak
dijalankan saat container start.

File ini sudah ada di repositori di `nodered/Dockerfile`.

### Mengapa `/usr/src/node-red` Penting

Image dasar mengatur working directory-nya ke `/usr/src/node-red`, di mana
`package.json` Node-RED sendiri berada. Menginstal di sana menaruh node **di
dalam image**.

Menginstal ke `/data` sebagai gantinya akan menaruhnya di volume — yang persis
situasi yang ingin kita hindari.

## Instruksi Umum

| Instruksi | Tujuan |
|---|---|
| `FROM` | Image dasar |
| `RUN` | Jalankan perintah saat build |
| `COPY` | Salin file dari mesinmu ke dalam image |
| `WORKDIR` | Atur working directory untuk instruksi berikutnya |
| `ENV` | Set environment variable |
| `EXPOSE` | Dokumentasikan port mana yang didengarkan aplikasi |
| `CMD` | Perintah default saat container start |

`EXPOSE` mendokumentasikan; ia tidak mempublikasikan. Publikasi tetap menggunakan
`-p` atau key `ports:` di compose. Ini sering membingungkan orang.

## Build

```powershell
docker build -t my-nodered:1.0 ./nodered
```

| Bagian | Arti |
|---|---|
| `-t my-nodered:1.0` | Tag: nama dan versi |
| `./nodered` | Build context — direktori yang dikirim ke builder |

Perhatikan outputnya. Setiap instruksi menjadi langkah, dan setiap langkah
menjadi layer.

Konfirmasi sudah ada:

```powershell
docker images | Select-String my-nodered
```

## Gunakan di Compose

Compose bisa membangun image untukmu. Ganti baris `image:` di service `nodered`:

```yaml
  nodered:
    build: ./nodered          # sebagai pengganti: image: nodered/node-red:4.1.0-22
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1880:1880"
    environment:
      TZ: ${TZ}
    volumes:
      - nodered_data:/data
    networks:
      - iiot-net
```

Kemudian:

```powershell
docker compose up -d --build
```

`--build` memaksa rebuild. Tanpanya, Compose menggunakan apapun yang dibangun
terakhir kali dan perubahan Dockerfile-mu diabaikan — sumber kebingungan yang sering.

Verifikasi hasilnya dengan menghapus volume dan start dari awal, kemudian buka
<http://localhost:1880>. Node InfluxDB ada di palette pada volume yang benar-benar
kosong. Tidak ada yang menginstal apapun.

## Layer Caching

Docker menyimpan setiap layer dalam cache. Saat rebuild ia menggunakan kembali
setiap layer hingga instruksi pertama yang inputnya berubah — kemudian membangun
ulang segalanya setelahnya.

Ini menjadikan urutan instruksi sebagai keputusan performa:

```dockerfile
# Lambat: perubahan source apapun menjalankan ulang npm install
COPY . /app
RUN npm install

# Cepat: npm install hanya dijalankan ulang ketika package.json berubah
COPY package.json package-lock.json /app/
RUN npm install
COPY . /app
```

Letakkan yang jarang berubah di awal, dan yang sering berubah di akhir.

## Menjaga Image Tetap Kecil

```powershell
docker images
```

Bandingkan `nodered/node-red` dengan `my-nodered`. Perbedaannya adalah layer
yang kamu tambahkan — kecil di sini, tapi kebiasaannya penting seiring image
bertumbuh:

| Teknik | Alasan |
|---|---|
| Pilih base slim (`-alpine`, `-slim`) | Sering 10x lebih kecil |
| Rangkai perintah `RUN` terkait dengan `&&` | Satu layer alih-alih beberapa |
| Bersihkan package cache dalam `RUN` yang sama | Langkah cleanup terpisah tidak mengecilkan layer sebelumnya |
| Gunakan `.dockerignore` | Menjaga `node_modules` dan `.git` dari build context |

Poin ketiga mengejutkan banyak orang: **menghapus file di layer berikutnya tidak
menghapusnya dari image**, hanya menyembunyikannya. Byte-nya masih dikirimkan.

Bersihkan di dalam `RUN` yang sama yang membuat kekacauan, dirangkai dengan `&&`,
sehingga cache tidak pernah menjadi layer tersendiri.

## Kapan Build vs Kapan Konfigurasi

| Masuk ke image | Masuk ke volume atau environment |
|---|---|
| Package dan node yang terinstal | Flow yang kamu edit |
| Kode aplikasi | Dashboard |
| Dependensi sistem | Password dan token |
| Konfigurasi default | Pengaturan per-deployment |

Garis pembatasnya: **apapun yang identik untuk setiap deployment masuk ke image;
apapun yang berbeda tetap di luar.** Memasukkan password ke dalam image berarti
kamu harus rebuild untuk memutarnya — dan itu tetap ada di riwayat image selamanya.

## Latihan

Perluas `nodered/Dockerfile` untuk juga menginstal `node-red-dashboard`, rebuild,
dan konfirmasi bahwa node dashboard muncul pada volume yang kosong.

<details>
<summary>Solusi</summary>

```dockerfile
FROM nodered/node-red:4.1.0-22

RUN npm install --no-audit --no-fund --no-update-notifier \
    node-red-contrib-influxdb \
    node-red-dashboard
```

Kemudian rebuild dengan `docker compose up -d --build`.

</details>

## Checkpoint

1. Apa perbedaan antara `RUN` dan `CMD`?
2. Mengapa `EXPOSE 1880` tidak membuat port bisa dijangkau dari browsermu?
3. Mengapa menghapus file di layer berikutnya gagal mengecilkan image?

---

**Sebelumnya:** [11 — Cleanup](11-cleanup.md) · **Selanjutnya:** [13 — Bonus: Backup, Restore, dan Migrasi ke Mesin Lain](13-backup-migration.md)

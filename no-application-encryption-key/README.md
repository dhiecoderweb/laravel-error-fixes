# No Application Encryption Key Laravel: Cara Mengatasinya

```
Illuminate\Encryption\MissingAppKeyException
No application encryption key has been specified.
```


Error `No application encryption key has been specified` muncul karena Laravel tidak menemukan nilai `APP_KEY` saat membutuhkan enkripsi, misalnya untuk mengenkripsi cookie sesi. Solusi untuk proyek baru cukup satu perintah: `php artisan key:generate`. Namun di server produksi, penyebabnya sering lebih halus, seperti file `.env` yang tidak ikut ter-deploy atau konfigurasi yang sudah di-cache sebelum key diisi.

Hal yang paling penting: **jangan membuat key baru secara sembarangan di aplikasi yang sudah berjalan**. Mengganti `APP_KEY` membuat seluruh pengguna ter-logout dan data yang dienkripsi dengan key lama tidak bisa dibaca lagi. Artikel ini membahas penyebab dari yang paling sering terjadi beserta cara memperbaikinya dengan aman.

## Apa fungsi APP_KEY?

`APP_KEY` adalah kunci rahasia yang dipakai oleh layanan enkripsi Laravel (`Illuminate\Encryption\Encrypter`) dengan cipher default `AES-256-CBC`. Kunci ini dipakai untuk:

- Mengenkripsi cookie, termasuk cookie sesi, melalui middleware `EncryptCookies`.
- Data yang dienkripsi dengan `Crypt::encryptString()` atau cast `encrypted` pada model Eloquent.
- Membuat tanda tangan untuk *signed URL* (misalnya link verifikasi email).


Password pengguna **tidak** memakai `APP_KEY`. Password disimpan sebagai hash (bcrypt atau argon) melalui `Hash::make()`, sehingga tidak terpengaruh ketika key berubah. Banyak orang keliru di sini, lalu takut atau justru terlalu berani mengganti key.

Key yang valid berbentuk string base64 dengan prefix `base64:`, berisi 32 byte acak untuk AES-256-CBC:

```
APP_KEY=base64:Zm9vYmFyYmF6cXV4cXV1eGNvcmdlZ3JhdWx0Z2FycGx5PQ==
```


Contoh di atas hanya ilustrasi format. Jangan pernah menyalin key dari artikel, tutorial, atau repositori orang lain.

## Penyebab #1: Proyek baru di-clone dan .env belum dibuat

File `.env` tidak di-commit ke Git (dan memang tidak boleh). Saat Anda meng-clone proyek, yang ada hanya `.env.example` dengan `APP_KEY=` kosong. Langkah standarnya:

```
cp .env.example .env
composer install
php artisan key:generate
```


Perintah `key:generate` membuat key acak dan menuliskannya langsung ke baris `APP_KEY=` di file `.env`. Perhatikan dua hal:

- File `.env` harus sudah ada. Jika tidak ada, perintah gagal menulis key.
- Baris `APP_KEY=` harus ada di `.env`. Versi Laravel yang lebih baru akan menolak dengan pesan bahwa variabel `APP_KEY` tidak ditemukan, alih-alih diam-diam tidak menulis apa pun.


Jika Anda hanya ingin melihat key tanpa mengubah file (misalnya untuk diisi manual di panel hosting), gunakan:

```
php artisan key:generate --show
```


## Penyebab #2: Konfigurasi sudah di-cache sebelum key diisi

Ini penyebab paling sering di server produksi. Urutan kejadiannya biasanya begini: Anda menjalankan `php artisan config:cache` saat `APP_KEY` masih kosong, lalu menjalankan `key:generate`. File `.env` sudah berisi key, tetapi Laravel tetap membaca `bootstrap/cache/config.php` yang menyimpan key kosong. Error tidak hilang.

```
php artisan config:clear
php artisan config:cache
```


Ingat juga: ketika config di-cache, fungsi `env()` yang dipanggil di luar folder `config/` akan mengembalikan `null`. Jika ada kode Anda yang membaca `env('APP_KEY')` langsung, ganti menjadi `config('app.key')`.

Setelah membuat ulang cache, restart PHP-FPM bila OPcache tidak memvalidasi timestamp, dan jalankan `php artisan queue:restart` agar worker memuat konfigurasi baru.

## Penyebab #3: File .env tidak ikut ter-deploy atau tidak terbaca

Pada deploy otomatis (GitHub Actions, rsync, Docker build), file `.env` sering tertinggal karena memang dikecualikan dari repositori. Periksa langsung di server:

```
cd /var/www/aplikasi-anda
ls -la .env
grep '^APP_KEY=' .env
php artisan about | grep -i -A3 environment
```


Beberapa hal yang perlu dipastikan:

- **Lokasi file.** `.env` harus berada di root proyek (sejajar dengan `artisan`), bukan di folder `public/`. Pada shared hosting, file ini sering salah ditaruh ke `public_html`.
- **Izin baca.** User web server (misalnya `www-data`) harus bisa membaca file. Jika file dimiliki root dengan izin `600`, PHP-FPM tidak dapat membacanya.
- **Format.** Tidak ada spasi di sekitar tanda `=` dan tidak ada karakter aneh dari copy-paste.


Di Docker, cara yang umum adalah menyuntikkan `APP_KEY` sebagai environment variable container, bukan memasukkan `.env` ke image. Perlu diketahui, environment variable yang sudah ada di sistem tidak ditimpa oleh `.env`. Jadi kalau container memiliki `APP_KEY` kosong dari konfigurasi orkestrasi, nilai di `.env` tidak akan dipakai.

```
# docker-compose.yml (potongan)
services:
  app:
    environment:
      APP_KEY: ${APP_KEY}
```


## Penyebab #4: Environment pengujian tidak punya key

Saat menjalankan `php artisan test` atau PHPUnit di CI, Laravel memuat `.env.testing` jika ada, dan variabel di `phpunit.xml`. Jika keduanya tidak menyediakan `APP_KEY` dan CI tidak punya file `.env`, test yang menyentuh sesi atau cookie akan gagal dengan error ini. Tambahkan langkah pembuatan key di pipeline:

```
cp .env.example .env
php artisan key:generate
```


## Penyebab #5: Key ada, tetapi formatnya salah

Jika key terisi tetapi panjang atau formatnya tidak sesuai, error yang muncul biasanya berbeda: `Unsupported cipher or incorrect key length`. Ini terjadi jika prefix `base64:` terhapus, key terpotong saat disalin, atau `APP_CIPHER`/`cipher` di `config/app.php` diubah tanpa menyesuaikan panjang key. Solusinya sama: gunakan key hasil `key:generate` secara utuh.

## Jangan merotasi APP_KEY secara membabi buta

Saat panik, banyak yang menjalankan `php artisan key:generate --force` di produksi. Jika aplikasi sebelumnya sudah memiliki key yang valid (misalnya error ternyata hanya karena cache), tindakan ini menimbulkan masalah baru:  Yang terdampakAkibat setelah key diganti  Cookie sesiTidak bisa didekripsi, semua pengguna ter-logout Cookie "remember me"Tidak berlaku, pengguna harus login ulang Data `Crypt::encryptString()` / cast `encrypted`Gagal dibaca, muncul `DecryptException: The MAC is invalid.` Signed URL yang sudah dikirimDianggap tidak valid Password pengguna (hash)Tidak terpengaruh

Data terenkripsi di database adalah risiko terbesar, karena tanpa key lama data tersebut praktis hilang. Sebelum mengganti key, selalu cek apakah key lama masih ada di backup `.env`, panel hosting, atau secret manager.

### Rotasi yang aman di Laravel 11 ke atas

Jika key memang harus diganti (misalnya bocor), Laravel 11 menyediakan `APP_PREVIOUS_KEYS`. Key lama dimasukkan ke variabel ini (dipisah koma), sehingga Laravel tetap bisa mendekripsi data lama, sementara enkripsi baru memakai key baru:

```
APP_KEY=base64:KEY_BARU...
APP_PREVIOUS_KEYS=base64:KEY_LAMA...
```


Dengan cara ini pengguna tidak langsung ter-logout. Setelah semua data penting dienkripsi ulang dengan key baru, key lama bisa dihapus. Pada versi sebelum Laravel 11, Anda perlu menulis skrip migrasi sendiri: dekripsi dengan key lama, enkripsi ulang dengan key baru.

## Memeriksa key yang benar-benar dipakai aplikasi

Isi file `.env` belum tentu sama dengan nilai yang dibaca Laravel, terutama jika ada cache konfigurasi atau environment variable dari sistem. Untuk melihat nilai yang benar-benar aktif, tanyakan langsung ke aplikasi melalui Tinker:

```
php artisan tinker
> filled(config('app.key'))
= true
> app()->configurationIsCached()
= false
```


Jika `filled(config('app.key'))` bernilai `false` padahal `.env` sudah terisi, berarti Laravel membaca sumber lain: cache konfigurasi lama atau environment variable kosong dari server. Hindari mencetak isi key ke layar atau log bersama, karena siapa pun yang memegang key dapat mendekripsi cookie dan data terenkripsi aplikasi Anda.

## Langkah perbaikan yang disarankan di produksi

1. Cek apakah `.env` ada dan `APP_KEY` terisi: `grep '^APP_KEY=' .env`.
2. Jika terisi, jalankan `php artisan config:clear`, lalu `config:cache`. Biasanya error selesai di sini.
3. Jika kosong, cari key lama di backup atau panel deploy. Gunakan key lama bila ada.
4. Hanya jika aplikasi benar-benar baru atau key lama tidak bisa ditemukan, jalankan `php artisan key:generate --force`.
5. Simpan key di tempat aman (password manager atau secret manager) agar tidak hilang saat server diganti.


## Kesalahan umum

- Meng-commit `.env` berisi `APP_KEY` ke repositori publik. Jika sudah terjadi, anggap key bocor dan lakukan rotasi aman.
- Memakai key yang sama untuk banyak aplikasi atau menyalin key dari tutorial.
- Menjalankan `key:generate` di setiap deploy melalui skrip otomatis, sehingga pengguna ter-logout setiap kali deploy.
- Lupa `config:clear` setelah mengisi key.
- Menaruh `.env` di folder publik sehingga bisa diunduh lewat browser.


Untuk membuat password atau secret lain yang kuat (bukan `APP_KEY`, yang sebaiknya selalu dari artisan), Anda dapat memakai [Password Generator](https://dhiecoderweb.com/tools/password). Jika ingin melihat isi string base64, gunakan [Base64 Encode/Decode](https://dhiecoderweb.com/tools/base64) secara lokal di browser. Error lain yang sering menyertai deploy dibahas di [kumpulan solusi error Laravel](https://dhiecoderweb.com/kumpulan-solusi-error-laravel-paling-sering-terjadi) dan panduan [deploy Laravel ke shared hosting cPanel](https://dhiecoderweb.com/cara-deploy-laravel-ke-shared-hosting-cpanel).

## FAQ

### Apakah saya harus menjalankan key:generate setiap kali deploy?

Tidak. Key dibuat sekali per aplikasi per lingkungan, lalu disimpan. Menjalankannya setiap deploy akan mengganti key dan membuat semua pengguna ter-logout.

### Apakah lingkungan lokal dan produksi boleh memakai key yang sama?

Sebaiknya tidak. Gunakan key berbeda per lingkungan agar kebocoran key di laptop pengembang tidak berdampak ke produksi.

### Error masih muncul padahal APP_KEY sudah terisi. Kenapa?

Hampir pasti karena cache konfigurasi. Jalankan `php artisan config:clear`. Jika memakai Docker, pastikan environment variable `APP_KEY` di container tidak kosong, karena nilai tersebut didahulukan dibanding `.env`.

### Apakah mengganti APP_KEY membuat password pengguna tidak bisa dipakai?

Tidak. Password disimpan sebagai hash yang tidak bergantung pada `APP_KEY`. Yang terdampak adalah sesi, cookie, signed URL, dan data yang dienkripsi.


---

<!-- Aktifkan tautan ini setelah artikel terbit di blog:
Artikel lengkap: https://dhiecoderweb.com/no-application-encryption-key-has-been-specified-laravel
-->

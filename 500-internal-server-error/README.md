# Error 500 Internal Server Error di Laravel

Error **500 Internal Server Error** (sering juga muncul sebagai layar putih kosong / *white screen of death*) adalah error yang paling bikin panik, karena Laravel tidak memberi tahu apa penyebabnya di layar. Padahal, penyebabnya biasanya salah satu dari beberapa hal umum. Artikel ini memandu Anda menemukan dan memperbaikinya secara sistematis.

## Langkah 0: Lihat Log Error Dulu

Sebelum menebak-nebak, buka log Laravel — di sinilah penyebab sebenarnya tercatat:

```
storage/logs/laravel.log
```


Lihat baris paling bawah (error terbaru). Untuk sementara, Anda juga bisa mengaktifkan tampilan error dengan mengatur `.env`:

```
APP_DEBUG=true
```


Lalu muat ulang halaman untuk melihat pesan detail. **Kembalikan ke `false`** setelah selesai agar error tidak bocor ke publik.

## 1. Permission Folder storage & bootstrap/cache

Penyebab paling umum di server. Laravel butuh menulis ke dua folder ini. Perbaiki izinnya:

```
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
```


Ganti `www-data` dengan user web server Anda (mis. `apache`, atau user cPanel).

## 2. APP_KEY Belum Di-generate

Jika `APP_KEY` di `.env` kosong, Laravel gagal mengenkripsi dan langsung error 500. Generate:

```
php artisan key:generate
```


## 3. File .env Salah atau Tidak Ada

Setelah upload ke server, sering `.env` lupa dibuat. Pastikan file `.env` ada dan berisi konfigurasi database yang benar. Salin dari `.env.example` bila perlu, lalu isi kredensialnya.

## 4. Config Ter-cache dari Environment Lama

Jika Anda memindahkan project antar-server tetapi cache lama ikut terbawa, config bisa menunjuk ke path/kredensial yang salah. Bersihkan:

```
php artisan config:clear
php artisan cache:clear
php artisan view:clear
php artisan route:clear
```


## 5. Dependency Composer Belum Terpasang

Kalau folder `vendor` tidak ada atau tidak lengkap, aplikasi tidak bisa jalan. Pasang dependency:

```
composer install --optimize-autoloader --no-dev
```


## 6. Versi PHP Tidak Sesuai

Laravel modern butuh PHP versi tertentu (mis. Laravel 10 butuh PHP 8.1+). Kalau server memakai versi lebih lama, akan error. Di cPanel, ubah lewat menu **Select PHP Version**. Cek versi via CLI:

```
php -v
```


## Alur Diagnosis yang Disarankan

1. Aktifkan `APP_DEBUG=true` atau baca `storage/logs/laravel.log`.
2. Baca pesan error spesifiknya — biasanya sudah menunjuk file & baris.
3. Cek permission `storage` & `bootstrap/cache`.
4. Pastikan `.env`, `APP_KEY`, dan `vendor` beres.
5. Bersihkan cache config, lalu coba lagi.
6. Matikan kembali `APP_DEBUG`.


## Kesimpulan

Error 500 di Laravel jarang misterius kalau Anda mulai dari **membaca log**. Sebagian besar kasus disebabkan permission folder, `APP_KEY` kosong, atau `.env` yang salah. Ikuti alur diagnosis di atas secara berurutan dan error biasanya ketemu dalam beberapa menit. Untuk kasus khusus setelah login (error 419), lihat panduan kami tentang mengatasi error 419 Page Expired di Laravel.


---

Artikel lengkap dengan penjelasan tambahan: https://dhiecoderweb.com/cara-mengatasi-error-500-internal-server-error-laravel

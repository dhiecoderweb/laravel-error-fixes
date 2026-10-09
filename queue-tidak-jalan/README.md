# Queue Laravel Tidak Jalan atau Macet: Penyebab & Solusi

```
# gejala 1: job menumpuk di tabel jobs, tidak ada yang memproses
mysql> SELECT COUNT(*) FROM jobs;
+----------+
| COUNT(*) |
+----------+
|     1342 |
+----------+

# gejala 2: job gagal berulang di log / tabel failed_jobs
Illuminate\Queue\MaxAttemptsExceededException:
App\Jobs\SendInvoiceEmail has been attempted too many times.
```


Jika job queue Laravel tidak berjalan, penyebabnya hampir selalu salah satu dari empat hal: tidak ada worker yang berjalan (`queue:work` mati atau tidak pernah dijalankan), `QUEUE_CONNECTION` tidak sesuai sehingga job masuk ke tempat yang tidak diawasi worker, worker masih menjalankan kode lama karena belum di-restart setelah deploy, atau job terus gagal karena timeout dan pengaturan `retry_after` yang salah.

Artikel ini membahas cara mendiagnosis masalah queue dari yang paling mungkin, lengkap dengan perintah dan contoh konfigurasi Supervisor untuk produksi.

## Cara cepat mengetahui di mana masalahnya

Sebelum mengubah apa pun, jawab tiga pertanyaan ini:

1. **Apakah job masuk ke antrean?** Untuk driver database, cek tabel `jobs`. Untuk semua driver, gunakan `php artisan queue:monitor`.
2. **Apakah ada worker yang berjalan?** Cek dengan `ps aux | grep "queue:work"` atau `sudo supervisorctl status`.
3. **Apakah job gagal?** Jalankan `php artisan queue:failed` dan lihat `storage/logs/laravel.log`.


```
php artisan queue:monitor database:default
ps aux | grep "queue:work" | grep -v grep
php artisan queue:failed
```


Jika job menumpuk dan tidak ada worker, masalahnya di proses worker. Jika antrean kosong padahal Anda yakin sudah dispatch, masalahnya di konfigurasi koneksi atau job dijalankan secara sinkron. Jika job ada di `failed_jobs`, masalahnya di kode job atau timeout.

## Penyebab #1: Tidak ada worker yang berjalan

Queue di Laravel tidak berjalan sendiri. Job hanya disimpan ke antrean, dan harus ada proses `queue:work` yang terus berjalan untuk mengambil dan mengeksekusinya. Di lokal, banyak yang lupa menjalankan perintah ini di terminal terpisah. Di server, worker sering mati setelah server reboot atau setelah sesi SSH ditutup.

```
php artisan queue:work

# uji satu job saja dengan output lebih detail
php artisan queue:work --once -v
```


Menjalankan `queue:work` di terminal SSH hanya cocok untuk uji coba. Saat koneksi SSH putus, worker ikut berhenti. Untuk produksi, gunakan process manager seperti Supervisor (dibahas di bawah) agar worker otomatis hidup lagi jika mati.

## Penyebab #2: QUEUE_CONNECTION masih sync atau tidak cocok

Driver `sync` menjalankan job **langsung saat dispatch**, di dalam request yang sama. Tidak ada antrean, tidak ada worker. Jika `.env` masih berisi `QUEUE_CONNECTION=sync` (nilai default di `.env.example` Laravel 10), request akan lambat karena menunggu job selesai, dan error job langsung muncul sebagai error halaman.

Laravel 11 mengubah default menjadi `database`. Untuk memakai driver database atau Redis:

```
# .env
QUEUE_CONNECTION=database
# atau
QUEUE_CONNECTION=redis
```


Driver database membutuhkan tabel `jobs`. Proyek Laravel 11 sudah menyertakan migrasinya. Di Laravel 10, buat dulu:

```
# Laravel 10
php artisan queue:table
php artisan queue:failed-table   # jika belum ada
php artisan migrate

# Laravel 11
php artisan make:queue-table
php artisan migrate
```


Setelah mengubah `.env`, jangan lupa `php artisan config:clear` (atau `config:cache` di produksi). Jika config sudah di-cache, perubahan `QUEUE_CONNECTION` tidak terbaca.

### Worker mendengarkan koneksi atau nama antrean yang salah

Kasus yang lebih halus: job dikirim ke antrean bernama `emails`, tetapi worker hanya memproses antrean `default`. Job akan diam di antrean selamanya.

```
// job dikirim ke antrean "emails"
SendInvoiceEmail::dispatch($invoice)->onQueue('emails');
```


```
# worker harus mendengarkan antrean tersebut (urutan = prioritas)
php artisan queue:work --queue=emails,default

# jika job dikirim ke koneksi tertentu, sebutkan koneksinya
php artisan queue:work redis --queue=emails,default
```


### Job tidak mengimplementasikan ShouldQueue

Jika class job tidak memiliki `implements ShouldQueue`, Laravel menjalankannya secara sinkron walaupun `QUEUE_CONNECTION` sudah benar. Hal yang sama berlaku untuk listener, mail, dan notification yang ingin diantrikan.

```
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class SendInvoiceEmail implements ShouldQueue
{
    use Queueable;
    // ...
}
```


Contoh di atas memakai trait `Queueable` versi Laravel 11. Di Laravel 10, job memakai beberapa trait (`Dispatchable`, `InteractsWithQueue`, `Queueable`, `SerializesModels`) seperti yang dibuat oleh `php artisan make:job`.

## Penyebab #3: Worker tidak di-restart setelah deploy

`queue:work` memuat aplikasi sekali lalu menyimpannya di memori. Artinya, setelah Anda deploy kode baru atau mengubah konfigurasi, worker yang sudah berjalan **tetap memakai kode lama**. Gejalanya membingungkan: bug yang sudah diperbaiki masih muncul, atau job baru gagal dengan error yang tidak masuk akal.

```
php artisan queue:restart
```


Perintah ini memberi sinyal ke semua worker untuk berhenti dengan rapi setelah job yang sedang berjalan selesai. Supervisor lalu menyalakan worker baru dengan kode terbaru. Dua catatan penting:

- Sinyal restart disimpan di **cache**. Pastikan driver cache Anda persisten dan sama untuk CLI dan worker (misalnya `file`, `database`, atau `redis`), bukan `array`.
- Tanpa Supervisor, worker yang berhenti tidak akan hidup lagi. Jadi `queue:restart` tanpa process manager sama dengan mematikan worker.


Masukkan `php artisan queue:restart` ke skrip deploy Anda, setelah `composer install`, migrasi, dan pembuatan cache.

### queue:work vs queue:listen

`queue:listen` memuat ulang aplikasi untuk setiap job, sehingga perubahan kode langsung terbaca tanpa restart. Kekurangannya, jauh lebih boros CPU dan lebih lambat. Gunakan `queue:listen` hanya di lokal saat sedang aktif mengubah kode job; di produksi gunakan `queue:work` ditambah `queue:restart` setiap deploy.

## Menjalankan worker dengan Supervisor di VPS

Supervisor menjaga worker tetap hidup dan menyalakannya ulang bila mati atau server reboot. Pasang dan buat file konfigurasi:

```
sudo apt install supervisor
sudo nano /etc/supervisor/conf.d/laravel-worker.conf
```


```
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/aplikasi-anda/artisan queue:work database --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/aplikasi-anda/storage/logs/worker.log
stopwaitsecs=3600
```


```
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start "laravel-worker:*"
sudo supervisorctl status
```


Beberapa poin dari konfigurasi ini:

- `user` harus user yang bisa menulis ke `storage/`. Jika worker berjalan sebagai root, file log yang dibuatnya bisa membuat web server gagal menulis log.
- `--max-time=3600` membuat worker berhenti sendiri setiap jam lalu dihidupkan ulang Supervisor, berguna untuk mencegah kebocoran memori.
- `stopwaitsecs` sebaiknya lebih besar dari durasi job terpanjang agar job tidak terpotong saat worker dihentikan.
- `numprocs` menentukan jumlah worker. Ingat, setiap worker memakai satu koneksi database.


Langkah lengkap menyiapkan server ada di panduan [deploy Laravel ke VPS Ubuntu dengan Nginx](https://dhiecoderweb.com/cara-deploy-laravel-ke-vps-ubuntu-nginx).

## Penyebab #4: Job gagal atau macet karena timeout dan retry_after

Ini penyebab job yang terlihat "macet" atau dieksekusi dua kali. Ada dua pengaturan yang saling berkaitan:

- **`--timeout`** pada worker (default 60 detik): batas maksimal satu job berjalan sebelum proses worker dihentikan paksa.
- **`retry_after`** di `config/queue.php` (default 90 detik untuk database dan Redis): berapa lama job yang sedang diproses dianggap hilang lalu dilepas kembali ke antrean.


Aturannya: `retry_after` harus **lebih besar** dari `--timeout`, dengan selisih beberapa detik. Jika sebaliknya, job yang masih berjalan dianggap hilang dan diambil worker lain, sehingga job berjalan dua kali. Jika job memang butuh lebih lama, naikkan keduanya:

```
// config/queue.php
'database' => [
    'driver' => 'database',
    'table' => 'jobs',
    'queue' => 'default',
    'retry_after' => 330,
],
```


```
php artisan queue:work database --timeout=300 --tries=3
```


Anda juga bisa menetapkan batas per job dengan properti `public $timeout = 300;` dan `public $tries = 3;` di class job. Pesan `has been attempted too many times` berarti job sudah melewati batas percobaan, sering karena timeout berulang.

### Mengelola failed_jobs

```
php artisan queue:failed            # daftar job gagal
php artisan queue:retry all         # coba ulang semua
php artisan queue:retry 5f1c...     # coba ulang satu job (UUID)
php artisan queue:forget 5f1c...    # hapus satu job gagal
php artisan queue:flush             # hapus semua job gagal
```


Kolom `exception` di tabel `failed_jobs` berisi stack trace lengkap. Baca bagian ini sebelum menjalankan `queue:retry`, karena mencoba ulang tanpa memperbaiki penyebab hanya akan menghasilkan kegagalan yang sama.

### Job dijalankan sebelum transaksi selesai

Jika job di-dispatch di dalam transaksi database, worker bisa mengambilnya sebelum transaksi di-commit, sehingga model belum ada dan job gagal dengan `ModelNotFoundException`. Atasi dengan `->afterCommit()` saat dispatch, atau set `'after_commit' => true` pada koneksi di `config/queue.php`.

## Queue di shared hosting

Banyak shared hosting tidak mengizinkan proses berjalan permanen atau tidak menyediakan Supervisor. Alternatifnya, jalankan worker lewat cron setiap menit dan biarkan berhenti saat antrean kosong:

```
* * * * * cd /home/user/aplikasi && php artisan queue:work --stop-when-empty --max-time=55 >> /dev/null 2>&1
```


Pastikan path PHP sesuai versi yang dipakai hosting (kadang perlu path lengkap seperti `/usr/local/bin/php`). Untuk memahami ekspresi cron, gunakan [Cron Explainer](https://dhiecoderweb.com/tools/cron). Langkah upload aplikasi dibahas di [deploy Laravel ke shared hosting cPanel](https://dhiecoderweb.com/cara-deploy-laravel-ke-shared-hosting-cpanel).

## Kesalahan umum

- Lupa menjalankan worker sama sekali, lalu mengira queue rusak.
- Mengubah `QUEUE_CONNECTION` tanpa `config:clear`.
- Deploy tanpa `queue:restart`, sehingga worker menjalankan kode lama.
- Memakai driver cache `array`, sehingga `queue:restart` tidak pernah sampai ke worker.
- `retry_after` lebih kecil dari `--timeout`, menyebabkan job dobel.
- Menjalankan worker sebagai root sehingga izin file di `storage/` kacau.


## Checklist

1. Cek antrean (`queue:monitor` atau tabel `jobs`), worker (`supervisorctl status`), dan `queue:failed`.
2. Pastikan `QUEUE_CONNECTION` bukan `sync` dan tabel `jobs` sudah ada.
3. Pastikan job mengimplementasikan `ShouldQueue`.
4. Samakan nama koneksi dan antrean antara dispatch dan worker.
5. Jalankan worker lewat Supervisor dan tambahkan `queue:restart` ke skrip deploy.
6. Atur `retry_after` > `--timeout`.
7. Baca kolom `exception` di `failed_jobs` sebelum retry.


Error Laravel lain yang sering ditemui ada di [kumpulan solusi error Laravel](https://dhiecoderweb.com/kumpulan-solusi-error-laravel-paling-sering-terjadi).

## FAQ

### Kenapa job berjalan di lokal tetapi tidak di server?

Biasanya karena di lokal `QUEUE_CONNECTION=sync` sehingga job langsung dijalankan, sedangkan di server memakai `database` atau `redis` tetapi tidak ada worker yang berjalan.

### Berapa jumlah worker yang sebaiknya dijalankan?

Mulai dari 1–2 worker, lalu tambah berdasarkan panjang antrean dan beban server. Setiap worker memakai memori dan satu koneksi database, jadi jangan berlebihan.

### Apakah perlu Laravel Horizon?

Horizon berguna jika Anda memakai Redis dan butuh dashboard serta pengaturan worker yang lebih rapi. Untuk driver database atau aplikasi kecil, Supervisor dengan `queue:work` sudah cukup.

### Bagaimana menghapus job yang menumpuk di antrean?

Gunakan `php artisan queue:clear` (opsional dengan nama koneksi dan `--queue=`). Hati-hati, job yang dihapus tidak akan dijalankan, jadi pastikan Anda memang tidak membutuhkannya.


---

<!-- Aktifkan tautan ini setelah artikel terbit di blog:
Artikel lengkap: https://dhiecoderweb.com/queue-laravel-tidak-jalan-macet-solusi
-->

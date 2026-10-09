# MySQL Too Many Connections (1040) di Laravel: Solusi

```
SQLSTATE[HY000] [1040] Too many connections

# varian batas per user (umum di shared hosting)
SQLSTATE[HY000] [1203] User db_user already has more than 'max_user_connections' active connections
SQLSTATE[42000] [1226] User 'db_user' has exceeded the 'max_user_connections' resource (current value: 10)
```


Error `Too many connections` (kode 1040) berarti jumlah koneksi aktif ke server MySQL sudah mencapai batas `max_connections`, sehingga koneksi baru ditolak. Varian `max_user_connections` (kode 1203 atau 1226) artinya batas yang tercapai adalah batas **per user database**, bukan batas server. Varian ini sangat umum di shared hosting.

Menaikkan batas bukan selalu solusi yang tepat. Lebih sering, masalahnya adalah koneksi yang tidak dilepas: koneksi persisten, query lambat yang menumpuk, terlalu banyak worker, atau proses cron yang saling bertumpuk. Artikel ini membahas cara mengecek kondisi sebenarnya, penyebab dari yang paling mungkin, dan perbaikannya untuk VPS maupun shared hosting.

## Bedanya 1040, 1203, dan 1226  KodeBatas yang tercapaiSiapa yang bisa mengubah  1040 Too many connections`max_connections` global serverAdmin server (VPS/dedicated) 1203 ... max_user_connections active connectionsVariabel global `max_user_connections` (berlaku untuk tiap akun)Admin server 1226 ... exceeded the 'max_user_connections' resourceBatas khusus akun, diset lewat `CREATE/ALTER USER ... WITH MAX_USER_CONNECTIONS`Admin server

Di shared hosting, Anda tidak bisa mengubah ketiganya. Yang bisa Anda lakukan adalah mengurangi jumlah koneksi yang dipakai aplikasi secara bersamaan.

## Langkah pertama: lihat apa yang sebenarnya terjadi

Jangan menebak. Login ke MySQL (jika semua slot penuh, lihat tips di bawah) dan jalankan:

```
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE 'max_user_connections';
SHOW VARIABLES LIKE 'wait_timeout';

SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
SHOW GLOBAL STATUS LIKE 'Aborted_connects';
```


`Threads_connected` adalah jumlah koneksi saat ini, sedangkan `Max_used_connections` adalah puncak tertinggi sejak server terakhir di-restart. Jika `Max_used_connections` sama dengan `max_connections`, server memang pernah benar-benar penuh. Nilai default `max_connections` di MySQL 8 adalah 151, tetapi distribusi dan panel hosting sering mengubahnya, jadi selalu lihat nilai aktualnya.

Berikutnya, lihat isi koneksi:

```
SHOW FULL PROCESSLIST;

-- ringkasan per user dan status
SELECT user, host, command, COUNT(*) AS jumlah,
       MAX(time) AS detik_terlama
FROM information_schema.processlist
GROUP BY user, host, command
ORDER BY jumlah DESC;
```


Cara membaca hasilnya:

- **Banyak baris `Sleep` dengan `Time` besar** → koneksi dibuka tapi tidak dipakai dan tidak ditutup. Curigai koneksi persisten atau proses yang berjalan lama.
- **Banyak baris `Query` dengan state seperti `Waiting for table metadata lock` atau `Sending data`** → query lambat atau saling mengunci sehingga koneksi menumpuk.
- **Satu user mendominasi** → aplikasi tertentu (atau worker tertentu) adalah sumber masalah.


### Tidak bisa login karena koneksi penuh?

MySQL menyediakan satu slot cadangan di atas `max_connections` untuk akun dengan hak `CONNECTION_ADMIN` (atau `SUPER` di versi lama). Login dengan root lewat `sudo mysql` biasanya masih bisa. Sejak MySQL 8.0.14, Anda juga dapat mengaktifkan antarmuka admin terpisah melalui opsi `admin_address` (port default 33062) yang tidak terkena batas koneksi biasa.

## Penyebab #1: Koneksi persisten (PDO::ATTR_PERSISTENT)

Secara default, Laravel membuka satu koneksi per request dan menutupnya ketika request selesai. Namun jika di `config/database.php` ada opsi berikut, koneksi tidak ditutup dan disimpan oleh proses PHP-FPM:

```
'mysql' => [
    // ...
    'options' => extension_loaded('pdo_mysql') ? array_filter([
        PDO::ATTR_PERSISTENT => true,   // penyebab koneksi menumpuk
        PDO::MYSQL_ATTR_SSL_CA => env('MYSQL_ATTR_SSL_CA'),
    ]) : [],
],
```


Dengan koneksi persisten, setiap proses PHP-FPM menyimpan koneksinya sendiri walaupun sedang menganggur. Jika Anda punya beberapa pool PHP-FPM dengan total `pm.max_children` besar, jumlah koneksi bisa mendekati total tersebut. Untuk aplikasi Laravel biasa, hapus baris `PDO::ATTR_PERSISTENT`. Di shared hosting, koneksi persisten hampir selalu menjadi penyebab error `max_user_connections`.

## Penyebab #2: Query lambat dan lock yang menumpuk

Satu query lambat di jam ramai bisa memicu efek berantai: request baru terus datang, masing-masing membuka koneksi dan menunggu, sampai batas habis. Tanda-tandanya terlihat di processlist: banyak koneksi dengan status `Query` dan waktu yang terus bertambah.

Langkah perbaikan:

1. Aktifkan slow query log untuk menemukan query bermasalah:

```
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SHOW VARIABLES LIKE 'slow_query_log_file';
```

2. Analisis query dengan `EXPLAIN` dan tambahkan index pada kolom yang dipakai di `WHERE`, `JOIN`, dan `ORDER BY`.
3. Hindari query N+1 di Laravel dengan eager loading (`with()`), dan gunakan `chunkById()` untuk memproses data besar.
4. Cache hasil query yang mahal dan jarang berubah dengan `Cache::remember()`.


Jika perlu menghentikan query yang macet saat darurat, gunakan `KILL <id>;` berdasarkan kolom `Id` di processlist. Lakukan dengan hati-hati karena transaksi yang dihentikan akan di-rollback.

## Penyebab #3: Terlalu banyak worker dan proses

Hitung semua proses yang bisa membuka koneksi pada saat bersamaan, bukan hanya request web:

- Total `pm.max_children` dari semua pool PHP-FPM yang terhubung ke database ini.
- Jumlah queue worker (`numprocs` di Supervisor, atau proses Horizon). Setiap worker menahan koneksinya selama berjalan.
- Proses scheduler dan cron, terutama yang bisa berjalan bertumpuk.
- Aplikasi lain, panel admin, atau tool backup yang memakai server MySQL yang sama.


Jika jumlah tersebut melebihi `max_connections`, error ini hanya soal waktu. Solusinya bisa berupa menurunkan `pm.max_children` agar sesuai kapasitas, mengurangi jumlah worker, atau menaikkan `max_connections` bila RAM server mencukupi.

Untuk tugas terjadwal di Laravel, cegah eksekusi bertumpuk:

```
// routes/console.php (Laravel 11) atau app/Console/Kernel.php
Schedule::command('report:generate')
    ->everyFiveMinutes()
    ->withoutOverlapping();
```


Pada skrip atau command yang berjalan lama dan jarang memakai database, lepaskan koneksi setelah selesai dipakai dengan `DB::disconnect();`. Laravel akan membuka ulang koneksi otomatis bila dibutuhkan lagi.

## Penyebab #4: Koneksi Sleep tidak pernah ditutup (wait_timeout terlalu lama)

Nilai default `wait_timeout` adalah 28800 detik (8 jam). Koneksi yang ditinggalkan tanpa ditutup dengan benar, misalnya dari skrip yang crash atau tool eksternal, akan tetap menempati slot sampai waktu itu habis. Di VPS, Anda bisa menurunkannya:

```
SET GLOBAL wait_timeout = 300;
SET GLOBAL interactive_timeout = 300;
```


Jangan menurunkan terlalu rendah jika Anda memakai queue worker yang lama menganggur. Laravel akan mencoba menyambung ulang ketika mendeteksi "MySQL server has gone away" atau "Lost connection", tetapi nilai yang terlalu kecil tetap menambah beban sambung-ulang.

## Perbaikan di VPS: menaikkan max_connections dengan benar

Setelah penyebab di atas dibereskan dan Anda yakin memang butuh lebih banyak koneksi, naikkan batasnya. Perubahan sementara (hilang saat restart):

```
SET GLOBAL max_connections = 300;
```


Perubahan permanen di MySQL 8 bisa memakai `SET PERSIST`, atau tulis di file konfigurasi, misalnya `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```
[mysqld]
max_connections = 300
wait_timeout    = 300
```


```
sudo systemctl restart mysql
```


MariaDB tidak mendukung `SET PERSIST`, jadi gunakan file konfigurasi. Setiap koneksi memakai memori (buffer per thread seperti `sort_buffer_size` dan `join_buffer_size`), sehingga menaikkan batas terlalu tinggi di server dengan RAM kecil justru berisiko membuat MySQL dimatikan oleh OOM killer. Naikkan bertahap dan pantau penggunaan memori.

## Perbaikan di shared hosting

Di shared hosting, batas `max_user_connections` ditentukan penyedia dan biasanya kecil. Langkah yang bisa Anda lakukan:

1. Pastikan tidak ada `PDO::ATTR_PERSISTENT` di konfigurasi.
2. Kurangi query per halaman dan gunakan cache (driver `file` pun cukup membantu).
3. Jangan menjalankan `queue:work` permanen dengan banyak proses; untuk shared hosting, cron dengan `queue:work --stop-when-empty` lebih aman.
4. Periksa log akses: lonjakan dari bot atau crawler agresif sering menjadi pemicu. Batasi dengan rate limit atau aturan di `.htaccess`/firewall hosting.
5. Jika aplikasi memang sudah ramai, tanyakan batas pastinya ke penyedia, lalu pertimbangkan upgrade paket atau pindah ke VPS. Panduan [memilih domain dan hosting](https://dhiecoderweb.com/cara-memilih-domain-dan-hosting-untuk-website) bisa membantu menentukan pilihan.


## Kesalahan umum

- Langsung menaikkan `max_connections` ke angka sangat besar tanpa menghitung RAM.
- Mengaktifkan koneksi persisten karena mengira pasti lebih cepat.
- Menambah jumlah queue worker tanpa memperhitungkan koneksi database.
- Restart MySQL berulang kali tanpa melihat processlist, sehingga penyebab tidak pernah ditemukan.
- Membiarkan cron berjalan bertumpuk karena tidak memakai `withoutOverlapping()`.


## Checklist

1. Bedakan 1040 (batas server) dan 1203/1226 (batas per user).
2. Cek `Threads_connected`, `Max_used_connections`, dan `SHOW FULL PROCESSLIST`.
3. Hapus `PDO::ATTR_PERSISTENT` jika ada.
4. Cari dan perbaiki query lambat (slow query log, `EXPLAIN`, index).
5. Hitung PHP-FPM children + worker + cron; samakan dengan kapasitas.
6. Turunkan `wait_timeout` secukupnya di VPS.
7. Baru naikkan `max_connections` bertahap sambil memantau RAM.


Untuk memperdalam dasar MySQL seperti index dan user, baca [panduan belajar MySQL untuk pemula](https://dhiecoderweb.com/panduan-belajar-mysql-untuk-pemula-dari-dasar-sampai-mahir). Konfigurasi PHP-FPM dan MySQL di server dibahas di [deploy Laravel ke VPS Ubuntu dengan Nginx](https://dhiecoderweb.com/cara-deploy-laravel-ke-vps-ubuntu-nginx), dan error Laravel lainnya ada di [kumpulan solusi error Laravel](https://dhiecoderweb.com/kumpulan-solusi-error-laravel-paling-sering-terjadi).

## FAQ

### Berapa nilai max_connections yang ideal?

Tidak ada angka universal. Hitung jumlah proses yang bisa terhubung bersamaan (PHP-FPM, worker, cron), tambahkan cadangan, lalu pastikan RAM server cukup untuk jumlah tersebut. Pantau `Max_used_connections` selama beberapa hari sebagai acuan.

### Apakah restart MySQL menyelesaikan masalah?

Hanya sementara. Restart memutus semua koneksi, tetapi jika penyebabnya (query lambat, koneksi persisten, worker berlebih) tidak diperbaiki, error akan kembali.

### Apakah queue worker Laravel selalu memegang koneksi database?

Ya, worker yang sudah memakai database biasanya mempertahankan koneksinya selama proses berjalan. Itu normal, tetapi jumlah worker harus diperhitungkan dalam kapasitas koneksi.

### Saya di shared hosting dan tidak bisa menjalankan SHOW GLOBAL STATUS. Apa yang bisa dicek?

Anda biasanya masih bisa menjalankan `SHOW PROCESSLIST` untuk melihat koneksi milik user Anda sendiri dan `SHOW VARIABLES LIKE 'max_user_connections'`. Selebihnya, fokus mengurangi koneksi dari sisi aplikasi dan minta data dari penyedia hosting.


---

<!-- Aktifkan tautan ini setelah artikel terbit di blog:
Artikel lengkap: https://dhiecoderweb.com/mysql-too-many-connections-1040-laravel-solusi
-->

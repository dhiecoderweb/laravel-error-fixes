# SQLSTATE[HY000] [2002] Connection Refused Laravel: Solusi

```
SQLSTATE[HY000] [2002] Connection refused (Connection: mysql, SQL: select * from `users` where `email` = admin@example.com limit 1)
```


Error `SQLSTATE[HY000] [2002] Connection refused` berarti PHP berhasil mencoba membuka koneksi TCP ke alamat dan port database yang Anda tulis di `.env`, tetapi tidak ada proses yang mendengarkan di sana. Penyebab paling umum: MySQL belum berjalan, `DB_HOST` atau `DB_PORT` salah, atau aplikasi berjalan di dalam Docker sehingga `127.0.0.1` menunjuk ke container itu sendiri, bukan ke server database.

Variannya, `SQLSTATE[HY000] [2002] No such file or directory`, muncul ketika koneksi dicoba lewat *Unix socket* dan file socket-nya tidak ditemukan. Artikel ini membahas kedua varian tersebut, diurutkan dari penyebab yang paling sering terjadi, lengkap dengan perintah untuk memeriksa dan memperbaikinya.

## Memahami pesan error-nya terlebih dahulu

Kode `2002` berasal dari client MySQL (driver `mysqlnd` di PHP), bukan dari server MySQL. Artinya, permintaan Anda bahkan belum sampai ke tahap autentikasi. Ini penting untuk membedakan dengan error lain:

- **[2002] Connection refused** — koneksi TCP ditolak. Host bisa dijangkau, tetapi port tujuan tertutup atau tidak ada service yang mendengarkan.
- **[2002] No such file or directory** — PHP mencoba memakai Unix socket (biasanya karena `DB_HOST=localhost`), tetapi file socket tidak ada di path yang dicari.
- **[2002] Connection timed out** — paket tidak dibalas sama sekali, biasanya firewall atau alamat IP yang salah.
- **[1045] Access denied for user** — koneksi sudah berhasil, tetapi username atau password salah. Kalau Anda melihat error ini, masalah jaringan sudah selesai.


Sejak Laravel 10, pesan error ikut menampilkan `(Connection: mysql, SQL: ...)`. Bagian `Connection` memberi tahu koneksi mana yang dipakai, sangat berguna kalau aplikasi Anda punya lebih dari satu koneksi database.

## Penyebab #1: MySQL atau MariaDB tidak berjalan

Ini penyebab paling sering di laptop pengembang: komputer baru dinyalakan ulang dan service database belum aktif, atau XAMPP/Laragon/MAMP belum dijalankan. Periksa statusnya sesuai sistem operasi Anda.

```
# Ubuntu/Debian (systemd)
sudo systemctl status mysql
sudo systemctl start mysql

# MariaDB
sudo systemctl status mariadb

# macOS dengan Homebrew
brew services list
brew services start mysql
```


Untuk memastikan ada proses yang benar-benar mendengarkan di port 3306, gunakan `ss` atau `lsof`:

```
# Linux
sudo ss -ltnp | grep 3306

# macOS
lsof -nP -iTCP:3306 -sTCP:LISTEN
```


Kalau perintah tersebut tidak mengeluarkan apa pun, berarti memang tidak ada server MySQL yang mendengarkan di port itu. Jika service gagal start, lihat log-nya dengan `sudo journalctl -u mysql -n 50` atau buka file `/var/log/mysql/error.log`. Penyebab yang sering muncul di log adalah disk penuh, memori tidak cukup, atau file konfigurasi yang rusak setelah diedit.

## Penyebab #2: DB_HOST memakai localhost, padahal seharusnya 127.0.0.1 (atau sebaliknya)

Bagian ini paling sering membingungkan. Driver MySQL di PHP memperlakukan `localhost` secara khusus: ketika host bernilai `localhost`, driver **tidak** memakai TCP, melainkan Unix socket. Sementara `127.0.0.1` selalu memakai TCP ke port yang ditentukan.  Nilai DB_HOSTJenis koneksiError khas bila gagal  `localhost`Unix socket[2002] No such file or directory `127.0.0.1`TCP ke DB_PORT[2002] Connection refused Nama service Docker (mis. `mysql`)TCP di jaringan Docker[2002] Connection refused / php_network_getaddresses

Jadi, kalau Anda melihat *No such file or directory*, solusi tercepat biasanya mengganti `DB_HOST=localhost` menjadi `DB_HOST=127.0.0.1`. Sebaliknya, kalau MySQL dikonfigurasi hanya menerima socket (opsi `skip-networking` aktif), maka TCP akan ditolak dan Anda perlu memakai socket.

### Menentukan path socket secara eksplisit

Laravel menyediakan variabel `DB_SOCKET` yang dipetakan ke opsi `unix_socket` di `config/database.php`. Cari dulu lokasi socket yang sebenarnya:

```
mysql -u root -p -e "SHOW VARIABLES LIKE 'socket';"

# atau cek konfigurasi PHP
php -i | grep -i "default_socket"
```


Lalu isi di `.env`:

```
DB_CONNECTION=mysql
DB_HOST=localhost
DB_SOCKET=/var/run/mysqld/mysqld.sock
DB_PORT=3306
```


Path socket berbeda di tiap instalasi. Di Ubuntu umumnya `/var/run/mysqld/mysqld.sock`, di Homebrew sering `/tmp/mysql.sock`, dan MAMP memakai path di dalam folder aplikasinya sendiri. Jangan menebak; selalu ambil dari output perintah di atas.

## Penyebab #3: Aplikasi berjalan di Docker atau Laravel Sail

Di dalam container, `127.0.0.1` dan `localhost` menunjuk ke container itu sendiri. Container PHP tidak menjalankan MySQL, sehingga koneksi pasti ditolak. Container lain dalam satu jaringan Docker Compose harus dihubungi memakai **nama service**.

Di Laravel Sail, service database bernama `mysql` (atau `mariadb`/`pgsql` tergantung pilihan Anda saat instalasi). Isi `.env` menjadi:

```
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=sail
DB_PASSWORD=password
```


Hal yang sering menjebak: setelah mengganti `DB_HOST=mysql`, perintah `php artisan migrate` yang dijalankan **dari laptop** (bukan dari container) justru gagal, karena nama `mysql` tidak dikenal di luar jaringan Docker. Jalankan perintah artisan lewat container:

```
./vendor/bin/sail artisan migrate

# atau dengan docker compose biasa
docker compose exec app php artisan migrate
```


Sebaliknya, kalau aplikasi di container perlu mengakses MySQL yang terpasang di komputer host, gunakan `host.docker.internal`. Di Docker Desktop (macOS/Windows) nama ini tersedia otomatis; di Linux Anda perlu menambahkan `extra_hosts: ["host.docker.internal:host-gateway"]` pada service di `docker-compose.yml`. Pastikan juga MySQL di host tidak hanya mendengarkan di `127.0.0.1`.

Periksa juga apakah container database sudah sehat. Container bisa saja dalam status *restarting* karena password root kosong atau volume rusak:

```
docker compose ps
docker compose logs mysql --tail=50
```


## Penyebab #4: Port tidak sesuai

Port default MySQL adalah 3306, tetapi tidak semua lingkungan memakainya. MAMP di macOS secara default memakai port 8889 untuk MySQL. Di Docker, Anda mungkin memetakan port host berbeda, misalnya `"3307:3306"`, sehingga dari luar container port-nya 3307, sementara dari dalam jaringan Docker tetap 3306.

Cara memastikan port yang dipakai server:

```
mysql -u root -p -e "SHOW VARIABLES LIKE 'port';"
```


Uji juga koneksi langsung dengan client MySQL memakai parameter yang sama persis dengan `.env`. Jika client ini gagal, masalahnya ada di server/jaringan, bukan di Laravel:

```
mysql -h 127.0.0.1 -P 3306 -u nama_user -p nama_database
```


## Penyebab #5: Konfigurasi Laravel ter-cache

Anda sudah mengubah `.env`, tetapi error tetap sama persis? Kemungkinan besar konfigurasi sudah di-cache dengan `php artisan config:cache` (atau `php artisan optimize`) sebelumnya. Saat cache aktif, Laravel tidak membaca `.env` sama sekali; ia memakai file `bootstrap/cache/config.php`.

```
php artisan config:clear

# di produksi, buat ulang cache setelah .env benar
php artisan config:cache
```


Kalau Anda memakai PHP-FPM dan OPcache dengan `opcache.validate_timestamps=0`, restart juga PHP-FPM agar file cache yang baru terbaca, misalnya `sudo systemctl restart php8.3-fpm` (sesuaikan versi PHP Anda). Untuk queue worker yang berjalan lama, jalankan `php artisan queue:restart` karena worker menyimpan konfigurasi lama di memori.

## Penyebab #6: MySQL hanya mendengarkan di alamat tertentu (server terpisah)

Jika database berada di server lain, MySQL secara default di banyak distribusi hanya mendengarkan di `127.0.0.1` melalui opsi `bind-address`. Koneksi dari server aplikasi akan ditolak. Periksa file konfigurasi, di Ubuntu biasanya `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```
[mysqld]
bind-address = 0.0.0.0
# atau lebih aman: IP privat server database
# bind-address = 10.0.0.5
```


Setelah mengubah, restart MySQL dan pastikan firewall (`ufw`, security group cloud) hanya membuka port 3306 untuk IP server aplikasi, bukan untuk seluruh internet. User MySQL juga harus diizinkan dari host tersebut, misalnya `'appuser'@'10.0.0.%'`; jika belum, Anda akan berpindah ke error 1045 atau 1130, yang berarti masalah jaringan sudah teratasi.

## Kasus khusus: Laravel 11 dan SQLite default

Proyek Laravel 11 baru memakai `DB_CONNECTION=sqlite` secara default dan baris `DB_HOST` dkk. dikomentari di `.env`. Saat Anda beralih ke MySQL, pastikan baris-baris itu benar-benar di-uncomment. Kalau `DB_HOST` masih dikomentari, Laravel memakai nilai default dari `config/database.php`, yaitu `127.0.0.1`, yang mungkin bukan host yang Anda maksud.

## Kesalahan umum saat memperbaiki error 2002

- **Mengedit .env tetapi lupa `config:clear`.** Perubahan tidak berpengaruh sama sekali selama cache aktif.
- **Menjalankan artisan dari host padahal DB_HOST berisi nama service Docker.** Hasilnya error, padahal aplikasi di browser berjalan normal.
- **Mengira error 2002 adalah masalah password.** Password salah menghasilkan error 1045, bukan 2002.
- **Membuka port 3306 ke publik** hanya agar koneksi berhasil. Batasi dengan firewall ke IP tertentu.
- **Ada spasi atau tanda kutip yang salah di `.env`**, misalnya `DB_HOST= 127.0.0.1`. Tulis nilai tanpa spasi tambahan.


## Checklist singkat

1. Pastikan MySQL berjalan: `systemctl status mysql` atau `docker compose ps`.
2. Pastikan ada yang mendengarkan di port: `ss -ltnp | grep 3306`.
3. Gunakan `127.0.0.1` untuk TCP, atau isi `DB_SOCKET` bila memakai `localhost`.
4. Di Docker/Sail, pakai nama service (`mysql`) dan jalankan artisan dari dalam container.
5. Cocokkan `DB_PORT` dengan output `SHOW VARIABLES LIKE 'port'`.
6. Uji dengan client `mysql -h ... -P ...` memakai parameter yang sama.
7. Jalankan `php artisan config:clear` lalu coba lagi.


Untuk daftar error Laravel lain yang sering muncul, lihat [kumpulan solusi error Laravel paling sering terjadi](https://dhiecoderweb.com/kumpulan-solusi-error-laravel-paling-sering-terjadi). Jika Anda baru menyiapkan server, panduan [deploy Laravel ke VPS Ubuntu dengan Nginx](https://dhiecoderweb.com/cara-deploy-laravel-ke-vps-ubuntu-nginx) membahas instalasi MySQL dari awal, dan [panduan belajar MySQL untuk pemula](https://dhiecoderweb.com/panduan-belajar-mysql-untuk-pemula-dari-dasar-sampai-mahir) membantu memahami user dan hak akses.

## FAQ

### Apa beda error "Connection refused" dan "No such file or directory"?

"Connection refused" muncul pada koneksi TCP (biasanya `DB_HOST=127.0.0.1` atau IP lain) ketika tidak ada service di port tujuan. "No such file or directory" muncul pada koneksi Unix socket (biasanya `DB_HOST=localhost`) ketika file socket tidak ditemukan.

### Kenapa di browser aplikasi jalan, tetapi php artisan migrate gagal?

Biasanya karena web server dan terminal memakai lingkungan berbeda. Di Docker, browser dilayani container yang mengenal host `mysql`, sedangkan terminal di laptop tidak. Bisa juga karena PHP CLI dan PHP-FPM memakai `php.ini` berbeda dengan path socket berbeda.

### Apakah aman mengganti localhost menjadi 127.0.0.1?

Ya, selama MySQL mendengarkan di TCP port tersebut. Perbedaannya hanya mekanisme koneksi. Socket sedikit lebih cepat untuk koneksi lokal, tetapi bedanya jarang terasa di aplikasi web biasa.

### Error masih muncul setelah .env diperbaiki, apa lagi yang harus dicek?

Jalankan `php artisan config:clear`, restart PHP-FPM, dan jalankan `php artisan queue:restart` jika ada worker. Lalu pastikan file `.env` yang Anda edit memang file di direktori proyek yang aktif, bukan salinan di folder lain.

### Bagaimana di shared hosting?

Di shared hosting cPanel, umumnya `DB_HOST=localhost` sudah benar karena MySQL berada di server yang sama. Jika muncul error 2002, kemungkinan besar service MySQL hosting sedang bermasalah; hubungi penyedia hosting. Panduan lengkapnya ada di artikel [deploy Laravel ke shared hosting cPanel](https://dhiecoderweb.com/cara-deploy-laravel-ke-shared-hosting-cpanel).


---

<!-- Aktifkan tautan ini setelah artikel terbit di blog:
Artikel lengkap: https://dhiecoderweb.com/cara-mengatasi-sqlstate-hy000-2002-connection-refused-laravel
-->

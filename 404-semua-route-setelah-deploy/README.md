# Laravel 404 Not Found di Semua Route Setelah Deploy

Kalau setelah deploy hanya halaman utama Laravel yang terbuka, sementara `/login`, `/about`, atau route lain langsung menampilkan 404, hampir pasti masalahnya ada di web server, bukan di kode Anda. Request ke URL selain `/` tidak pernah diteruskan ke `public/index.php`, sehingga router Laravel tidak sempat bekerja. Solusinya: aktifkan rewrite (Apache `mod_rewrite` + `.htaccess` + `AllowOverride All`, atau `try_files` di Nginx) dan pastikan document root mengarah ke folder `public`.

Gejalanya biasanya terlihat seperti ini di browser:

```
Not Found

The requested URL was not found on this server.

Apache/2.4.58 (Ubuntu) Server at contoh.com Port 80
```


atau, di Nginx:

```
404 Not Found
nginx/1.24.0
```


Artikel ini membahas penyebabnya dari yang paling sering terjadi sampai yang paling jarang, lengkap dengan perintah dan konfigurasi yang bisa langsung Anda pakai.

## Langkah 0: Pastikan siapa yang menampilkan 404

Sebelum mengubah apa pun, lihat dulu halaman 404 yang muncul. Ini diagnosis paling penting dan paling cepat.

- **Halaman 404 polos dari Apache/Nginx/LiteSpeed** (ada tulisan nama server di bawahnya) berarti request berhenti di web server. Laravel tidak pernah menerima request tersebut. Fokus ke rewrite dan document root.
- **Halaman 404 bergaya Laravel** (tulisan "404 | Not Found" di tengah layar, atau halaman error kustom Anda) berarti request sudah sampai ke Laravel, tetapi router tidak menemukan route yang cocok. Fokus ke route cache, prefix, atau nama file route.


Tes kedua yang sangat berguna: buka URL dengan menyisipkan `index.php`, misalnya `https://contoh.com/index.php/login`. Jika halaman ini terbuka normal sementara `https://contoh.com/login` tetap 404, Anda sudah memastikan bahwa Laravel sehat dan yang rusak adalah aturan rewrite.

## Penyebab 1: Apache tidak membaca .htaccess (paling sering)

Laravel menyertakan file `public/.htaccess` yang tugasnya meneruskan semua request yang bukan file atau folder nyata ke `index.php`. Di server Apache, file ini hanya bekerja jika dua syarat terpenuhi: modul `mod_rewrite` aktif, dan konfigurasi virtual host mengizinkan `.htaccess` menimpa pengaturan lewat `AllowOverride`.

### Aktifkan mod_rewrite

Di Ubuntu/Debian dengan akses root, jalankan:

```
sudo a2enmod rewrite
sudo systemctl restart apache2

# cek apakah modul sudah aktif
apache2ctl -M | grep rewrite
# output yang diharapkan: rewrite_module (shared)
```


Di CentOS/AlmaLinux/Rocky, modul ini biasanya sudah dimuat lewat `/etc/httpd/conf.modules.d/00-base.conf`. Pastikan baris `LoadModule rewrite_module modules/mod_rewrite.so` tidak dikomentari.

### Ubah AllowOverride None menjadi All

Ini jebakan klasik. Konfigurasi bawaan Apache di Ubuntu memasang `AllowOverride None` untuk `/var/www/`, yang artinya Apache mengabaikan seluruh isi `.htaccess`. Tidak ada error, tidak ada peringatan, aturan rewrite Anda diam-diam tidak dibaca. Buka file virtual host Anda dan tambahkan blok `Directory` seperti ini:

```
<VirtualHost *:80>
    ServerName contoh.com
    DocumentRoot /var/www/contoh/public

    <Directory /var/www/contoh/public>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/contoh-error.log
    CustomLog ${APACHE_LOG_DIR}/contoh-access.log combined
</VirtualHost>
```


Setelah itu uji sintaks dan muat ulang Apache:

```
sudo apache2ctl configtest
sudo systemctl reload apache2
```


Jika Anda memakai HTTPS, jangan lupa bahwa ada virtual host terpisah untuk port 443 (biasanya dibuat Certbot dengan akhiran `-le-ssl.conf`). Blok `Directory` yang sama harus berlaku untuk virtual host tersebut, atau diletakkan di `apache2.conf` agar berlaku global.

### Pastikan file .htaccess benar-benar ada

File yang namanya diawali titik bersifat tersembunyi. Banyak klien FTP dan File Manager tidak menampilkan atau tidak ikut mengunggahnya, begitu juga perintah `cp *` atau arsip zip yang dibuat dengan pola tertentu. Cek langsung di server:

```
ls -la /var/www/contoh/public/
# harus ada: .htaccess  index.php  favicon.ico  robots.txt ...
```


Jika tidak ada, salin isi `public/.htaccess` dari proyek lokal Anda atau dari repositori resmi `laravel/laravel`. Bagian terpentingnya adalah aturan berikut:

```
<IfModule mod_rewrite.c>
    RewriteEngine On

    # Redirect Trailing Slashes If Not A Folder...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} (.+)/$
    RewriteRule ^ %1 [L,R=301]

    # Send Requests To Front Controller...
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteRule ^ index.php [L]
</IfModule>
```


Perhatikan bahwa semuanya dibungkus `<IfModule mod_rewrite.c>`. Karena itu, jika `mod_rewrite` mati, Apache tidak memunculkan error 500, melainkan diam-diam melewati seluruh aturan. Hasilnya persis gejala yang Anda lihat: homepage jalan (karena `DirectoryIndex` otomatis memanggil `index.php`), tetapi route lain 404.

## Penyebab 2: Nginx tanpa try_files yang benar

Nginx tidak membaca `.htaccess` sama sekali. Mengunggah file tersebut ke server Nginx tidak berpengaruh apa pun. Aturan rewrite harus ditulis di blok `server`. Konfigurasi minimal yang benar untuk Laravel:

```
server {
    listen 80;
    server_name contoh.com;
    root /var/www/contoh/public;

    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```


Kesalahan yang sering terjadi:

- **`try_files $uri $uri/ =404;`** — ini konfigurasi bawaan file `default` Nginx. Baris ini menyuruh Nginx mengembalikan 404 jika file tidak ditemukan, padahal Laravel butuh fallback ke `/index.php`.
- **Lupa `?$query_string`** — route tetap jalan, tetapi parameter query seperti `?page=2` hilang.
- **Versi socket PHP-FPM salah** — kalau path socket tidak cocok, gejalanya biasanya 502 Bad Gateway, bukan 404. Cek dengan `ls /var/run/php/`.


Selalu uji dan muat ulang setelah mengubah konfigurasi:

```
sudo nginx -t
sudo systemctl reload nginx
```


Panduan konfigurasi Nginx yang lebih lengkap ada di artikel [cara deploy Laravel ke VPS Ubuntu dengan Nginx](https://dhiecoderweb.com/cara-deploy-laravel-ke-vps-ubuntu-nginx).

## Penyebab 3: Document root tidak mengarah ke folder public

Laravel dirancang agar hanya folder `public` yang terekspos ke internet. Jika document root mengarah ke folder proyek (misalnya `/var/www/contoh` atau `public_html` berisi seluruh proyek), beberapa hal terjadi sekaligus: homepage mungkin terbuka lewat `/public`, URL lain 404, dan yang lebih berbahaya, file seperti `.env` berpotensi bisa diunduh siapa saja.

Di VPS, perbaiki `DocumentRoot` (Apache) atau `root` (Nginx) agar berakhir dengan `/public`, seperti contoh di atas.

Di shared hosting cPanel, ada dua pendekatan yang aman:

1. **Ubah document root domain** lewat menu Domains di cPanel agar menunjuk ke `nama-proyek/public`. Ini cara paling bersih jika hosting Anda mengizinkannya.
2. **Letakkan proyek di luar `public_html`**, lalu pindahkan isi folder `public` (termasuk `.htaccess`) ke `public_html` dan sesuaikan path di `index.php`.


Untuk pendekatan kedua, dua baris di `public_html/index.php` perlu diubah agar menunjuk ke lokasi proyek yang baru:

```
// Laravel 11
require __DIR__.'/../laravel-app/vendor/autoload.php';
$app = require_once __DIR__.'/../laravel-app/bootstrap/app.php';

// Pastikan juga baris maintenance mode ikut disesuaikan:
if (file_exists($maintenance = __DIR__.'/../laravel-app/storage/framework/maintenance.php')) {
    require $maintenance;
}
```


Langkah lengkapnya dibahas di [cara deploy Laravel ke shared hosting cPanel](https://dhiecoderweb.com/cara-deploy-laravel-ke-shared-hosting-cpanel). Hindari trik menaruh `.htaccess` di root proyek yang me-rewrite semua request ke `public/` tanpa memblokir akses ke file sensitif, karena celah keamanannya mudah terlewat.

## Penyebab 4: Route cache basi

Jika halaman 404 yang muncul adalah halaman Laravel, bukan halaman web server, curigai route cache. Perintah `php artisan route:cache` menyimpan daftar route ke `bootstrap/cache/routes-v7.php`. Selama file itu ada, Laravel tidak membaca ulang `routes/web.php`. Kalau Anda menambah route lalu deploy tanpa membangun ulang cache, route baru tidak dikenal.

```
php artisan route:clear
php artisan route:list --path=login   # pastikan route terdaftar

# setelah yakin semua benar, bangun ulang cache untuk produksi
php artisan optimize:clear
php artisan optimize
```


Hal yang sama berlaku jika file cache ikut ter-commit atau ter-upload dari mesin lokal. Pastikan folder `bootstrap/cache` hanya berisi `.gitignore` di repositori, dan cache dibangun di server sebagai bagian dari skrip deploy.

## Penyebab 5: Penyebab lain yang lebih jarang

### Proyek di subfolder

Jika aplikasi diakses lewat `https://contoh.com/aplikasi`, Apache butuh `Alias` atau document root yang tepat, dan Nginx butuh blok `location /aplikasi` dengan `try_files` yang mengarah ke `/aplikasi/index.php`. Set juga `APP_URL` dengan subfolder agar URL yang dihasilkan helper `url()` dan `route()` benar. Jika memungkinkan, gunakan subdomain karena jauh lebih sederhana.

### Huruf besar-kecil

Windows dan macOS (default) tidak membedakan huruf besar-kecil pada nama file, sedangkan Linux membedakannya. Ini jarang menyebabkan 404 pada route, tetapi sering menyebabkan error "View not found" atau class tidak ditemukan setelah deploy. Jika route menampilkan 404 Laravel padahal ada di `route:list`, cek apakah Anda memakai `abort(404)` atau `findOrFail()` di controller, karena keduanya juga menghasilkan 404 bergaya Laravel.

### LiteSpeed dan OpenLiteSpeed

LiteSpeed Enterprise (banyak dipakai shared hosting) membaca `.htaccess` seperti Apache. OpenLiteSpeed tidak otomatis; Anda perlu mengaktifkan "Rewrite" dan "Auto Load from .htaccess" di pengaturan virtual host lewat WebAdmin, lalu melakukan restart.

## Kesalahan umum saat memperbaiki

- Mengubah file `.htaccess` berkali-kali padahal `AllowOverride` masih `None`. Selama itu belum diubah, isi `.htaccess` apa pun tidak berpengaruh.
- Mengedit virtual host port 80 tetapi situs diakses lewat HTTPS yang memakai virtual host port 443.
- Lupa reload/restart web server setelah mengubah konfigurasi.
- Menyalin `.htaccess` ke root proyek, bukan ke folder yang menjadi document root.
- Menjalankan `route:cache` di lokal lalu mengunggah folder `bootstrap/cache` ke server.
- Mengandalkan cache browser: respons redirect 301 disimpan browser, jadi uji dengan jendela incognito atau `curl -I`.


## Checklist cepat

1. Halaman 404 milik web server atau milik Laravel?
2. Apakah `/index.php/route-anda` terbuka? Jika ya, masalahnya di rewrite.
3. Apache: `a2enmod rewrite`, `AllowOverride All`, `.htaccess` ada di folder public.
4. Nginx: `try_files $uri $uri/ /index.php?$query_string;`
5. Document root berakhir di `/public`.
6. `php artisan optimize:clear` lalu cek `php artisan route:list`.
7. Lihat log: `/var/log/apache2/error.log`, `/var/log/nginx/error.log`, dan `storage/logs/laravel.log`.


Masih menemukan error lain setelah route berjalan? Lihat [kumpulan solusi error Laravel yang paling sering terjadi](https://dhiecoderweb.com/kumpulan-solusi-error-laravel-paling-sering-terjadi).

## FAQ

### Kenapa homepage Laravel bisa terbuka padahal rewrite tidak aktif?

Karena untuk URL `/`, web server memakai pengaturan `DirectoryIndex` (Apache) atau `index` (Nginx) yang otomatis memanggil `index.php`. Laravel lalu menangani route `/` seperti biasa. Untuk URL lain, web server mencari file fisik bernama `login` atau `about`, tidak menemukannya, dan mengembalikan 404 tanpa melibatkan Laravel.

### Saya tidak punya akses root di shared hosting. Apa yang bisa saya lakukan?

Di sebagian besar shared hosting, `mod_rewrite` dan `AllowOverride` sudah aktif. Jadi fokuslah pada dua hal: pastikan `.htaccess` benar-benar terunggah ke folder yang menjadi document root, dan pastikan document root mengarah ke isi folder `public`. Jika tetap gagal, minta dukungan hosting memeriksa apakah rewrite diizinkan untuk akun Anda.

### Apakah route:cache bisa menyebabkan 404?

Bisa, jika cache dibuat sebelum route baru ditambahkan atau dibuat dari kode yang berbeda dengan yang sedang berjalan. Bedanya, 404 yang muncul adalah halaman Laravel. Jalankan `php artisan route:clear` dan periksa ulang.

### Apakah aman menaruh seluruh proyek Laravel di public_html?

Tidak dianjurkan. Jika konfigurasi rewrite gagal, file seperti `.env`, `composer.json`, atau log di `storage` bisa terekspos. Arahkan document root ke folder `public` atau pindahkan proyek ke luar `public_html`.

### Bagaimana cara mengecek tanpa cache browser?

Gunakan `curl -I https://contoh.com/login`. Kode status langsung terlihat, dan tidak ada cache redirect seperti di browser. Untuk melihat isi halamannya, jalankan `curl -s https://contoh.com/login | head -20`: halaman 404 web server biasanya pendek dan menyebut nama server (Apache/nginx), sedangkan halaman 404 Laravel berisi HTML lengkap dari view error Laravel atau view kustom Anda.


---

<!-- Aktifkan tautan ini setelah artikel terbit di blog:
Artikel lengkap: https://dhiecoderweb.com/laravel-404-not-found-semua-route-setelah-deploy
-->

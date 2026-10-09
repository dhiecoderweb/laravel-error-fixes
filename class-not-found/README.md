# Class Not Found di Laravel/PHP: Penyebab dan Solusinya

```
Error: Class "App\Services\PaymentService" not found

# varian yang sering muncul di Laravel
Target class [App\Http\Controllers\Admin\ReportController] does not exist.
```


Error `Class "..." not found` berarti autoloader Composer tidak berhasil menemukan file untuk nama class yang Anda panggil. Hampir selalu penyebabnya salah satu dari ini: namespace tidak cocok dengan lokasi folder (aturan PSR-4), huruf besar-kecil nama file berbeda dengan nama class (dan baru ketahuan di server Linux), autoload Composer belum diperbarui, atau statement `use` yang keliru.

Artikel ini mengurutkan penyebab dari yang paling sering terjadi, beserta cara memeriksa dan perintah untuk memperbaikinya, baik untuk class buatan sendiri maupun class dari package Composer.

## Cara autoload bekerja di Laravel (singkat)

PHP tidak tahu di file mana sebuah class berada. Laravel menyerahkan tugas ini ke Composer melalui aturan **PSR-4** yang tertulis di `composer.json`:

```
"autoload": {
    "psr-4": {
        "App\\": "app/",
        "Database\\Factories\\": "database/factories/",
        "Database\\Seeders\\": "database/seeders/"
    }
}
```


Artinya: class `App\Services\PaymentService` **harus** berada di file `app/Services/PaymentService.php`. Prefix `App\` dipetakan ke folder `app/`, lalu setiap segmen namespace berikutnya menjadi nama folder, dan nama class menjadi nama file ditambah `.php`. Kalau satu huruf saja berbeda, autoloader tidak akan menemukannya.

## Penyebab #1: Namespace tidak cocok dengan path folder

Ini penyebab nomor satu, terutama setelah memindahkan file ke folder lain atau menyalin class dari proyek lain. Misalnya Anda membuat file di `app/Services/Payment/PaymentService.php`, tetapi deklarasi namespace di dalamnya masih:

```
<?php

namespace App\Services;   // salah, seharusnya App\Services\Payment

class PaymentService
{
    // ...
}
```


Perbaiki namespace agar sama persis dengan path folder:

```
<?php

namespace App\Services\Payment;

class PaymentService
{
    // ...
}
```


Periksa juga hal-hal kecil berikut, karena masing-masing bisa memicu error yang sama:

- Nama class di dalam file sama dengan nama file. File `PaymentService.php` berisi `class PaymentService`, bukan `class PaymentServices`.
- Tidak ada salah ketik di keyword: `namespace`, bukan `namspace`.
- Satu file hanya berisi satu class (PSR-4 tidak mendukung beberapa class dalam satu file untuk autoload).
- File diawali `<?php` tanpa spasi atau BOM di depannya.


Untuk menemukan pelanggaran PSR-4 di seluruh proyek sekaligus, jalankan:

```
composer dump-autoload -o
```


Composer versi 2 akan menampilkan peringatan seperti *"Class App\Services\PaymentService located in ./app/Services/Payment/PaymentService.php does not comply with psr-4 autoloading standard. Skipping."* Pesan ini langsung menunjuk file yang bermasalah.

## Penyebab #2: Huruf besar-kecil berbeda (jalan di lokal, gagal di server)

Pola yang sangat khas: aplikasi berjalan normal di laptop Windows atau macOS, tetapi setelah di-deploy ke server Linux muncul *Class not found*. Penyebabnya, sistem file default Windows dan macOS bersifat *case-insensitive*, sedangkan sistem file Linux bersifat *case-sensitive*.

Di laptop, file `app/services/PaymentService.php` (folder huruf kecil) tetap ditemukan saat PHP mencari `app/Services/PaymentService.php`. Di Linux, keduanya dianggap path yang berbeda. Contoh lain yang sering terjadi:

- File bernama `Paymentservice.php`, class bernama `PaymentService`.
- Folder `app/Http/controllers` (huruf kecil) sementara namespace `App\Http\Controllers`.
- Pemanggilan `use App\Models\user;` padahal class-nya `User`.


### Mengganti nama file agar Git ikut mencatat

Masalah tambahan: di sistem file case-insensitive, Git sering tidak mendeteksi perubahan yang hanya mengubah huruf besar-kecil, sehingga perbaikan Anda tidak ikut ter-push. Gunakan `git mv`:

```
# ganti nama folder lewat nama sementara
git mv app/services app/services_tmp
git mv app/services_tmp app/Services

# ganti nama file
git mv app/Services/Paymentservice.php app/Services/PaymentService.php
git commit -m "Perbaiki huruf besar-kecil nama file"
```


Setelah deploy, jalankan `composer dump-autoload -o` di server.

## Penyebab #3: Autoload Composer belum diperbarui

Untuk class baru di dalam folder yang sudah tercakup PSR-4 (misalnya di bawah `app/`), Anda sebenarnya tidak perlu menjalankan apa pun dalam mode autoload standar. Namun ada beberapa kondisi yang membuat class baru tidak terdeteksi:

- **Autoloader dioptimasi atau authoritative.** Jika produksi memakai `composer install --optimize-autoloader` atau `--classmap-authoritative`, Composer membuat classmap statis. Dengan mode *authoritative*, class yang tidak ada di classmap langsung dianggap tidak ada. Class baru baru dikenali setelah `composer dump-autoload` dijalankan ulang.
- **Anda menambah namespace atau folder baru** di luar `app/`, misalnya `Modules\` di folder `modules/`. Folder ini harus didaftarkan di `composer.json`.
- **File helper** berisi fungsi (bukan class) harus didaftarkan di bagian `"files"`.


```
"autoload": {
    "psr-4": {
        "App\\": "app/",
        "Modules\\": "modules/"
    },
    "files": [
        "app/helpers.php"
    ]
}
```


Setiap kali Anda mengubah bagian `autoload` di `composer.json`, jalankan:

```
composer dump-autoload
```


## Penyebab #4: Statement use salah atau tidak ada

Jika Anda memanggil class tanpa `use`, PHP akan mencarinya di namespace file saat ini. Contohnya, di dalam controller dengan namespace `App\Http\Controllers`, pemanggilan `PaymentService::class` tanpa import akan dicari sebagai `App\Http\Controllers\PaymentService`, dan pesan error akan menampilkan nama lengkap tersebut. Itu petunjuk penting: **baca nama class lengkap di pesan error**, lalu bandingkan dengan namespace yang seharusnya.

```
<?php

namespace App\Http\Controllers;

use App\Services\Payment\PaymentService;   // tambahkan import ini

class CheckoutController extends Controller
{
    public function store(PaymentService $payment)
    {
        // ...
    }
}
```


Kesalahan serupa terjadi pada class bawaan PHP. Di dalam file ber-namespace, `new DateTime()` akan dicari sebagai `App\...\DateTime`. Tulis `new \DateTime()` atau tambahkan `use DateTime;`. Hal yang sama berlaku untuk facade seperti `DB` dan `Log`: impor dengan `use Illuminate\Support\Facades\DB;`.

## Penyebab #5: Cache Laravel (config, route, package discovery)

Laravel menyimpan beberapa cache di `bootstrap/cache/`. Jika cache tersebut menyimpan nama class lama, misalnya service provider yang sudah dihapus atau controller yang sudah dipindah, error *Class not found* atau *Target class does not exist* tetap muncul walaupun kode sudah benar.

```
php artisan optimize:clear
```


Perintah ini membersihkan cache config, route, view, event, dan cache aplikasi sekaligus. Jika error terjadi bahkan sebelum artisan bisa berjalan (misalnya karena provider di cache tidak ada), hapus file cache secara manual:

```
rm -f bootstrap/cache/config.php bootstrap/cache/routes-*.php \
      bootstrap/cache/packages.php bootstrap/cache/services.php
composer dump-autoload
php artisan package:discover
```


Jangan menghapus file `bootstrap/cache/.gitignore`. Di server dengan OPcache yang tidak memvalidasi timestamp, restart PHP-FPM setelahnya. Jika ada queue worker, jalankan `php artisan queue:restart`.

## Penyebab #6: Class dari package Composer tidak ditemukan

Jika class yang hilang berasal dari package (misalnya `Barryvdh\DomPDF\Facade\Pdf`), periksa beberapa hal berikut:

1. **Package benar-benar terpasang.** Jalankan `composer show nama-vendor/nama-package`. Jika tidak ada, jalankan `composer require nama-vendor/nama-package`.
2. **Folder `vendor/` di server sudah diperbarui.** Folder `vendor/` tidak di-commit ke Git, sehingga setelah `git pull` Anda wajib menjalankan `composer install` di server.
3. **Package ada di `require-dev`.** Package seperti Laravel Debugbar atau Telescope sering dipasang sebagai dev dependency. Di produksi yang memakai `composer install --no-dev`, package tersebut tidak terpasang. Jika ada kode atau service provider yang tetap mereferensikannya, muncullah *Class not found*. Daftarkan provider-nya secara kondisional atau pindahkan package ke `require`.
4. **Namespace dari dokumentasi package berbeda versi.** Nama class sering berubah antar versi mayor. Cek nama class yang benar langsung di folder `vendor/nama-vendor/nama-package/src`.


```
composer install --no-dev --optimize-autoloader
php artisan package:discover
php artisan optimize:clear
```


## Cara cepat mendiagnosis

Gunakan Tinker untuk menguji apakah autoloader bisa menemukan class tertentu tanpa melewati route atau controller:

```
php artisan tinker
> class_exists(\App\Services\Payment\PaymentService::class)
= true
```


Jika hasilnya `false`, masalahnya pada autoload (namespace, path, huruf besar-kecil, Composer). Jika `true` tetapi aplikasi tetap error, masalahnya pada pemanggilan (statement `use`) atau cache Laravel.

## Kesalahan umum

- Hanya memperbaiki nama file di lokal tanpa `git mv`, sehingga server tetap memakai nama lama.
- Menjalankan `composer dump-autoload` di laptop, padahal masalahnya di server.
- Mengabaikan peringatan "does not comply with psr-4" dari Composer.
- Menyalin controller dari proyek lain tanpa mengganti namespace-nya.
- Meng-commit folder `bootstrap/cache` berisi file cache dari lingkungan lain.


## Checklist

1. Baca nama class lengkap di pesan error.
2. Pastikan namespace = path folder, nama class = nama file (huruf besar-kecil sama persis).
3. Periksa statement `use` dan backslash untuk class global.
4. Jalankan `composer dump-autoload -o` dan baca peringatannya.
5. Jalankan `php artisan optimize:clear`.
6. Untuk package: `composer install` di server, cek `require` vs `require-dev`.
7. Restart PHP-FPM dan queue worker.


Error ini termasuk dalam daftar [error Laravel yang paling sering terjadi](https://dhiecoderweb.com/kumpulan-solusi-error-laravel-paling-sering-terjadi). Jika Anda sedang men-deploy, ikuti urutan perintah di panduan [deploy Laravel ke VPS Ubuntu](https://dhiecoderweb.com/cara-deploy-laravel-ke-vps-ubuntu-nginx) atau [deploy ke shared hosting cPanel](https://dhiecoderweb.com/cara-deploy-laravel-ke-shared-hosting-cpanel), yang sudah memasukkan `composer install` dan pembersihan cache.

## FAQ

### Apakah setiap membuat class baru saya harus menjalankan composer dump-autoload?

Tidak, selama class berada di folder yang sudah dipetakan PSR-4 dan autoloader tidak dalam mode classmap authoritative. Anda perlu menjalankannya bila mengubah `composer.json`, menambah file di bagian `files`, atau memakai autoloader yang dioptimasi di produksi.

### Apa beda "Class not found" dan "Target class does not exist"?

"Class not found" adalah error PHP saat class dipakai langsung. "Target class [...] does not exist" adalah pesan service container Laravel saat gagal me-resolve class, misalnya controller pada route. Penyebab dan solusinya hampir sama.

### Kenapa jalan di Windows tetapi error di hosting?

Karena sistem file Windows tidak membedakan huruf besar-kecil, sedangkan server Linux membedakannya. Samakan huruf besar-kecil nama file dan folder dengan namespace, lalu ganti nama dengan `git mv`.

### Apakah aman menjalankan php artisan optimize:clear di produksi?

Aman. Perintah ini hanya menghapus file cache. Setelah itu, buat ulang cache dengan `php artisan optimize` atau `config:cache` dan `route:cache` agar performa tetap optimal.


---

<!-- Aktifkan tautan ini setelah artikel terbit di blog:
Artikel lengkap: https://dhiecoderweb.com/cara-mengatasi-class-not-found-laravel-php
-->

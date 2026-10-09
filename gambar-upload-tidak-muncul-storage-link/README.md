# Gambar Upload Laravel Tidak Muncul? Solusi storage:link

Gambar yang sudah berhasil di-upload di Laravel tetapi tidak muncul di browser hampir selalu disebabkan oleh satu dari lima hal: file tersimpan di disk yang salah (`local`, bukan `public`), symlink `public/storage` belum dibuat dengan `php artisan storage:link`, URL dibentuk dengan path yang keliru, `APP_URL` tidak sesuai, atau izin folder/server menolak membaca file. Artikel ini membahas cara mendiagnosis kelimanya secara berurutan, termasuk solusi ketika fungsi `symlink()` dimatikan di shared hosting.

Fokus tulisan ini adalah masalah *menampilkan* gambar, bukan proses validasi upload. Jadi kita anggap form dan controller upload Anda sudah berjalan, file sudah masuk ke server, tetapi tag `<img>` menampilkan ikon gambar rusak atau request-nya berujung 404.

## Memahami alur penyimpanan file di Laravel

Sebelum memperbaiki apa pun, pahami dulu kenapa Laravel tidak langsung menaruh file upload di folder `public`. Laravel memisahkan antara **tempat file disimpan** (folder `storage`) dan **tempat yang bisa diakses web** (folder `public`). Web server hanya melayani isi folder `public`. Folder `storage` berada di luar document root sehingga tidak bisa dibuka langsung lewat URL. Ini disengaja, supaya file seperti log, cache, dan dokumen privat tidak bocor.

Agar file tertentu tetap bisa diakses publik, Laravel menyediakan *disk* bernama `public` yang menyimpan file di `storage/app/public`. Lalu dibuat symbolic link (symlink) dari `public/storage` ke `storage/app/public`. Hasilnya, file di `storage/app/public/avatars/foto.jpg` bisa diakses di `https://domain-anda.com/storage/avatars/foto.jpg`.

Konfigurasinya ada di `config/filesystems.php`:

```
'disks' => [
    'local' => [
        'driver' => 'local',
        'root' => storage_path('app/private'), // Laravel 11; di Laravel 10: storage_path('app')
        'throw' => false,
    ],

    'public' => [
        'driver' => 'local',
        'root' => storage_path('app/public'),
        'url' => env('APP_URL').'/storage',
        'visibility' => 'public',
        'throw' => false,
    ],
],

'links' => [
    public_path('storage') => storage_path('app/public'),
],
```


Perhatikan dua hal penting. Pertama, disk `public` memakai `APP_URL` untuk membentuk URL. Kedua, array `links` menentukan symlink apa saja yang dibuat oleh perintah `storage:link`.

## Penyebab 1: file tersimpan di disk local, bukan public

Ini penyebab yang paling sering. Default disk Laravel diatur oleh `FILESYSTEM_DISK` di file `.env`, dan nilai bawaannya adalah `local`. Jika Anda menulis kode seperti ini:

```
$path = $request->file('foto')->store('avatars');
```


maka file masuk ke disk default, yaitu `local`. Di Laravel 11 lokasinya `storage/app/private/avatars`, di Laravel 10 `storage/app/avatars`. Folder itu tidak terhubung ke `public/storage`, sehingga gambar tidak akan pernah bisa dibuka lewat URL, sebanyak apa pun Anda menjalankan `storage:link`.

Solusinya, sebutkan disk secara eksplisit:

```
$path = $request->file('foto')->store('avatars', 'public');
// $path berisi "avatars/namaacak.jpg"

$user->update(['foto' => $path]);
```


Atau gunakan facade Storage:

```
use Illuminate\Support\Facades\Storage;

$path = Storage::disk('public')->putFile('avatars', $request->file('foto'));
```


Cara cepat memastikan lokasi file: buka terminal di root proyek lalu cari file tersebut.

```
find storage/app -name "namaacak.jpg"
```


Jika hasilnya bukan di bawah `storage/app/public`, masalahnya ada di disk. Pindahkan file lama secara manual atau upload ulang setelah kode diperbaiki.

### Kesalahan warisan: prefix "public/" di path

Di tutorial lama sering muncul pola `store('public/avatars')`. Pada Laravel 10 ke bawah, pola ini memang menaruh file di `storage/app/public/avatars` karena disk `local` berakar di `storage/app`. Masalahnya, nilai yang dikembalikan adalah `public/avatars/namaacak.jpg`. Kalau nilai itu disimpan ke database lalu dipakai dengan `asset('storage/'.$path)`, URL yang terbentuk menjadi `/storage/public/avatars/...` yang jelas tidak ada. Di Laravel 11, pola ini malah menaruh file di `storage/app/private/public/avatars`. Hindari pola ini dan selalu pakai disk `public`.

## Penyebab 2: symlink public/storage belum dibuat

Jika file sudah benar berada di `storage/app/public` tetapi URL `/storage/...` menghasilkan 404, kemungkinan besar symlink belum ada. Jalankan:

```
php artisan storage:link
```


Keluarannya kurang lebih menyatakan bahwa link `public/storage` sudah dibuat. Periksa hasilnya:

```
ls -la public/ | grep storage
# lrwxr-xr-x  storage -> /var/www/proyek/storage/app/public
```


Huruf `l` di awal dan tanda panah menunjukkan symlink valid. Beberapa situasi yang sering membuat symlink "hilang" atau rusak:

- **Deploy baru.** Folder `public/storage` tercantum di `.gitignore` bawaan Laravel, jadi tidak ikut ter-commit. Setiap server baru wajib menjalankan `storage:link` sekali.
- **Proyek dipindah folder.** Symlink menyimpan path absolut. Jika proyek dipindah dari `/home/lama` ke `/home/baru`, link menunjuk ke lokasi yang sudah tidak ada. Hapus link lama (`rm public/storage`) lalu buat ulang.
- **Folder biasa bernama storage.** Kadang ada yang tidak sengaja membuat folder asli `public/storage` (misalnya karena upload FTP). Perintah `storage:link` akan menolak dengan pesan bahwa link sudah ada. Pindahkan isinya, hapus folder itu, lalu jalankan ulang perintah.
- **Deploy tanpa downtime.** Pada pola folder `releases/` dengan symlink `current`, pastikan `storage` adalah folder bersama (shared) dan link dibuat untuk setiap rilis.


Opsi `--relative` membuat link dengan path relatif sehingga tetap valid ketika folder proyek dipindah. Opsi ini membutuhkan paket `symfony/filesystem`:

```
composer require symfony/filesystem
php artisan storage:link --relative
```


## Penyebab 3: symlink dinonaktifkan di shared hosting

Di banyak shared hosting, fungsi PHP `symlink()` dimatikan lewat `disable_functions`. Gejalanya, `storage:link` gagal dengan pesan seperti `Call to undefined function symlink()` atau `symlink() has been disabled for security reasons`. Beberapa alternatif yang bisa Anda pilih, dari yang paling disarankan:

### Alternatif A: buat symlink lewat SSH atau Terminal cPanel

Yang dimatikan hanya fungsi PHP-nya, bukan kemampuan sistem file. Jika hosting menyediakan SSH atau menu Terminal di cPanel, buat link langsung dengan perintah shell:

```
cd ~/public_html
ln -s /home/usernameanda/laravel/storage/app/public storage
```


Sesuaikan path dengan struktur Anda. Jika Anda mengikuti pola deploy di mana isi folder `public` Laravel dipindah ke `public_html`, link harus dibuat di dalam `public_html`. Panduan struktur folder tersebut ada di artikel [cara deploy Laravel ke shared hosting cPanel](https://dhiecoderweb.com/cara-deploy-laravel-ke-shared-hosting-cpanel).

### Alternatif B: buat symlink lewat Cron Job sekali jalan

Jika tidak ada SSH, menu Cron Jobs di cPanel tetap menjalankan perintah shell. Buat cron yang berjalan setiap menit berisi perintah `ln -s` di atas, tunggu satu menit, cek hasilnya di File Manager, lalu **hapus cron tersebut**.

### Alternatif C: arahkan root disk public langsung ke folder publik

Jika symlink sama sekali tidak memungkinkan, ubah disk `public` agar menyimpan file langsung di folder yang dilayani web:

```
'public' => [
    'driver' => 'local',
    'root' => public_path('storage'),
    'url' => env('APP_URL').'/storage',
    'visibility' => 'public',
],
```


Pada struktur `public_html`, Anda mungkin perlu path absolut, misalnya `'root' => '/home/usernameanda/public_html/storage'`. Kelemahannya, file upload kini bercampur dengan aset publik, dan ketika Anda menimpa folder `public` saat deploy, file upload bisa ikut terhapus. Selalu cadangkan folder ini.

### Alternatif D: layani file lewat route

Pilihan terakhir adalah membuat route yang membaca file dari storage dan mengirimkannya:

```
use Illuminate\Support\Facades\Storage;

Route::get('/media/{path}', function (string $path) {
    abort_unless(Storage::disk('public')->exists($path), 404);
    return Storage::disk('public')->response($path);
})->where('path', '.*');
```


Cara ini berfungsi di mana saja, tetapi setiap gambar kini melewati PHP dan framework, sehingga lebih lambat dan lebih berat dibanding file statis. Cocok untuk situs kecil, kurang ideal untuk galeri besar.

## Penyebab 4: URL dibentuk dengan cara yang salah

Ada dua cara umum menampilkan gambar dari disk public di Blade:

```
<img src="{{ asset('storage/' . $user->foto) }}" alt="Foto profil">

<img src="{{ Storage::url($user->foto) }}" alt="Foto profil">
```


Keduanya menghasilkan URL yang sama bila konfigurasi standar. Bedanya:   Aspekasset('storage/...')Storage::url()   Sumber URL dasarHost request saat ini (atau `ASSET_URL`)Nilai `url` di konfigurasi disk (berbasis `APP_URL`) Ikut berubah bila disk pindah ke S3Tidak, tetap mengarah ke /storageYa, otomatis memakai URL S3 Disk yang dipakaiTidak peduli diskDisk default, kecuali Anda pakai `Storage::disk('public')->url()`

Catatan penting: `Storage::url()` tanpa memilih disk akan memakai disk default. Jika `FILESYSTEM_DISK=local`, hasilnya bisa tidak sesuai harapan. Lebih aman menulis `Storage::disk('public')->url($path)`. Cara rapi lainnya adalah accessor di model sehingga logika URL hanya ada di satu tempat:

```
use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Support\Facades\Storage;

protected function fotoUrl(): Attribute
{
    return Attribute::get(fn () => $this->foto
        ? Storage::disk('public')->url($this->foto)
        : asset('images/avatar-default.png'));
}
```


Lalu di Blade cukup `{{ $user->foto_url }}`. Pastikan juga yang tersimpan di database hanya path relatif seperti `avatars/namaacak.jpg`, bukan path absolut server seperti `/var/www/.../storage/app/public/avatars/namaacak.jpg`.

## Penyebab 5: APP_URL tidak sesuai

Karena disk `public` memakai `env('APP_URL').'/storage'`, nilai `APP_URL` yang salah langsung merusak URL dari `Storage::url()`. Kasus yang sering terjadi:

- Saat development dengan `php artisan serve`, aplikasi berjalan di `http://127.0.0.1:8000`, tetapi `APP_URL` masih `http://localhost`. URL gambar mengarah ke port 80 yang tidak melayani apa pun.
- Di produksi `APP_URL` masih berisi alamat lokal, sehingga gambar mengarah ke `http://localhost/storage/...` di browser pengunjung.
- Situs sudah HTTPS tetapi `APP_URL` masih `http://`. Browser memblokir atau memperingatkan konten campuran (mixed content).


Perbaiki `.env`, lalu bersihkan cache konfigurasi karena di produksi nilai `.env` biasanya sudah di-cache:

```
APP_URL=https://domain-anda.com

php artisan config:clear
php artisan config:cache
```


Cara cepat memeriksa: buka `php artisan tinker` lalu jalankan `Storage::disk('public')->url('tes.jpg')`. URL yang keluar harus persis sama dengan domain yang Anda buka di browser.

## Penyebab 6: izin folder dan konfigurasi web server

Jika file ada, link benar, URL benar, tetapi responsnya 403 Forbidden, masalahnya di izin akses. Web server (misalnya user `www-data` di Ubuntu) harus bisa membaca file dan masuk ke setiap folder di sepanjang path-nya.

```
sudo chown -R www-data:www-data storage bootstrap/cache
sudo find storage -type d -exec chmod 755 {} \;
sudo find storage -type f -exec chmod 644 {} \;
```


Hindari `chmod -R 777`. Izin itu memang "membereskan" masalah, tetapi membuat semua user di server bisa menulis ke folder Anda.

Pada web server, pastikan symlink diizinkan diikuti. Di Apache, document root perlu `Options FollowSymLinks` (atau `SymLinksIfOwnerMatch`, yang mensyaratkan pemilik link dan target sama). Di Nginx, direktif `disable_symlinks` default-nya `off`, tetapi beberapa panel hosting mengaktifkannya. Untuk server VPS, contoh konfigurasi Nginx yang lengkap bisa Anda lihat di [cara deploy Laravel ke VPS Ubuntu dengan Nginx](https://dhiecoderweb.com/cara-deploy-laravel-ke-vps-ubuntu-nginx).

## Langkah diagnosis berurutan

1. Buka DevTools browser, tab Network, klik request gambar. Catat URL lengkap dan status kodenya (404, 403, atau gagal koneksi).
2. Bandingkan host dan port URL itu dengan alamat situs. Jika berbeda, periksa `APP_URL`.
3. Jika URL mengandung `/storage/public/` atau path absolut server, perbaiki nilai yang disimpan di database.
4. Cari file fisiknya dengan `find storage/app -name ...`. Pastikan ada di `storage/app/public`.
5. Cek `ls -la public/storage`. Pastikan symlink ada dan menunjuk ke folder yang benar.
6. Jika status 403, periksa izin folder dan pengaturan symlink di web server.


Urutan ini bergerak dari yang paling murah diperiksa ke yang paling teknis, sehingga Anda tidak membuang waktu mengutak-atik izin padahal masalahnya hanya port yang salah.

## Kesalahan umum yang perlu dihindari

- Menjalankan `storage:link` berulang kali padahal file tersimpan di disk `local`.
- Menyimpan URL lengkap (dengan domain) ke database. Ketika domain berganti, semua gambar rusak. Simpan path relatif saja.
- Meng-commit folder `public/storage` atau isi `storage/app/public` ke Git.
- Lupa `config:clear` setelah mengubah `.env` di server yang memakai config cache.
- Memakai `chmod 777` sebagai solusi permanen.


Masalah serupa yang berhubungan dengan konfigurasi juga dibahas di [kumpulan solusi error Laravel paling sering terjadi](https://dhiecoderweb.com/kumpulan-solusi-error-laravel-paling-sering-terjadi).

## Checklist ringkas

- Upload memakai `store('folder', 'public')` atau `Storage::disk('public')`.
- Database menyimpan path relatif, misalnya `avatars/abc.jpg`.
- `public/storage` adalah symlink ke `storage/app/public`.
- `APP_URL` sama persis dengan alamat situs, termasuk skema dan port.
- URL dibentuk dengan `Storage::disk('public')->url()` atau `asset('storage/...')`.
- Izin 755 untuk folder, 644 untuk file, dimiliki user web server.


## FAQ

### Apakah saya perlu menjalankan storage:link setiap kali upload?

Tidak. Symlink cukup dibuat sekali per server atau per lokasi proyek. Setelah ada, semua file baru di `storage/app/public` otomatis bisa diakses. Ulangi hanya bila server baru, proyek dipindah, atau link terhapus.

### Gambar muncul di lokal tetapi tidak di hosting, kenapa?

Penyebab paling umum adalah symlink yang belum dibuat di server (karena `public/storage` tidak ikut ter-upload atau di-ignore Git) dan `APP_URL` di `.env` server yang belum diganti. Periksa dua hal itu terlebih dahulu.

### Lebih baik pakai asset() atau Storage::url()?

Untuk proyek yang mungkin pindah ke penyimpanan cloud seperti S3, `Storage::disk('public')->url()` lebih fleksibel karena mengikuti konfigurasi disk. Untuk proyek sederhana yang pasti di server sendiri, `asset('storage/...')` juga sah dan tidak bergantung pada `APP_URL`.

### Apakah aman menyimpan file privat di disk public?

Tidak. Semua file di disk `public` bisa dibuka siapa saja yang tahu URL-nya. Dokumen seperti KTP atau invoice sebaiknya disimpan di disk `local` dan dikirim lewat route yang memeriksa otorisasi, misalnya dengan `Storage::download()`.

### Kenapa storage:link bilang link sudah ada padahal gambar tetap 404?

Kemungkinan `public/storage` adalah folder biasa atau symlink yang menunjuk ke path lama. Cek dengan `ls -la public`. Jika bukan symlink yang benar, hapus lalu jalankan ulang `php artisan storage:link`.


---

<!-- Aktifkan tautan ini setelah artikel terbit di blog:
Artikel lengkap: https://dhiecoderweb.com/cara-mengatasi-gambar-upload-laravel-tidak-muncul-storage-link
-->

# Laravel 405 Method Not Allowed: Penyebab dan Solusinya

Error 405 Method Not Allowed di Laravel muncul ketika URL yang Anda akses memang ada di daftar route, tetapi tidak untuk HTTP method yang dipakai. Contohnya, route didaftarkan dengan `Route::post()` tetapi diakses dengan GET, atau form mengirim POST ke route `Route::put()` tanpa `@method('PUT')`. Cara tercepat memperbaikinya: baca bagian "Supported methods" di pesan error, lalu cocokkan dengan method yang benar-benar dikirim browser.

Pesan lengkapnya di halaman debug atau log biasanya seperti ini:

```
Symfony\Component\HttpKernel\Exception\MethodNotAllowedHttpException

The GET method is not supported for route posts/store. Supported methods: POST.
```


Pesan tersebut sebenarnya sudah memberi tahu tiga hal penting: method yang dikirim (`GET`), path yang diminta (`posts/store`), dan method yang diterima route itu (`POST`). Di produksi dengan `APP_DEBUG=false`, Anda hanya melihat halaman "405 | Method Not Allowed", jadi buka `storage/logs/laravel.log` atau aktifkan debug sementara di lingkungan lokal untuk melihat detailnya.

## Langkah 0: Lihat method yang terdaftar dan yang dikirim

Dua informasi ini menyelesaikan sebagian besar kasus. Pertama, lihat method apa saja yang didaftarkan untuk path tersebut:

```
php artisan route:list --path=posts
```


```
  GET|HEAD   posts ................ posts.index › PostController@index
  POST       posts ................ posts.store › PostController@store
  GET|HEAD   posts/{post} ......... posts.show › PostController@show
  PUT|PATCH  posts/{post} ......... posts.update › PostController@update
  DELETE     posts/{post} ......... posts.destroy › PostController@destroy
```


Kedua, lihat apa yang sebenarnya dikirim browser. Buka DevTools, tab Network, centang "Preserve log", lalu ulangi aksi yang memicu error. Perhatikan kolom Method dan Status. Jika Anda melihat request POST dengan status 301 atau 302 diikuti request GET ke URL yang mirip, Anda sedang berhadapan dengan masalah redirect (dibahas di penyebab 3).

## Penyebab 1: Form HTML tidak mendukung PUT, PATCH, dan DELETE

Ini penyebab paling sering. Elemen `<form>` HTML hanya mendukung GET dan POST. Jika Anda menulis `method="PUT"` atau `method="DELETE"`, browser menganggap nilainya tidak valid dan mengirim form sebagai GET. Jika tidak ada route GET untuk URL tersebut, hasilnya error seperti "The GET method is not supported for route profile. Supported methods: PUT." Jika ada (misalnya route `show` pada resource), gejalanya lebih halus: form seolah terkirim, tetapi yang terbuka hanya halaman detail dan tidak ada data yang tersimpan.

Laravel menyediakan *method spoofing*: form dikirim sebagai POST dengan field tersembunyi `_method`, lalu Laravel memperlakukannya sebagai PUT/PATCH/DELETE.

```
<!-- Salah: browser mengirim GET -->
<form action="{{ route('posts.update', $post) }}" method="PUT">

<!-- Benar -->
<form action="{{ route('posts.update', $post) }}" method="POST">
    @csrf
    @method('PUT')
    <input type="text" name="title" value="{{ old('title', $post->title) }}">
    <button type="submit">Simpan</button>
</form>

<!-- Tombol hapus -->
<form action="{{ route('posts.destroy', $post) }}" method="POST">
    @csrf
    @method('DELETE')
    <button type="submit" onclick="return confirm('Hapus?')">Hapus</button>
</form>
```


`@method('PUT')` menghasilkan `<input type="hidden" name="_method" value="PUT">`. Pastikan field ini ada di dalam form yang dikirim, bukan di form lain atau di luar tag `<form>`. Kesalahan kecil yang sering terjadi adalah form bersarang: HTML tidak mengizinkan form di dalam form, sehingga browser mengabaikan form dalam dan tombol "Hapus" justru mengirim form "Edit" yang luar.

### Link biasa untuk aksi POST

Kasus serupa: tombol logout dibuat dengan tag `<a href="/logout">`. Link selalu menghasilkan request GET, padahal route logout di Laravel Breeze, Jetstream, dan Fortify didaftarkan sebagai POST. Hasilnya "The GET method is not supported for route logout. Supported methods: POST." Ubah menjadi form:

```
<form method="POST" action="{{ route('logout') }}">
    @csrf
    <button type="submit">Logout</button>
</form>
```


Hal yang sama terjadi jika Anda membuka URL POST langsung di address bar, me-refresh halaman hasil POST yang tidak melakukan redirect, atau menekan tombol Back ke halaman tersebut. Pola Post/Redirect/Get (selalu `return redirect()->route(...)` setelah menyimpan data) mencegah masalah ini.

## Penyebab 2: Action form mengarah ke URL yang salah

Jika `action` form salah tulis, request mungkin jatuh ke route lain yang path-nya cocok tetapi method-nya berbeda. Contoh: form tambah data mengirim POST ke `/posts/create`, padahal `/posts/create` hanya GET (menampilkan form), dan penyimpanan seharusnya ke `/posts`. Selalu gunakan nama route agar URL tidak ditulis manual:

```
<form action="{{ route('posts.store') }}" method="POST">
```


Jika `action` dikosongkan, browser mengirim form ke URL halaman saat ini. Untuk halaman edit di `/posts/5/edit`, itu berarti POST ke route yang hanya menerima GET.

## Penyebab 3: Redirect diam-diam mengubah POST menjadi GET

Ini penyebab yang paling membingungkan karena kode form Anda terlihat benar. Ketika server membalas request POST dengan redirect 301 atau 302, browser pada praktiknya mengikuti redirect tersebut dengan request GET dan membuang isi form. Hanya status 307 dan 308 yang mempertahankan method dan body.

Sumber redirect yang umum:

- **Trailing slash.** File `public/.htaccess` bawaan Laravel punya aturan "Redirect Trailing Slashes If Not A Folder" yang mengembalikan 301 dari `/posts/` ke `/posts`. Form dengan `action="/posts/"` berubah menjadi GET `/posts`.
- **HTTP ke HTTPS.** Form yang action-nya masih `http://` (misalnya karena `APP_URL` masih http, atau aplikasi di belakang proxy/Cloudflare tidak mendeteksi HTTPS) diarahkan ulang ke `https://` dengan 301.
- **www dan non-www.** Form mengirim ke `contoh.com` padahal server me-redirect ke `www.contoh.com`, atau sebaliknya.


Cara memastikannya mudah: di tab Network, klik request POST yang statusnya 301 atau 302, lalu lihat header `Location` pada respons. URL di header itulah tujuan redirect, dan biasanya perbedaannya langsung terlihat, entah garis miring di akhir, skema `http`, atau awalan `www`.

Perbaikannya: hilangkan sumber redirect, jangan mengakalinya. Gunakan helper `route()` yang tidak menambahkan garis miring di akhir, set `APP_URL` dengan skema dan host yang benar, dan jika aplikasi berjalan di belakang load balancer atau Cloudflare, konfigurasikan trusted proxies. Di Laravel 11, ini dilakukan di `bootstrap/app.php`:

```
->withMiddleware(function (Middleware $middleware) {
    $middleware->trustProxies(at: '*');
})
```


Sebagai cadangan, Anda juga bisa memaksa URL yang dihasilkan memakai HTTPS di `AppServiceProvider`:

```
use Illuminate\Support\Facades\URL;

public function boot(): void
{
    if ($this->app->environment('production')) {
        URL::forceScheme('https');
    }
}
```


## Penyebab 4: Route cache basi

Jika Anda menambah atau mengubah method sebuah route (misalnya dari `Route::get` ke `Route::match(['get', 'post'], ...)`) dan server memakai route cache, perubahan tersebut tidak terbaca. `route:list` di server akan menunjukkan method lama.

```
php artisan route:clear
php artisan route:list --path=kontak
php artisan route:cache   # bangun ulang untuk produksi
```


Perhatikan juga urutan definisi route. Jika dua route punya URI dan method yang sama, yang didefinisikan belakangan menimpa yang pertama. Route seperti `Route::resource('posts', ...)->only(['index', 'show'])` sengaja tidak mendaftarkan `store`, `update`, dan `destroy`, sehingga POST ke `/posts` menghasilkan 405.

## Penyebab 5: Route API vs route web

Route di `routes/api.php` otomatis diberi prefix `/api`. Jika JavaScript Anda mengirim `POST /posts` padahal route-nya didefinisikan di `api.php`, request jatuh ke route web `GET /posts` dan Anda mendapat 405. Gunakan URL lengkap `/api/posts`.

Di Laravel 11, file `routes/api.php` tidak lagi tersedia secara default. Jika Anda membuatnya manual tanpa mendaftarkannya, route tidak pernah dimuat. Jalankan perintah resmi berikut agar file dibuat dan didaftarkan di `bootstrap/app.php`:

```
php artisan install:api
```


Untuk request dari JavaScript, pastikan method yang dikirim sesuai. Jika Anda harus mengirim PUT lewat `FormData` (misalnya untuk upload file), kirim sebagai POST dan sertakan `_method`:

```
const data = new FormData(form);
data.append('_method', 'PUT');

await fetch(`/api/posts/${id}`, {
    method: 'POST',
    headers: { 'Accept': 'application/json' },
    body: data,
});
```


Header `Accept: application/json` membuat Laravel membalas error dalam format JSON, sehingga pesan 405 lengkap mudah dibaca di tab Network. Jika Anda belum familier dengan konsep method HTTP pada API, baca [apa itu REST API dan cara kerjanya](https://dhiecoderweb.com/apa-itu-rest-api-dan-cara-kerjanya-untuk-pemula).

## Jangan tertukar dengan 404 dan 419

- **404**: path sama sekali tidak cocok dengan route mana pun.
- **405**: path cocok, method tidak.
- **419 Page Expired**: method dan path cocok, tetapi token CSRF tidak ada atau kedaluwarsa. Solusinya `@csrf` di form, bukan mengubah method.


Penjelasan lebih dasar tentang method dan status code ada di artikel [cara kerja HTTP dan HTTPS](https://dhiecoderweb.com/cara-kerja-http-dan-https-untuk-pemula).

## Kesalahan umum

- Menulis `method="PUT"` atau `method="DELETE"` langsung di tag form.
- Meletakkan `@method` di luar form atau di form yang salah karena form bersarang.
- Logout dengan tag `<a>` padahal route-nya POST.
- Action form diakhiri garis miring sehingga terkena redirect 301.
- Mengakali 405 dengan mengganti semua route menjadi `Route::any()`. Ini menghilangkan gejalanya tetapi membuka aksi yang mengubah data lewat GET.
- Lupa membersihkan route cache setelah deploy.


## Checklist

1. Baca "Supported methods" di pesan error.
2. Bandingkan dengan `php artisan route:list --path=...`.
3. Cek tab Network: adakah 301/302 sebelum request yang gagal?
4. Form PUT/PATCH/DELETE memakai `method="POST"` + `@method(...)` + `@csrf`.
5. Gunakan `route()` untuk action, tanpa garis miring di akhir.
6. Untuk API, pastikan prefix `/api` dan method di `fetch`/axios benar.
7. `php artisan route:clear` di server.


Daftar error Laravel lain yang sering muncul ada di [kumpulan solusi error Laravel](https://dhiecoderweb.com/kumpulan-solusi-error-laravel-paling-sering-terjadi).

## FAQ

### Kenapa form saya dengan method="PUT" dikirim sebagai GET?

Karena form HTML hanya mengenal GET dan POST. Nilai lain dianggap tidak valid dan browser memakai GET. Gunakan `method="POST"` dan tambahkan `@method('PUT')` di dalam form.

### Apakah aman memakai Route::any() atau Route::match() untuk menghindari 405?

`Route::match()` dengan daftar method yang jelas boleh dipakai jika satu URL memang perlu melayani lebih dari satu method. `Route::any()` untuk aksi yang mengubah data tidak disarankan, karena aksi tersebut jadi bisa dipicu oleh link atau gambar biasa.

### Kenapa error 405 hanya terjadi di produksi?

Biasanya karena ada redirect di produksi yang tidak ada di lokal (HTTP ke HTTPS, www ke non-www) atau karena route cache basi. Periksa tab Network dan jalankan `php artisan route:clear`.

### Bagaimana cara mengirim DELETE dari JavaScript?

Dengan `fetch` atau axios, Anda bisa langsung memakai `method: 'DELETE'`. Untuk route web, sertakan header `X-CSRF-TOKEN`. Method spoofing dengan `_method` hanya diperlukan untuk form HTML atau ketika mengirim `FormData` berisi file.


---

<!-- Aktifkan tautan ini setelah artikel terbit di blog:
Artikel lengkap: https://dhiecoderweb.com/laravel-405-method-not-allowed-solusi
-->

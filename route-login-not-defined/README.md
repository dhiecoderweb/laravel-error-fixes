# Route [login] not defined di Laravel

Pernahkah kamu menjalankan aplikasi Laravel lalu tiba-tiba muncul error seperti ini di browser?

```
Symfony\Component\Routing\Exception\RouteNotFoundException
Route [login] not defined.
```


Error ini sangat umum ditemui oleh developer Laravel, terutama bagi yang baru mulai belajar atau baru saja mengaktifkan middleware `auth`. Artikel ini akan menjelaskan penyebabnya secara detail dan memberikan beberapa solusi praktis.

## Penyebab Error Route [login] not defined

Error ini muncul karena middleware `auth` bawaan Laravel mencoba melakukan redirect ke route bernama `login` ketika pengguna belum terautentikasi. Jika route dengan nama `login` tidak terdaftar di aplikasi kamu, maka Laravel akan melempar exception ini.

Beberapa skenario umum penyebabnya:

- Kamu menerapkan middleware `auth` pada route tapi belum membuat sistem autentikasi (login).
- Kamu lupa meng-install atau mendaftarkan package autentikasi seperti Laravel Breeze atau Jetstream.
- Route login sudah ada tapi tidak diberi nama `login`.
- File `routes/auth.php` tidak di-include di `routes/web.php`.


## Solusi 1: Install Laravel Breeze (Direkomendasikan)

Jika kamu memang membutuhkan fitur autentikasi lengkap (login, register, lupa password), cara termudah adalah menggunakan Laravel Breeze.

1. Install package Breeze via Composer:


```
composer require laravel/breeze --dev
```


1. Jalankan perintah install Breeze:


```
php artisan breeze:install blade
```


1. Install dependensi frontend dan compile aset:


```
npm install && npm run dev
```


1. Jalankan migrasi database:


```
php artisan migrate
```


Setelah itu, route bernama `login` akan otomatis terdaftar dan error akan hilang.

## Solusi 2: Definisikan Route Login Secara Manual

Jika kamu tidak ingin menggunakan Breeze dan sudah memiliki controller login sendiri, pastikan kamu mendefinisikan route dengan nama `login` di file `routes/web.php`:

```
use App\Http\Controllers\Auth\LoginController;

Route::get('/login', [LoginController::class, 'showLoginForm'])->name('login');
Route::post('/login', [LoginController::class, 'login'])->name('login.post');
Route::post('/logout', [LoginController::class, 'logout'])->name('logout');
```


Perhatikan penggunaan `->name('login')` — ini yang membuat Laravel bisa menemukan route tersebut ketika middleware `auth` melakukan redirect.

## Solusi 3: Ubah Redirect di Middleware Auth

Jika kamu menggunakan nama route yang berbeda untuk halaman login, kamu bisa mengubah konfigurasi redirect middleware auth. Buka file `app/Http/Middleware/Authenticate.php`:

```
<?php

namespace App\Http\Middleware;

use Illuminate\Auth\Middleware\Authenticate as Middleware;
use Illuminate\Http\Request;

class Authenticate extends Middleware
{
    protected function redirectTo(Request $request): ?string
    {
        return $request->expectsJson() ? null : route('auth.login'); // ganti dengan nama route kamu
    }
}
```


Pada Laravel 11, konfigurasi ini bisa juga diatur melalui `bootstrap/app.php`:

```
<?php

use Illuminate\Foundation\Application;

return Application::configure(basePath: dirname(__DIR__))
    ->withMiddleware(function ($middleware) {
        $middleware->redirectGuestsTo(fn () => route('auth.login'));
    })
    ->create();
```


## Solusi 4: Hapus Middleware Auth Jika Tidak Diperlukan

Jika kamu menerapkan middleware `auth` secara tidak sengaja pada route yang seharusnya bisa diakses publik, cukup hapus middleware tersebut dari route terkait.

Sebelum (menyebabkan error):

```
Route::get('/dashboard', function () {
    return view('dashboard');
})->middleware(['auth'])->name('dashboard');
```


Sesudah (jika halaman memang publik):

```
Route::get('/dashboard', function () {
    return view('dashboard');
})->name('dashboard');
```


## Cara Memeriksa Daftar Route yang Terdaftar

Untuk memastikan route `login` sudah terdaftar, jalankan perintah Artisan berikut di terminal:

```
php artisan route:list --name=login
```


Jika muncul baris dengan nama `login`, berarti route sudah terdaftar dengan benar. Jika hasilnya kosong, berarti kamu perlu mendefinisikannya terlebih dahulu menggunakan salah satu solusi di atas.

## Kesimpulan

Error **Route [login] not defined** di Laravel terjadi karena middleware `auth` tidak bisa menemukan route bernama `login` untuk melakukan redirect. Solusi terbaik tergantung kebutuhanmu: gunakan Laravel Breeze untuk autentikasi lengkap secara instan, atau definisikan route login secara manual jika kamu membuat sistem auth sendiri. Pastikan selalu menggunakan `->name('login')` pada route halaman login agar middleware dapat menemukan tujuan redirect dengan benar.


---

Artikel lengkap dengan penjelasan tambahan: https://dhiecoderweb.com/cara-mengatasi-error-route-login-not-defined-laravel

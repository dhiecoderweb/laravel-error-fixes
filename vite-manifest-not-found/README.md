# Vite manifest not found di Laravel

Setelah upgrade ke Laravel versi baru yang memakai **Vite**, banyak developer tiba-tiba mendapat halaman error putih dengan pesan:

```
Vite manifest not found at: /path/public/build/manifest.json
```


atau varian lain: *"Unable to locate file in Vite manifest"*. Error ini muncul karena Laravel mencari hasil build aset (CSS/JS) yang belum tersedia. Kabar baiknya, penyebabnya sedikit dan mudah diperbaiki. Mari kita bahas.

## Kenapa Error Ini Muncul?

Sejak Laravel 9.19+, Laravel Mix diganti **Vite**. Directive `@vite` di Blade akan mencari file `public/build/manifest.json` untuk tahu nama file aset hasil build. Kalau file itu belum ada — karena aset belum di-build atau dev server tidak jalan — Laravel melempar error di atas.

## Solusi untuk Mode Development

Saat ngoding di lokal, Anda cukup menjalankan **Vite dev server**. Buka terminal kedua (biarkan `php artisan serve` tetap jalan di terminal pertama), lalu:

```
npm install     # sekali saja, jika belum
npm run dev
```


Selama `npm run dev` aktif, Vite membuat file penanda `public/hot` dan menyajikan aset secara live. Kalau Anda menutup dev server, error bisa muncul lagi — cukup jalankan ulang.

## Solusi untuk Mode Production

Di server production Anda **tidak** menjalankan `npm run dev`. Sebagai gantinya, build aset satu kali agar `manifest.json` dibuat permanen:

```
npm install
npm run build
```


Perintah ini menghasilkan folder `public/build/` lengkap dengan `manifest.json`. Setelah itu error hilang tanpa perlu Node berjalan terus.

## Masalah Umum di Shared Hosting

Banyak shared hosting **tidak menyediakan Node.js**, sehingga `npm run build` tidak bisa dijalankan di server. Solusinya: build di komputer lokal, lalu upload folder `public/build/` ke server.

```
# di lokal
npm run build
# lalu upload seluruh isi public/build/ ke public/build/ di server
```


Pastikan folder `public/build/` (termasuk `manifest.json`) ikut ter-upload dan tidak masuk `.gitignore` jika Anda deploy via Git.

## Cek Konfigurasi @vite di Blade

Pastikan layout Anda memanggil `@vite` dengan path yang benar sesuai `vite.config.js`:

```
<!-- resources/views/layouts/app.blade.php -->
@vite(['resources/css/app.css', 'resources/js/app.js'])
```


Path di dalam `@vite` harus cocok dengan yang didaftarkan di `vite.config.js`:

```
// vite.config.js
export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,
        }),
    ],
});
```


## Checklist Cepat

- **Development:** sudah menjalankan `npm run dev`?
- **Production:** sudah `npm run build` dan folder `public/build/` ada?
- File `public/build/manifest.json` benar-benar ada di server?
- Path aset di `@vite` cocok dengan `vite.config.js`?
- Sudah `npm install` sehingga folder `node_modules` tersedia (untuk build)?


## Kesimpulan

Error "Vite manifest not found" bukan bug, melainkan pertanda aset belum di-build. Di lokal jalankan `npm run dev`; di production jalankan `npm run build` (atau upload hasil build bila hosting tanpa Node). Setelah `manifest.json` tersedia, halaman kembali normal. Kalau Anda deploy ke shared hosting, cek juga panduan kami tentang cara deploy Laravel ke cPanel.


---

Artikel lengkap dengan penjelasan tambahan: https://dhiecoderweb.com/cara-mengatasi-vite-manifest-not-found-laravel

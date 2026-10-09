# PHP 8 Undefined Array Key & Undefined Variable: Solusi

Pesan `Undefined array key`, `Undefined variable`, dan `Attempt to read property on null` berarti kode Anda membaca sesuatu yang tidak ada: indeks array yang tidak di-set, variabel yang belum diberi nilai, atau properti dari objek yang ternyata `null`. Sejak PHP 8.0, ketiganya naik level dari Notice menjadi Warning, sehingga lebih mudah terlihat, dan di Laravel langsung berubah menjadi exception yang menghentikan halaman. Solusinya bukan menyembunyikan warning, tetapi memutuskan apa yang seharusnya terjadi saat data tidak ada, lalu menuliskannya dengan `??`, `?->`, `isset()`, atau validasi.

Bentuk pesannya di PHP 8:

```
Warning: Undefined array key "email" in /var/www/app/register.php on line 14

Warning: Undefined variable $total in /var/www/app/report.php on line 32

Warning: Attempt to read property "name" on null in /var/www/app/profile.php on line 9
```


Dan di Laravel, karena semua warning diubah menjadi exception:

```
ErrorException
Undefined array key "email"

ErrorException
Attempt to read property "name" on null
(View: /var/www/app/resources/views/posts/show.blade.php)
```


## Kenapa PHP 8 jadi "lebih cerewet"?

Di PHP 7, membaca indeks array yang tidak ada menghasilkan *Notice* "Undefined index", membaca variabel yang tidak ada menghasilkan Notice "Undefined variable", dan membaca properti dari non-objek menghasilkan Notice "Trying to get property of non-object". Banyak server produksi mengatur `error_reporting` agar Notice tidak ditampilkan, jadi bug ini lolos bertahun-tahun tanpa ketahuan. Nilainya diam-diam menjadi `null` dan kode tetap jalan.

PHP 8.0 mengklasifikasi ulang error engine. Ketiga kasus tadi sekarang menjadi *Warning*, dan pesannya diganti menjadi lebih jelas:   PHP 7.x (Notice)PHP 8.x (Warning)   Undefined index: emailUndefined array key "email" Undefined offset: 3Undefined array key 3 Undefined variable: totalUndefined variable $total Trying to get property 'name' of non-objectAttempt to read property "name" on null

Perilakunya sendiri tidak berubah: ekspresi tersebut tetap bernilai `null` dan skrip PHP murni tetap berjalan. Yang berubah adalah tingkat keparahannya. Jadi kode lama yang "baik-baik saja" di PHP 7 tiba-tiba memenuhi log atau layar setelah server di-upgrade ke PHP 8.

Laravel melangkah lebih jauh. Saat booting, Laravel memasang error handler yang mengubah setiap error PHP (kecuali deprecation, yang hanya dicatat ke log) menjadi `ErrorException`. Artinya, di aplikasi Laravel, warning sekecil apa pun langsung menghasilkan halaman error 500. Ini sengaja: bug muncul saat pengembangan, bukan di data pelanggan.

Perlu dibedakan dengan error yang benar-benar fatal: memanggil **method** pada null (`$user->getName()` saat `$user` null) menghasilkan `Error: Call to a member function getName() on null`, yang menghentikan skrip bahkan di PHP murni.

## Penyebab paling sering (berurutan)

1. **Input dari luar yang tidak dikirim**: `$_POST['email']`, `$_GET['page']`, atau field JSON yang opsional.
2. **Relasi atau hasil query yang kosong**: `$post->author->name` ketika author sudah dihapus, atau `User::where(...)->first()` yang mengembalikan null.
3. **Variabel yang hanya di-set di dalam if atau loop**, lalu dipakai di luarnya.
4. **Variabel view yang tidak dikirim controller**: Blade memakai `$posts`, tetapi salah satu controller lupa mengirimkannya.
5. **Struktur array dari API eksternal yang berubah** atau tidak selalu lengkap.


## Solusi untuk "Undefined array key"

### Operator null coalescing ??

Operator `??` mengembalikan nilai kiri jika ada dan tidak null, atau nilai kanan jika tidak. Operator ini tidak memicu warning untuk indeks yang tidak ada, termasuk pada array bertingkat.

```
// Sebelum: warning jika 'page' tidak dikirim
$page = $_GET['page'];

// Sesudah
$page = (int) ($_GET['page'] ?? 1);

// Array bertingkat juga aman
$city = $data['user']['address']['city'] ?? 'Tidak diketahui';

// Menetapkan default ke array itu sendiri (PHP 7.4+)
$options['timeout'] ??= 30;
```


### isset() vs array_key_exists()

Keduanya sering tertukar. `isset()` mengembalikan `false` jika kunci tidak ada *atau* nilainya `null`. `array_key_exists()` hanya peduli apakah kuncinya ada.

```
$data = ['nama' => 'Budi', 'telepon' => null];

isset($data['telepon']);              // false (nilainya null)
array_key_exists('telepon', $data);   // true  (kuncinya ada)
isset($data['email']);                // false
array_key_exists('email', $data);     // false
```


Gunakan `array_key_exists()` ketika "dikirim tapi kosong" punya arti berbeda dari "tidak dikirim", misalnya pada PATCH API yang membolehkan pengguna menghapus nomor telepon dengan mengirim `null`.

### Di Laravel: pakai helper, bukan superglobal

Objek `Request` dan helper array Laravel sudah aman terhadap kunci yang tidak ada:

```
$email = $request->input('email');             // null jika tidak ada
$page  = $request->integer('page', 1);         // default 1
$sort  = $request->query('sort', 'terbaru');

use Illuminate\Support\Arr;
$city = Arr::get($payload, 'user.address.city', 'Tidak diketahui');
$city = data_get($payload, 'user.address.city');
```


Lebih baik lagi, validasi input dulu. Setelah `$request->validate()`, Anda tahu persis kunci mana yang dijamin ada:

```
$data = $request->validate([
    'email' => ['required', 'email'],
    'telepon' => ['nullable', 'string'],
]);

$data['email'];            // dijamin ada
$data['telepon'] ?? null;  // nullable: hanya ada di hasil validate() jika memang dikirim
```


Jika Anda sering memeriksa struktur JSON dari API pihak ketiga, [JSON Viewer](https://dhiecoderweb.com/tools/json-viewer) membantu melihat kunci mana yang benar-benar ada sebelum menulis kode yang membacanya.

## Solusi untuk "Attempt to read property on null"

### Operator nullsafe ?->

PHP 8.0 menambahkan operator nullsafe. Jika bagian kiri bernilai null, seluruh rantai berhenti dan menghasilkan null, tanpa warning dan tanpa error.

```
// Sebelum: warning/exception jika author null
$nama = $post->author->name;

// Sesudah
$nama = $post->author?->name ?? 'Anonim';

// Juga bekerja untuk pemanggilan method
$tanggal = $user->lastLogin?->format('d M Y');
```


Catatan: `?->` hanya melindungi dari nilai `null`. Ia tidak melindungi dari variabel yang tidak didefinisikan (`$tidakAda?->name` tetap memicu "Undefined variable"), dan tidak bisa dipakai untuk menulis nilai.

### Di Eloquent: withDefault() dan findOrFail()

Untuk relasi `belongsTo`, `hasOne`, dan sejenisnya, Laravel bisa mengembalikan model kosong alih-alih null:

```
public function author()
{
    return $this->belongsTo(User::class)->withDefault([
        'name' => 'Penulis tidak tersedia',
    ]);
}
```


Dan untuk data yang *wajib* ada, jangan biarkan null merambat. Gunakan `findOrFail()` atau `firstOrFail()` supaya pengguna mendapat 404 yang jelas, bukan error 500 tiga baris kemudian:

```
$post = Post::where('slug', $slug)->firstOrFail();
```


## Solusi untuk "Undefined variable"

Variabel yang tidak didefinisikan hampir selalu tanda alur logika yang belum lengkap. Inisialisasi variabel sebelum blok kondisi atau loop:

```
// Sebelum
foreach ($items as $item) {
    $total += $item['harga'];   // warning di iterasi pertama
}
if ($status === 'aktif') {
    $pesan = 'Akun aktif';
}
echo $pesan;                    // warning jika status bukan 'aktif'

// Sesudah
$total = 0;
foreach ($items as $item) {
    $total += $item['harga'];
}
$pesan = $status === 'aktif' ? 'Akun aktif' : 'Akun tidak aktif';
```


### Di Blade

Error "Undefined variable $posts (View: ...)" berarti view dirender tanpa variabel tersebut. Perbaiki di sumbernya, yaitu controller atau view composer, lalu tambahkan pengaman di view jika variabel memang opsional:

```
// Controller
return view('posts.index', [
    'posts' => $posts,
    'kategori' => $kategori ?? null,
]);
```


```
{{-- View --}}
{{ $judul ?? 'Semua Artikel' }}

@isset($kategori)
    <p>Kategori: {{ $kategori->nama }}</p>
@endisset

{{ $post->author?->name ?? 'Anonim' }}

@forelse ($posts ?? [] as $post)
    <li>{{ $post->title }}</li>
@empty
    <li>Belum ada artikel.</li>
@endforelse
```


Untuk Blade component, deklarasikan nilai default dengan `@props` agar komponen tidak bergantung pada pemanggil yang selalu mengirim semua atribut:

```
@props(['judul' => 'Tanpa judul', 'warna' => 'biru'])
```


## Memperbaiki akar masalah vs menyembunyikan warning

Ada beberapa cara cepat untuk "menghilangkan" warning yang sebaiknya dihindari:

- **Operator `@`** (`@$data['email']`) menekan pesan, tetapi nilainya tetap null dan bug tetap ada. Operator ini juga menyembunyikan error lain di ekspresi yang sama.
- **Menurunkan `error_reporting`** agar warning tidak muncul hanya mengembalikan Anda ke kondisi PHP 7: bug tetap ada, Anda hanya tidak melihatnya.
- **Menaburkan `??` di semua tempat** tanpa berpikir. Jika `$order['total']` seharusnya selalu ada, menulis `$order['total'] ?? 0` bisa diam-diam membuat pesanan bernilai nol. Untuk data wajib, lebih baik gagal dengan jelas.


Pertanyaan yang tepat untuk setiap warning: "Apakah wajar jika data ini tidak ada?" Jika wajar, tentukan default yang masuk akal dengan `??` atau `?->`. Jika tidak wajar, cari kenapa datanya hilang (validasi, query, relasi), lalu gagalkan dengan jelas lewat validasi, `firstOrFail()`, atau exception.

Saat menelusuri warning di log, mulailah dari baris dan file yang disebut di pesan, lalu tanyakan dari mana nilai itu berasal: dari request, dari query, dari API, atau dari variabel yang dibuat di atasnya. Perbaikan di titik asal data biasanya menyelesaikan beberapa warning sekaligus, sedangkan menambal di titik pemakaian hanya memindahkan masalah ke tempat lain.

Laravel juga punya mode ketat untuk menangkap kasus serupa lebih awal. Di `AppServiceProvider`:

```
use Illuminate\Database\Eloquent\Model;

public function boot(): void
{
    Model::shouldBeStrict(! $this->app->isProduction());
}
```


Mode ini antara lain melempar exception saat Anda mengakses atribut model yang tidak ikut di-select, yang di mode normal diam-diam bernilai null.

## Kesalahan umum

- Mengira `isset()` dan `array_key_exists()` sama.
- Menganggap `?->` melindungi dari variabel yang belum didefinisikan.
- Memakai `empty()` untuk mengecek keberadaan, padahal `empty()` juga menganggap `"0"`, `0`, dan `[]` sebagai kosong.
- Menekan warning dengan `@` atau `error_reporting`.
- Memberi default pada data yang seharusnya wajib.


## Ringkasan

- Indeks array opsional: `$arr['key'] ?? default`, `Arr::get()`, atau `$request->input()`.
- Objek yang mungkin null: `$obj?->prop`, `withDefault()`, atau `firstOrFail()`.
- Variabel: inisialisasi sebelum if/loop; di Blade gunakan `??`, `@isset`, dan `@props`.
- Data wajib: validasi dan gagalkan dengan jelas, jangan disembunyikan.


Untuk error Laravel lain yang sering muncul, lihat [kumpulan solusi error Laravel](https://dhiecoderweb.com/kumpulan-solusi-error-laravel-paling-sering-terjadi).

## FAQ

### Apakah warning ini berbahaya jika aplikasi tetap jalan?

Di PHP murni, skrip memang tetap jalan dan nilainya menjadi null. Bahayanya ada di logika: null bisa tersimpan ke database, dihitung sebagai nol, atau lolos dari pengecekan. Di Laravel, warning ini menjadi exception sehingga halaman langsung error.

### Apa beda ?? dan ?:?

`??` hanya memeriksa apakah nilai ada dan bukan null, dan tidak memicu warning untuk kunci yang tidak ada. `?:` (Elvis) memeriksa truthy/falsy, sehingga `0`, `""`, dan `"0"` juga diganti, dan tetap memicu warning jika kuncinya tidak ada.

### Kenapa setelah upgrade ke PHP 8 muncul banyak warning padahal kode tidak diubah?

Karena PHP 8.0 menaikkan level pesan-pesan ini dari Notice ke Warning. Bug-nya sudah ada sejak lama, hanya sebelumnya tidak terlihat. Perbaiki satu per satu mulai dari yang paling sering muncul di log.

### Apakah operator nullsafe tersedia di PHP 7?

Tidak. Operator `?->` baru ada sejak PHP 8.0. Di versi lama, gunakan pengecekan manual atau helper `optional()` milik Laravel, misalnya `optional($post->author)->name`.

### Apakah "Passing null to parameter" di PHP 8.1 masalah yang sama?

Mirip tetapi berbeda. Sejak PHP 8.1, mengirim `null` ke parameter fungsi bawaan yang tidak menerima null (misalnya `strlen(null)`) memicu Deprecated. Penyebabnya sering sama, yaitu nilai yang tidak diduga null, dan solusinya juga sama: beri default atau perbaiki sumber datanya.


---

<!-- Aktifkan tautan ini setelah artikel terbit di blog:
Artikel lengkap: https://dhiecoderweb.com/php-8-undefined-array-key-undefined-variable-solusi
-->

# SQLSTATE[23000] Integrity Constraint Violation Laravel

`SQLSTATE[23000]: Integrity constraint violation` berarti MySQL menolak query Anda karena akan melanggar aturan integritas tabel: nilai unik yang dobel, foreign key yang menunjuk ke data yang tidak ada, data induk yang masih punya anak, atau kolom NOT NULL yang diisi null. Kuncinya ada di angka setelah titik dua: **1062** (duplikat), **1452** (foreign key anak gagal), **1451** (induk tidak bisa dihapus), dan **1048** (kolom tidak boleh null). Setiap kode punya penyebab dan solusi yang berbeda.

Di Laravel, error ini muncul sebagai `Illuminate\Database\QueryException`, lengkap dengan query yang gagal:

```
SQLSTATE[23000]: Integrity constraint violation: 1062 Duplicate entry 'budi@contoh.com'
for key 'users.users_email_unique'
(Connection: mysql, SQL: insert into `users` (`name`, `email`, ...) values (...))

SQLSTATE[23000]: Integrity constraint violation: 1452 Cannot add or update a child row:
a foreign key constraint fails (`toko`.`orders`, CONSTRAINT `orders_user_id_foreign`
FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))

SQLSTATE[23000]: Integrity constraint violation: 1451 Cannot delete or update a parent row:
a foreign key constraint fails (`toko`.`orders`, CONSTRAINT `orders_user_id_foreign`
FOREIGN KEY (`user_id`) REFERENCES `users` (`id`))

SQLSTATE[23000]: Integrity constraint violation: 1048 Column 'title' cannot be null
```


Penting untuk diingat: error ini adalah database yang *bekerja dengan benar*. Constraint dibuat persis untuk menolak data seperti ini. Tugas Anda bukan mematikan constraint, tetapi mencegah data buruk sampai ke query, atau menangani penolakannya dengan rapi.

## 1062 Duplicate entry (paling sering)

Kode 1062 berarti Anda mencoba menyimpan nilai yang sudah ada di kolom berindeks unik atau primary key. Nama indeks di pesan error memberi tahu kolom mana: `users_email_unique` berarti kolom `email` di tabel `users`. Jika tertulis `for key 'PRIMARY'`, yang dobel adalah primary key.

### Penyebab umum

1. Form registrasi atau tambah data tanpa validasi `unique`.
2. Validasi `unique` pada form edit tidak mengabaikan baris yang sedang diedit.
3. Dua request bersamaan (klik ganda, retry, job paralel) yang sama-sama lolos pengecekan "sudah ada belum?" lalu sama-sama insert. Ini *race condition*.
4. Data yang di-soft delete masih menempati nilai unik.
5. Seeder atau import yang dijalankan dua kali, atau yang mengisi ID secara manual.


### Solusi 1: Validasi sebelum menyimpan

Validasi memberi pengguna pesan yang jelas di form, bukan halaman error 500:

```
use Illuminate\Validation\Rule;

// Saat membuat data baru
$request->validate([
    'email' => ['required', 'email', 'unique:users,email'],
]);

// Saat mengedit: abaikan baris milik user ini sendiri
$request->validate([
    'email' => ['required', 'email', Rule::unique('users', 'email')->ignore($user->id)],
]);
```


Aturan `unique` secara default memeriksa semua baris, termasuk yang sudah di-soft delete. Itu justru melindungi dari kasus nomor 4: pengguna mendapat pesan "email sudah dipakai" alih-alih error database. Jika Anda ingin akun yang dihapus bisa mendaftar ulang, putuskan apakah barisnya di-`restore()` atau dihapus permanen dengan `forceDelete()`.

### Solusi 2: firstOrCreate, updateOrCreate, dan createOrFirst

Validasi saja tidak menyelesaikan race condition, karena dua request bisa lolos validasi bersamaan. Untuk pola "ambil jika ada, buat jika belum", gunakan method Eloquent:

```
// Cari berdasarkan email; jika tidak ada, buat dengan data tambahan
$subscriber = Subscriber::firstOrCreate(
    ['email' => $email],
    ['name' => $name, 'source' => 'footer-form'],
);

// Cari berdasarkan kunci; perbarui jika ada, buat jika tidak
$setting = Setting::updateOrCreate(
    ['user_id' => $user->id, 'key' => 'theme'],
    ['value' => 'dark'],
);
```


Di Laravel versi terbaru, `firstOrCreate()` memakai `createOrFirst()` di dalamnya: jika insert gagal karena pelanggaran unik (misalnya request lain baru saja membuat baris yang sama), Laravel menangkapnya lalu mengambil baris yang sudah ada. Syaratnya, harus ada **indeks unik di database** pada kolom yang dicari. Tanpa indeks unik, tidak ada yang mencegah duplikat dan race condition tetap bisa menghasilkan dua baris.

```
Schema::table('settings', function (Blueprint $table) {
    $table->unique(['user_id', 'key']);
});
```


### Solusi 3: upsert untuk data massal

Untuk import atau sinkronisasi ratusan baris, `upsert()` menjalankan satu query `INSERT ... ON DUPLICATE KEY UPDATE`:

```
Product::upsert(
    [
        ['sku' => 'KAOS-01', 'name' => 'Kaos Polos', 'price' => 75000],
        ['sku' => 'KAOS-02', 'name' => 'Kaos Sablon', 'price' => 95000],
    ],
    uniqueBy: ['sku'],
    update: ['name', 'price'],
);
```


Perhatikan dua hal. Pertama, kolom di `uniqueBy` harus punya indeks unik atau primary key. Kedua, di MySQL dan MariaDB, argumen `uniqueBy` sebenarnya diabaikan: database selalu memakai semua indeks unik dan primary key tabel untuk mendeteksi baris yang sudah ada. Jika tabel punya lebih dari satu indeks unik, hasilnya bisa mengejutkan, jadi pastikan Anda tahu indeks apa saja yang ada.

### Solusi 4: Tangkap exception-nya

Di Laravel 10 versi terbaru dan Laravel 11, pelanggaran unik dilempar sebagai `UniqueConstraintViolationException` (turunan `QueryException`), sehingga mudah ditangani secara spesifik:

```
use Illuminate\Database\UniqueConstraintViolationException;

try {
    $user = User::create($data);
} catch (UniqueConstraintViolationException $e) {
    return back()->withErrors(['email' => 'Email ini sudah terdaftar.'])->withInput();
}
```


Di versi yang lebih lama, tangkap `QueryException` lalu periksa `$e->errorInfo[1] === 1062`.

## 1452 Cannot add or update a child row

Kode 1452 berarti Anda menyimpan baris "anak" (misalnya `orders`) dengan foreign key yang nilainya tidak ada di tabel induk (`users`). Pesan error menyebut constraint dan kolomnya, jadi Anda tahu persis nilai mana yang bermasalah.

### Penyebab umum

1. ID induk salah atau sudah dihapus, misalnya `category_id` dari form yang dimanipulasi atau cache lama.
2. Nilai `0` atau string kosong dipakai sebagai pengganti "tidak ada". Foreign key hanya menerima `NULL` untuk "tidak ada", dan hanya jika kolomnya nullable.
3. Urutan seeder salah: `OrderSeeder` berjalan sebelum `UserSeeder`.
4. Menambahkan foreign key lewat migrasi ke tabel yang sudah berisi data "yatim" (orphan).


### Solusi

Validasi keberadaan induk sebelum menyimpan:

```
$request->validate([
    'category_id' => ['nullable', 'integer', 'exists:categories,id'],
]);
```


Jika relasinya opsional, kolom harus nullable dan Anda harus mengirim `null`, bukan `0`:

```
$table->foreignId('category_id')->nullable()->constrained()->nullOnDelete();
```


Untuk seeder, panggil dalam urutan induk lalu anak di `DatabaseSeeder`, atau gunakan factory dengan relasi (`Order::factory()->for(User::factory())`) agar induk dibuat otomatis.

Jika migrasi penambahan foreign key gagal dengan 1452, cari baris yatim terlebih dahulu:

```
SELECT o.id, o.user_id
FROM orders o
LEFT JOIN users u ON u.id = o.user_id
WHERE o.user_id IS NOT NULL
  AND u.id IS NULL;
```


Lalu putuskan: hapus barisnya, set foreign key-nya ke `NULL`, atau pulihkan data induknya. Lakukan di database cadangan dulu jika ini data produksi. Jika konsep JOIN dan foreign key masih terasa asing, [panduan belajar MySQL untuk pemula](https://dhiecoderweb.com/panduan-belajar-mysql-untuk-pemula-dari-dasar-sampai-mahir) membahasnya dari dasar.

## 1451 Cannot delete or update a parent row

Kebalikan dari 1452: Anda mencoba menghapus baris induk (misalnya user) yang masih dirujuk baris anak (orders). Secara default, foreign key yang dibuat Laravel tanpa aturan `ON DELETE` berperilaku menolak penghapusan ini.

Pilih perilaku yang sesuai dengan aturan bisnis Anda, bukan yang paling cepat menghilangkan error:   Method di migrasiSQLEfek saat induk dihapusCocok untuk   `cascadeOnDelete()`ON DELETE CASCADEAnak ikut terhapusData yang tidak berarti tanpa induk: komentar milik post, item milik keranjang `nullOnDelete()`ON DELETE SET NULLForeign key anak jadi NULLRelasi opsional: post tetap ada walau kategorinya dihapus `restrictOnDelete()`ON DELETE RESTRICTPenghapusan ditolak (1451)Data penting: user yang punya transaksi, produk yang ada di invoice

```
Schema::create('comments', function (Blueprint $table) {
    $table->id();
    $table->foreignId('post_id')->constrained()->cascadeOnDelete();
    $table->foreignId('user_id')->nullable()->constrained()->nullOnDelete();
    $table->text('body');
    $table->timestamps();
});
```


Untuk mengubah aturan pada tabel yang sudah ada, buat migrasi baru yang menghapus foreign key lama lalu membuatnya ulang:

```
Schema::table('orders', function (Blueprint $table) {
    $table->dropForeign(['user_id']);
    $table->foreign('user_id')->references('id')->on('users')->restrictOnDelete();
});
```


Jika penolakan memang yang Anda inginkan (misalnya user dengan riwayat transaksi tidak boleh dihapus), tangani di aplikasi dengan pesan yang ramah, atau pertimbangkan soft delete untuk induknya:

```
if ($user->orders()->exists()) {
    return back()->with('error', 'User ini memiliki pesanan dan tidak bisa dihapus.');
}
$user->delete();
```


Dua catatan penting. Cascade di level database tidak memicu event model Eloquent (`deleting`, `deleted`) untuk baris anak, jadi observer dan pembersihan file tidak berjalan. Selain itu, soft delete pada induk hanya mengisi `deleted_at`, sehingga tidak memicu cascade database sama sekali.

## 1048 Column cannot be null

Kode 1048 berarti query mengirim `NULL` secara eksplisit ke kolom `NOT NULL`.

### Penyebab khas di Laravel

Laravel punya middleware global `ConvertEmptyStringsToNull`. Input form yang dikosongkan tidak tiba sebagai string kosong `""`, melainkan `null`. Jadi jika field "deskripsi" dikosongkan dan kolom `description` di database NOT NULL, Anda mendapat 1048, padahal secara visual pengguna hanya "tidak mengisi".

Pilih solusi sesuai maksud data:

```
// 1. Field memang wajib: tolak di validasi
$request->validate(['title' => ['required', 'string', 'max:255']]);

// 2. Field opsional: izinkan null di database
Schema::table('products', function (Blueprint $table) {
    $table->text('description')->nullable()->change();
});

// 3. Field opsional dengan default: isi sebelum menyimpan
$data['stock'] = $data['stock'] ?? 0;
```


Di Laravel 11, `->change()` tidak lagi membutuhkan paket `doctrine/dbal` (di Laravel 10 umumnya masih perlu). Namun, sejak Laravel 11 pula, `change()` harus menyertakan semua modifier kolom yang ingin dipertahankan (misalnya `default` atau `comment`), karena atribut yang tidak disebut akan hilang.

### Jangan tertukar dengan 1364

Jika kolom NOT NULL tanpa default *tidak disebut sama sekali* di query, MySQL (dalam strict mode, default di Laravel) memberi error lain: `SQLSTATE[HY000]: General error: 1364 Field 'title' doesn't have a default value`. Di Laravel, penyebab tersering adalah kolom tidak tercantum di `$fillable`, sehingga `Model::create()` diam-diam membuangnya dari query.

## Kesalahan umum

- Mematikan pengecekan foreign key (`SET FOREIGN_KEY_CHECKS=0` atau `Schema::disableForeignKeyConstraints()`) di kode aplikasi untuk "memperbaiki" 1452/1451. Ini hanya pantas untuk seeding atau maintenance terkontrol.
- Mengandalkan pengecekan `exists()` lalu `create()` tanpa indeks unik.
- Memakai `cascadeOnDelete()` pada data finansial atau historis.
- Mengirim `0` sebagai foreign key "kosong".
- Menjadikan semua kolom nullable demi menghilangkan 1048, sehingga data wajib bisa kosong.


## Ringkasan per kode

- **1062**: validasi `unique` (dengan `ignore()` saat edit), indeks unik di database, `firstOrCreate`/`updateOrCreate`/`upsert`, tangkap `UniqueConstraintViolationException`.
- **1452**: validasi `exists`, gunakan `null` bukan `0`, urutkan seeder, bersihkan baris yatim sebelum menambah FK.
- **1451**: pilih `cascadeOnDelete`, `nullOnDelete`, atau `restrictOnDelete` secara sadar; tangani penolakan dengan pesan ramah.
- **1048**: `required` untuk field wajib, `nullable()` atau default untuk field opsional; ingat `ConvertEmptyStringsToNull`.


Error Laravel lain yang sering ditemui dirangkum di [kumpulan solusi error Laravel](https://dhiecoderweb.com/kumpulan-solusi-error-laravel-paling-sering-terjadi).

## FAQ

### Bagaimana cara tahu kode error MySQL dari QueryException?

Properti `$e->errorInfo` berisi array `[SQLSTATE, kode driver, pesan]`. Jadi `$e->errorInfo[1]` berisi 1062, 1452, 1451, atau 1048. Jangan mengandalkan `$e->getCode()` untuk ini, karena isinya SQLSTATE seperti "23000".

### Kenapa firstOrCreate tetap menghasilkan duplikat?

Hampir selalu karena tidak ada indeks unik di kolom pencarian. Tanpa indeks unik, database tidak punya alasan menolak baris kedua, sehingga dua request bersamaan bisa sama-sama membuat data.

### Apakah aman menonaktifkan foreign key saat seeding?

Untuk seeding di lingkungan pengembangan, misalnya saat melakukan truncate tabel yang saling terhubung, ini umum dilakukan dengan `Schema::disableForeignKeyConstraints()` lalu `Schema::enableForeignKeyConstraints()`. Jangan lakukan ini di alur aplikasi produksi.

### Kenapa error duplikat muncul padahal data dengan email itu tidak terlihat?

Kemungkinan barisnya di-soft delete. Query Eloquent biasa menyembunyikan baris dengan `deleted_at` terisi, tetapi indeks unik di database tetap menghitungnya. Cek dengan `User::withTrashed()->where('email', $email)->first()`.

### Lebih baik cascade atau restrict?

Tergantung arti datanya. Gunakan cascade untuk data yang tidak punya makna tanpa induknya, dan restrict untuk data yang harus tetap utuh demi catatan atau audit. Jika ragu, mulai dari restrict: lebih mudah melonggarkan aturan daripada mengembalikan data yang sudah ikut terhapus.


---

<!-- Aktifkan tautan ini setelah artikel terbit di blog:
Artikel lengkap: https://dhiecoderweb.com/sqlstate-23000-integrity-constraint-violation-laravel
-->

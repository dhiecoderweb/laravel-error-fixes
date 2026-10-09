# Error SMTP Laravel: Connection Could Not Be Established

Error `Connection could not be established with host` berarti aplikasi Laravel Anda bahkan tidak berhasil membuka koneksi ke server SMTP: biasanya karena host atau port salah, kombinasi port dan enkripsi tidak cocok, atau hosting memblokir koneksi SMTP keluar. Sementara `Failed to authenticate on SMTP server` berarti koneksi sudah tersambung, tetapi username/password ditolak, paling sering karena memakai password akun biasa alih-alih App Password, atau karena `.env` yang terbaca bukan yang Anda kira (config cache). Artikel ini membantu Anda memastikan error mana yang terjadi lalu memperbaikinya satu per satu.

Dua pesan error yang dibahas di sini biasanya muncul seperti ini di `storage/logs/laravel.log`:

```
Symfony\Component\Mailer\Exception\TransportException:
Connection could not be established with host "ssl://smtp.gmail.com:465":
stream_socket_client(): Unable to connect to ssl://smtp.gmail.com:465 (Connection timed out)
```


```
Symfony\Component\Mailer\Exception\TransportException:
Failed to authenticate on SMTP server with username "anda@gmail.com" using the following
authenticators: "LOGIN", "PLAIN", "XOAUTH2". Authenticator "LOGIN" returned "Expected
response code "235" but got code "535", with message "535-5.7.8 Username and Password
not accepted.
```


Jika Anda belum pernah mengatur email di Laravel sama sekali, mulailah dari tutorial [cara mengirim email Laravel dengan SMTP Gmail](https://dhiecoderweb.com/cara-mengirim-email-laravel-smtp-gmail). Artikel ini khusus untuk troubleshooting ketika konfigurasi sudah ada tetapi email tetap gagal terkirim.

## Bedakan dulu: gagal konek atau gagal login?

Kedua error ini terjadi di tahap yang berbeda, dan solusinya pun berbeda.   PesanTahapArah pemeriksaan   Connection could not be established with hostMembuka koneksi TCP/TLSHost, port, enkripsi, firewall, blokir hosting, sertifikat Failed to authenticate on SMTP serverLogin SMTP (AUTH)Username, password, App Password, karakter khusus di `.env`, config cache

Perhatikan juga bagian di dalam kurung pada pesan "Connection could not be established". `Connection timed out` umumnya berarti paket diblokir di jalan (firewall atau hosting). `Connection refused` berarti server tujuan aktif tetapi port itu tidak menerima koneksi. `php_network_getaddresses: getaddrinfo failed` berarti nama host salah ketik atau DNS server tidak bisa me-resolve-nya.

## Penyebab 1: Port dan enkripsi tidak cocok

Ada dua cara server SMTP menyediakan enkripsi, dan masing-masing punya port standar:

- **Port 465 — TLS implisit (SMTPS).** Koneksi langsung dibuka dalam keadaan terenkripsi sejak awal.
- **Port 587 — STARTTLS.** Koneksi dibuka polos, lalu klien meminta upgrade ke TLS dengan perintah STARTTLS.


Jika klien mencoba TLS implisit ke port 587, atau berbicara polos ke port 465, kedua sisi saling menunggu dan koneksi berakhir dengan timeout atau error SSL seperti `wrong version number`.

### Laravel 9 dan 10 (MAIL_ENCRYPTION)

Sejak Laravel 9, mailer memakai Symfony Mailer. Di sini port sangat menentukan: port 465 otomatis dibuka dengan TLS implisit, sedangkan port lain dibuka polos lalu otomatis di-upgrade dengan STARTTLS jika server mendukungnya. Konfigurasi yang aman untuk Laravel 10:

```
# Pilihan A: STARTTLS
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_ENCRYPTION=tls

# Pilihan B: TLS implisit
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=465
MAIL_ENCRYPTION=ssl
```


Di Laravel 8 ke bawah (SwiftMailer), nilai `MAIL_ENCRYPTION` harus benar-benar cocok dengan port: `ssl` untuk 465 dan `tls` untuk 587. Banyak tutorial lama berasal dari era ini, jadi jangan heran jika sarannya terlihat lebih ketat.

### Laravel 11: MAIL_SCHEME menggantikan MAIL_ENCRYPTION

Di Laravel 11, file `config/mail.php` bawaan tidak lagi punya kunci `encryption`. Yang ada adalah `scheme`:

```
'smtp' => [
    'transport' => 'smtp',
    'scheme' => env('MAIL_SCHEME'),
    'url' => env('MAIL_URL'),
    'host' => env('MAIL_HOST', '127.0.0.1'),
    'port' => env('MAIL_PORT', 2525),
    'username' => env('MAIL_USERNAME'),
    'password' => env('MAIL_PASSWORD'),
    'timeout' => null,
    'local_domain' => env('MAIL_EHLO_DOMAIN', parse_url(env('APP_URL', 'http://localhost'), PHP_URL_HOST)),
],
```


Nilai yang valid untuk `MAIL_SCHEME` hanya `smtp` dan `smtps`. Jika dibiarkan kosong (`null`), Laravel memilih `smtps` untuk port 465 dan `smtp` untuk port lain. Jadi, untuk sebagian besar kasus, cukup biarkan kosong dan atur port dengan benar:

```
MAIL_MAILER=smtp
MAIL_SCHEME=null
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=anda@gmail.com
MAIL_PASSWORD=abcdefghijklmnop
MAIL_FROM_ADDRESS="anda@gmail.com"
MAIL_FROM_NAME="${APP_NAME}"
```


Kesalahan yang sering terjadi saat upgrade adalah mengisi `MAIL_SCHEME=tls` atau `MAIL_SCHEME=ssl` karena terbiasa dengan `MAIL_ENCRYPTION`. Hasilnya error seperti:

```
The "tls" scheme is not supported; supported schemes for mailer "smtp" are: "smtp", "smtps".
```


Kesalahan kedua adalah memaksa `MAIL_SCHEME=smtps` dengan port 587. Laravel akan mencoba TLS implisit ke port STARTTLS dan koneksi gagal. Jika Anda upgrade dari Laravel 10 dan masih memakai `config/mail.php` lama yang berisi kunci `encryption`, ketahuilah bahwa framework Laravel 11 tidak lagi membaca kunci tersebut saat membuat transport SMTP; yang dipakai adalah `scheme` dan port.

## Penyebab 2: Hosting atau cloud memblokir SMTP keluar

Ini penyebab utama jika konfigurasi yang sama berjalan lancar di laptop tetapi gagal di server. Banyak penyedia hosting dan cloud membatasi koneksi keluar ke port SMTP untuk mencegah spam. Di shared hosting berbasis cPanel/WHM, fitur seperti "SMTP Restrictions" bisa membuat hanya server mail lokal yang boleh dipakai, sehingga koneksi ke `smtp.gmail.com` gagal. Beberapa penyedia VPS juga memblokir port SMTP untuk akun baru sampai Anda mengajukan permintaan pembukaan.

Uji langsung dari server (bukan dari laptop Anda) lewat SSH:

```
# apakah port bisa dijangkau?
nc -vz smtp.gmail.com 587
nc -vz smtp.gmail.com 465

# uji handshake TLS implisit (port 465)
openssl s_client -connect smtp.gmail.com:465 -crlf -quiet

# uji STARTTLS (port 587)
openssl s_client -starttls smtp -connect smtp.gmail.com:587 -crlf -quiet
```


Jika `nc` tidak pernah selesai lalu timeout, port tersebut diblokir. Pilihan Anda:

- Coba port lain (misalnya 465 jika 587 diblokir, atau sebaliknya). Beberapa layanan email transaksional juga menyediakan port alternatif seperti 2525.
- Pakai server SMTP milik hosting (biasanya `mail.domainanda.com`) dengan akun email yang dibuat di cPanel.
- Pakai mailer berbasis API (Mailgun, Postmark, Amazon SES, Resend) yang mengirim lewat HTTPS port 443, sehingga tidak terpengaruh blokir port SMTP.
- Hubungi dukungan hosting dan minta port SMTP keluar dibuka.


## Penyebab 3: Password ditolak (535 / Failed to authenticate)

### Gmail dan Google Workspace butuh App Password

Google tidak lagi menerima login SMTP dengan password akun biasa. Anda harus mengaktifkan 2-Step Verification, lalu membuat App Password di pengaturan keamanan akun Google. App Password berupa 16 karakter yang ditampilkan dalam empat kelompok; tulis di `.env` tanpa spasi. Jika akun Workspace Anda dikelola admin, App Password bisa saja dinonaktifkan oleh kebijakan organisasi, dan Anda perlu meminta admin mengizinkannya atau memakai SMTP relay Workspace.

Provider lain punya aturan serupa. Microsoft 365, misalnya, bisa menonaktifkan SMTP AUTH per mailbox atau per tenant. Jika provider mengembalikan kode 535 dengan pesan yang menyebut kebijakan atau autentikasi dinonaktifkan, masalahnya ada di pengaturan akun, bukan di Laravel.

### Karakter khusus di .env

Di file `.env`, tanda `#` tanpa tanda kutip dianggap awal komentar. Password `Rahasia#2024` akan terbaca sebagai `Rahasia`. Spasi dan tanda `$` juga bisa bermasalah. Bungkus password dengan tanda kutip tunggal agar dibaca apa adanya:

```
MAIL_PASSWORD='Rahasia#2024$abc'
```


Pastikan juga `MAIL_USERNAME` berisi alamat email lengkap jika provider memintanya (Gmail, cPanel), bukan hanya bagian sebelum tanda `@`.

## Penyebab 4: Config cache membuat perubahan .env tidak terbaca

Jika di server pernah dijalankan `php artisan config:cache` atau `php artisan optimize`, Laravel membaca konfigurasi dari `bootstrap/cache/config.php`, bukan dari `.env`. Anda bisa mengubah `.env` seratus kali tanpa efek apa pun. Cek nilai yang benar-benar dipakai:

```
php artisan config:clear
php artisan tinker
>>> config('mail.mailers.smtp')
>>> config('mail.default')
```


Jika email dikirim lewat antrean (`Mail::queue()` atau Mailable yang mengimplementasikan `ShouldQueue`), worker antrean menyimpan konfigurasi lama di memori sejak dijalankan. Setelah mengubah konfigurasi, jalankan:

```
php artisan queue:restart
```


Ini juga menjelaskan kasus aneh "email dari Tinker terkirim, tetapi email dari aplikasi gagal": Tinker membaca konfigurasi baru, sedangkan worker masih memakai yang lama.

## Penyebab 5: Sertifikat SSL tidak cocok dengan nama host

Di shared hosting, Anda mungkin memakai `MAIL_HOST=mail.domainanda.com`, padahal sertifikat SSL di server mail diterbitkan untuk nama host server (misalnya `server123.namahosting.com`). Hasilnya:

```
stream_socket_enable_crypto(): SSL operation failed with code 1. OpenSSL Error messages:
error:0A000086:SSL routines::certificate verify failed
```


Solusi yang benar adalah memakai nama host yang sesuai dengan sertifikat (tanyakan ke hosting atau lihat di menu "Connect Devices"/"Email Accounts" di cPanel). Sebagai upaya sementara saat debugging, Anda bisa menonaktifkan verifikasi sertifikat di `config/mail.php`, tetapi ini mengurangi keamanan dan sebaiknya tidak dibiarkan di produksi:

```
'smtp' => [
    // ...
    'verify_peer' => false,
],
```


## Cara menguji tanpa menebak-nebak

Kirim email uji langsung dari Tinker sehingga error muncul di terminal, bukan tersembunyi di antrean:

```
php artisan tinker
>>> Mail::raw('Tes SMTP', fn ($m) => $m->to('anda@contoh.com')->subject('Tes'));
```


Setelah email terkirim, jangan berhenti di situ. Email yang terkirim belum tentu masuk inbox. Pastikan domain pengirim punya SPF, DKIM, dan DMARC yang benar. Anda bisa memeriksanya dengan [SPF Checker](https://dhiecoderweb.com/tools/spf-checker) dan [DKIM Checker](https://dhiecoderweb.com/tools/dkim-checker). Jika email ditolak Gmail dengan kode 550, baca [cara mengatasi error Gmail 550 5.7.26](https://dhiecoderweb.com/cara-mengatasi-error-gmail-550-5-7-26-unauthenticated).

## Kesalahan umum

- Mengisi `MAIL_SCHEME=tls` di Laravel 11.
- Memaksa `smtps` dengan port 587.
- Menguji koneksi dari laptop, padahal yang diblokir adalah server produksi.
- Memakai password akun Gmail biasa, bukan App Password.
- Password berisi `#` tanpa tanda kutip.
- Lupa `config:clear` dan `queue:restart` setelah mengubah `.env`.
- `MAIL_FROM_ADDRESS` berbeda domain dengan akun SMTP, sehingga provider menolak atau menulis ulang pengirim.


## Checklist

1. Baca pesan lengkap di `storage/logs/laravel.log`: gagal konek atau gagal login?
2. Port 465 dengan TLS implisit, port 587 dengan STARTTLS; di Laravel 11 biarkan `MAIL_SCHEME` kosong.
3. Uji `nc -vz` dan `openssl s_client` dari server.
4. Gunakan App Password untuk Gmail/Workspace.
5. Bungkus password dengan tanda kutip tunggal.
6. `php artisan config:clear` dan `php artisan queue:restart`.
7. Uji kirim dari Tinker, lalu cek SPF/DKIM/DMARC.


## FAQ

### Apakah MAIL_ENCRYPTION masih dipakai di Laravel 11?

Tidak untuk transport SMTP. Laravel 11 memakai `MAIL_SCHEME` (`smtp` atau `smtps`) dan, jika kosong, menentukan skema dari port. Membiarkan `MAIL_ENCRYPTION` di `.env` tidak menimbulkan error, tetapi juga tidak berpengaruh.

### Lebih baik port 465 atau 587?

Keduanya aman jika dikonfigurasi benar. Port 587 dengan STARTTLS adalah port submission yang paling umum; port 465 dengan TLS implisit juga didukung luas. Pilih yang tidak diblokir oleh hosting Anda dan didukung provider email.

### Kenapa email jalan di lokal tapi gagal di server?

Penyebab paling umum adalah blokir port SMTP keluar di server, `.env` server yang berbeda, atau config cache. Uji koneksi dengan `nc` dari server, lalu periksa `config('mail.mailers.smtp')` lewat Tinker.

### Saya sudah pakai App Password tapi tetap 535. Apa lagi yang dicek?

Pastikan App Password ditulis tanpa spasi, username adalah alamat email lengkap akun yang membuat App Password tersebut, config cache sudah dibersihkan, dan worker antrean sudah di-restart. Untuk akun Workspace, tanyakan ke admin apakah App Password atau SMTP AUTH diizinkan.

### Apakah aman menonaktifkan verify_peer?

Hanya untuk diagnosis singkat. Menonaktifkan verifikasi sertifikat membuka peluang serangan man-in-the-middle. Perbaikan permanennya adalah memakai nama host yang cocok dengan sertifikat server mail.


---

<!-- Aktifkan tautan ini setelah artikel terbit di blog:
Artikel lengkap: https://dhiecoderweb.com/error-smtp-laravel-connection-could-not-be-established
-->

# Authentication di Laravel

Laravel menyediakan sistem authentication yang **lengkap, aman, dan mudah digunakan**. Alih-alih menulis kode authentication dari nol, Laravel sudah menyiapkan berbagai komponen siap pakai.

## Arsitektur Authentication Laravel

Authentication Laravel dibangun di atas dua konsep utama:

| Konsep | Penjelasan |
|---|---|
| **Guards** | Mendefinisikan **bagaimana** user diautentikasi di setiap request. Contoh: `session` guard (menyimpan state via session & cookie). |
| **Providers** | Mendefinisikan **dari mana** data user diambil (database, Eloquent, API eksternal, dll). |

Konfigurasi authentication Laravel ada di file `config/auth.php`.

## Authentication Bawaan (Browser)

Untuk aplikasi web monolith (Laravel tradisional), Laravel menggunakan **session-based authentication**:

1. User login via form → Laravel verifikasi kredensial
2. Jika cocok, data authentication disimpan di **session**
3. Browser mendapat **cookie** berisi session ID
4. Setiap request berikutnya, Laravel mencocokkan session ID dan menganggap user "terautentikasi"

Contoh implementasi manual:

```php
use Illuminate\Support\Facades\Auth;

if (Auth::attempt(['email' => $email, 'password' => $password])) {
    $request->session()->regenerate();
    return redirect()->intended('dashboard');
}
```

## Starter Kits

Laravel menyediakan **starter kits** (seperti Laravel Breeze & Laravel Jetstream) yang langsung membangun seluruh sistem authentication—registrasi, login, lupa password, verifikasi email—dalam hitungan menit:

```bash
composer require laravel/breeze --dev
php artisan breeze:install
php artisan migrate
```

## Authentication API (Token-Based)

Untuk aplikasi yang menyediakan API, Laravel menawarkan dua paket:

| Paket | Kegunaan |
|---|---|
| **Sanctum** ✅ *(direkomendasikan)* | Cocok untuk SPA, mobile apps, dan API token sederhana. Hybrid: otomatis pakai cookie untuk web browser dan token untuk API. |
| **Passport** | OAuth2 penuh. Cocok jika benar-benar membutuhkan semua fitur spesifikasi OAuth2 (seperti third-party client). |

## Melindungi Route

Cukup tambahkan middleware `auth` ke route yang ingin dilindungi:

```php
Route::get('/dashboard', function () {
    // Hanya user terautentikasi yang bisa akses
})->middleware('auth');
```

## Ringkasan Stack Authentication Laravel

| Jenis Aplikasi | Stack yang Direkomendasikan |
|---|---|
| Monolith (web browser) | Built-in Auth + Starter Kit |
| Monolith + API | Built-in Auth + **Sanctum** |
| SPA terpisah + Laravel backend | **Sanctum** + manual auth routes / **Fortify** |
| API untuk third-party | **Sanctum** (sederhana) atau **Passport** (OAuth2 penuh) |

> **Intinya**: Laravel menggunakan **session-based auth** untuk web browser dan **token-based auth** (Sanctum/Passport) untuk API, semuanya sudah terintegrasi dan siap pakai.

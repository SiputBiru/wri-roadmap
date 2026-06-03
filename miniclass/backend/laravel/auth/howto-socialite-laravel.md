# Cara Implementasi Laravel Socialite

> **Panduan langkah demi langkah** implementasi Socialite — OAuth login via provider eksternal (Google, GitHub, Facebook, Apple, dll).

---

## Instalasi Socialite

```bash
composer require laravel/socialite
```

---

## Konfigurasi Provider

### Daftarkan OAuth App

Buat OAuth App di masing-masing provider:

| Provider | Dashboard |
|----------|-----------|
| **Google** | https://console.cloud.google.com/apis/credentials |
| **GitHub** | https://github.com/settings/developers |
| **Facebook** | https://developers.facebook.com/apps |
| **Apple** | https://developer.apple.com/account/resources/identifiers/list |

Isi **Redirect URI** dengan: `http://localhost:8000/auth/{provider}/callback`

### Set Credentials di `.env`

```env
# Google
GOOGLE_CLIENT_ID=xxxxxxxxx
GOOGLE_CLIENT_SECRET=xxxxxxxxx

# GitHub
GITHUB_CLIENT_ID=xxxxxxxxx
GITHUB_CLIENT_SECRET=xxxxxxxxx

# Facebook
FACEBOOK_CLIENT_ID=xxxxxxxxx
FACEBOOK_CLIENT_SECRET=xxxxxxxxx
```

### Daftarkan di `config/services.php`

```php
<?php

return [
    // ... existing services

    'google' => [
        'client_id' => env('GOOGLE_CLIENT_ID'),
        'client_secret' => env('GOOGLE_CLIENT_SECRET'),
        'redirect' => env('GOOGLE_REDIRECT_URI', '/auth/google/callback'),
    ],

    'github' => [
        'client_id' => env('GITHUB_CLIENT_ID'),
        'client_secret' => env('GITHUB_CLIENT_SECRET'),
        'redirect' => '/auth/github/callback',
    ],

    'facebook' => [
        'client_id' => env('FACEBOOK_CLIENT_ID'),
        'client_secret' => env('FACEBOOK_CLIENT_SECRET'),
        'redirect' => '/auth/facebook/callback',
    ],
];
```

---

## Migration & Model Social Account

### Migration

```bash
php artisan make:migration create_social_accounts_table
```

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('social_accounts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('provider');       // 'google', 'github', 'facebook'
            $table->string('provider_id');    // ID user dari provider
            $table->string('avatar')->nullable();
            $table->json('meta')->nullable();  // data tambahan dari provider
            $table->timestamps();

            $table->unique(['provider', 'provider_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('social_accounts');
    }
};
```

Jalankan:
```bash
php artisan migrate
```

### Model `SocialAccount`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class SocialAccount extends Model
{
    protected $fillable = [
        'user_id',
        'provider',
        'provider_id',
        'avatar',
        'meta',
    ];

    protected function casts(): array
    {
        return [
            'meta' => 'array',
        ];
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

### Relasi di Model `User`

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Database\Eloquent\Relations\HasMany;

class User extends Authenticatable
{
    // ... trait lainnya

    public function socialAccounts(): HasMany
    {
        return $this->hasMany(SocialAccount::class);
    }
}
```

---

## Route

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Auth\SocialiteController;

// Redirect ke provider OAuth
Route::get('/auth/{provider}/redirect', [SocialiteController::class, 'redirect']);

// Callback dari provider
Route::get('/auth/{provider}/callback', [SocialiteController::class, 'callback']);
```

---

## Controller

```php
<?php

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Models\SocialAccount;
use App\Models\User;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Str;
use Laravel\Socialite\Facades\Socialite;

class SocialiteController extends Controller
{
    /**
     * Redirect user ke provider OAuth.
     */
    public function redirect(string $provider)
    {
        // Validasi provider
        if (!in_array($provider, ['google', 'github', 'facebook', 'apple'])) {
            return redirect('/login')->with('error', 'Provider tidak didukung.');
        }

        return Socialite::driver($provider)->redirect();
    }

    /**
     * Handle callback dari provider OAuth.
     */
    public function callback(string $provider)
    {
        try {
            $socialUser = Socialite::driver($provider)->user();
        } catch (\Exception $e) {
            return redirect('/login')->with('error', 'Gagal mendapatkan data dari ' . $provider);
        }

        // Cek apakah social account sudah terdaftar
        $socialAccount = SocialAccount::where('provider', $provider)
            ->where('provider_id', $socialUser->getId())
            ->first();

        if ($socialAccount) {
            // Social account sudah ada → tinggal login
            Auth::login($socialAccount->user);
            return redirect('/dashboard');
        }

        // Cek apakah user dengan email ini sudah terdaftar
        $user = User::where('email', $socialUser->getEmail())->first();

        if (!$user) {
            // User baru → buat akun baru
            $user = User::create([
                'name' => $socialUser->getName() ?? $socialUser->getNickname(),
                'email' => $socialUser->getEmail(),
                'password' => bcrypt(Str::password(16)),
            ]);
        }

        // Link social account ke user
        $user->socialAccounts()->create([
            'provider' => $provider,
            'provider_id' => $socialUser->getId(),
            'avatar' => $socialUser->getAvatar(),
            'meta' => $socialUser->getRaw(),
        ]);

        Auth::login($user);
        return redirect('/dashboard');
    }
}
```

### Data yang Tersedia dari Socialite

```php
$user = Socialite::driver('google')->user();

// Data dasar
$user->getId();          // ID user dari provider
$user->getNickname();    // Username/nickname
$user->getName();        // Full name
$user->getEmail();       // Email
$user->getAvatar();      // URL avatar (ukuran default)

// Data mentah (semua data dari provider)
$user->getRaw();         // Array lengkap dari response provider

// Token
$user->token;            // Access token
$user->refreshToken;     // Refresh token (kalau ada)
$user->expiresIn;        // Expiry timestamp
```

---

## Menambahkan Scopes Opsional

Beberapa provider butuh scope tambahan untuk mengakses field tertentu:

```php
// Di method redirect()
return Socialite::driver('google')
    ->scopes(['profile', 'email', 'https://www.googleapis.com/auth/user.phonenumbers.read'])
    ->redirect();
```

Atau via `with()` untuk parameter arbitrary:

```php
return Socialite::driver('facebook')
    ->with(['auth_type' => 'rerequest'])  // Force re-consent
    ->redirect();
```

---

## Stateless Mode (Stateless)

Untuk API / frontend SPA yang handle redirect sendiri:

```php
$socialUser = Socialite::driver('google')->stateless()->user();
```

Tanpa stateless, Socialite otomatis menyimpan state di session untuk CSRF protection. Mode stateless **tidak pakai session** — cocok untuk API.

> ⚠️ **Catatan**: Stateless mode tanpa validasi state rawan serangan CSRF. Pastikan ada mekanisme proteksi sendiri (misal: `state` param dikirim & diverifikasi manual oleh frontend).

---

## Cheatsheet Socialite

### 📦 Instalasi

```bash
composer require laravel/socialite
```

### ⚙️ Konfigurasi (`config/services.php`)

```php
'google' => [
    'client_id' => env('GOOGLE_CLIENT_ID'),
    'client_secret' => env('GOOGLE_CLIENT_SECRET'),
    'redirect' => '/auth/google/callback',
],
```

### 📋 Migration

```php
Schema::create('social_accounts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('provider');
    $table->string('provider_id');
    $table->string('avatar')->nullable();
    $table->json('meta')->nullable();
    $table->timestamps();
    $table->unique(['provider', 'provider_id']);
});
```

### 🧩 Alur Controller

| Step | Method | Deskripsi |
|------|--------|-----------|
| Redirect | `Socialite::driver('google')->redirect()` | Arahkan user ke halaman OAuth provider |
| Callback | `Socialite::driver('google')->user()` | Ambil data user dari callback |
| Login/Create | Cek `social_accounts` → Cek email → Buat user baru | Autentikasi atau registrasi |

### 🔑 Data User dari Provider

| Method | Field |
|--------|-------|
| `getId()` | ID unik provider |
| `getEmail()` | Email |
| `getName()` | Nama lengkap |
| `getNickname()` | Username |
| `getAvatar()` | URL avatar |
| `getRaw()` | Semua data mentah |

### 🛡️ Mode

```php
// Default (dengan session CSRF)
Socialite::driver('google')->user();

// Stateless — untuk API / tanpa session
Socialite::driver('google')->stateless()->user();
```

---

> **Panduan Konseptual**: Untuk penjelasan tentang apa itu OAuth, SSO, dan bagaimana Socialite bekerja secara konseptual, lihat file [`teknologi-auth.md`](./teknologi-auth.md).

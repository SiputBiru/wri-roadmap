# Cara Implementasi Laravel Fortify

> **Panduan langkah demi langkah** implementasi Fortify — headless authentication backend untuk Laravel. Fortify menyediakan semua route & logic auth (login, register, reset password, 2FA) tanpa UI — kamu yang bikin tampilannya sendiri.

---

## Instalasi Fortify

```bash
composer require laravel/fortify
php artisan fortify:install
php artisan migrate
```

Perintah `fortify:install` akan:
- Membuat `app/Actions/Fortify/` — action classes (CreateNewUser, ResetUserPassword, dll) ✅
- Membuat `app/Providers/FortifyServiceProvider.php` ✅
- Menerbitkan `config/fortify.php` ✅
- Membuat migration database ✅

---

## Konfigurasi Features

Di `config/fortify.php`, atur fitur yang ingin diaktifkan:

```php
<?php

use Laravel\Fortify\Features;

'features' => [
    Features::registration(),
    Features::resetPasswords(),
    Features::emailVerification(),
    Features::twoFactorAuthentication([
        'confirm' => true,
        'confirmPassword' => true,
    ]),
    Features::passkeys([
        'confirmPassword' => true,
    ]),
],
```

| Feature | Fungsi |
|---------|--------|
| `registration()` | Registrasi user baru |
| `resetPasswords()` | Lupa password & reset |
| `emailVerification()` | Verifikasi email |
| `twoFactorAuthentication()` | 2FA dengan TOTP (Google Authenticator, dll) |
| `passkeys()` | Login tanpa password (Face ID, Touch ID, Windows Hello) |

Untuk menonaktifkan fitur, tinggal koment atau hapus dari array.

### Disable Views Mode

Jika Fortify dipasangkan dengan SPA (React/Vue), kamu bisa matikan route view-nya:

```php
// config/fortify.php
'views' => false,
```

---

## Membuat View (Frontend)

Karena Fortify tidak menyediakan UI, kamu harus bikin view sendiri. Semua view dikustomisasi via closure di `FortifyServiceProvider`.

### Login View

```php
<?php

namespace App\Providers;

use Laravel\Fortify\Fortify;
use Illuminate\Support\ServiceProvider;

class FortifyServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Fortify::loginView(function () {
            return view('auth.login');
        });
    }
}
```

**View `resources/views/auth/login.blade.php`:**

```html
<form method="POST" action="/login">
    @csrf
    <input type="email" name="email" placeholder="Email" required>
    <input type="password" name="password" placeholder="Password" required>
    <input type="checkbox" name="remember" id="remember">
    <label for="remember">Ingat saya</label>
    <button type="submit">Login</button>
</form>

@if ($errors->any())
    <div>{{ $errors->first() }}</div>
@endif
```

### Register View

```php
Fortify::registerView(function () {
    return view('auth.register');
});
```

Form mengirim POST ke `/register` dengan field: `name`, `email`, `password`, `password_confirmation`.

### Forgot Password View

```php
Fortify::requestPasswordResetLinkView(function () {
    return view('auth.forgot-password');
});
```

Form mengirim POST ke `/forgot-password` dengan field `email`.

### Reset Password View

```php
Fortify::resetPasswordView(function (Request $request) {
    return view('auth.reset-password', ['request' => $request]);
});
```

Form mengirim POST ke `/reset-password` dengan field: `email`, `password`, `password_confirmation`, `token` (dari route).

### Email Verification View

```php
Fortify::verifyEmailView(function () {
    return view('auth.verify-email');
});
```

Tampilkan pesan "Cek email kamu untuk verifikasi". Sertakan tombol POST ke `/email/verification-notification` untuk kirim ulang.

### Password Confirmation View

```php
Fortify::confirmPasswordView(function () {
    return view('auth.confirm-password');
});
```

Form mengirim POST ke `/user/confirm-password` dengan field `password`.

### Two-Factor Challenge View

```php
Fortify::twoFactorChallengeView(function () {
    return view('auth.two-factor-challenge');
});
```

Form mengirim POST ke `/two-factor-challenge` dengan field `code` (TOTP) atau `recovery_code`.

---

## Custom Actions

Saat registrasi dan reset password, Fortify memanggil action classes. Semua ada di `app/Actions/Fortify/`.

### CreateNewUser

Ubah validasi atau tambah field registrasi:

```php
<?php

namespace App\Actions\Fortify;

use App\Models\User;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Validator;
use Laravel\Fortify\Contracts\CreatesNewUsers;

class CreateNewUser implements CreatesNewUsers
{
    public function create(array $input): User
    {
        Validator::make($input, [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'max:255', 'unique:users'],
            'phone' => ['required', 'string', 'max:20'], // custom field
            'password' => $this->passwordRules(),
        ])->validate();

        return User::create([
            'name' => $input['name'],
            'email' => $input['email'],
            'phone' => $input['phone'], // custom field
            'password' => Hash::make($input['password']),
        ]);
    }

    protected function passwordRules(): array
    {
        return ['required', 'string', 'min:8', 'confirmed'];
    }
}
```

### ResetUserPassword

```php
<?php

namespace App\Actions\Fortify;

use App\Models\User;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Validator;
use Laravel\Fortify\Contracts\ResetsUserPasswords;

class ResetUserPassword implements ResetsUserPasswords
{
    public function reset(User $user, array $input): void
    {
        Validator::make($input, [
            'password' => $this->passwordRules(),
        ])->validate();

        $user->forceFill([
            'password' => Hash::make($input['password']),
        ])->save();
    }

    protected function passwordRules(): array
    {
        return ['required', 'string', 'min:8', 'confirmed'];
    }
}
```

### Custom Authentication

Ganti logic autentikasi (misal login via username, bukan email):

```php
<?php

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Laravel\Fortify\Fortify;

public function boot(): void
{
    Fortify::authenticateUsing(function (Request $request) {
        $user = User::where('email', $request->email)->first();

        if ($user && Hash::check($request->password, $user->password)) {
            return $user;
        }

        return null;
    });
}
```

### Custom Redirect

Ubah redirect setelah login/logout:

```php
<?php

use Laravel\Fortify\Contracts\LogoutResponse;

public function register(): void
{
    $this->app->instance(LogoutResponse::class, new class implements LogoutResponse {
        public function toResponse($request)
        {
            return redirect('/goodbye');
        }
    });
}
```

Atur `home` di `config/fortify.php` untuk redirect setelah login:

```php
'home' => '/dashboard',
```

---

## Custom Authentication Pipeline

Fortify memproses login melalui pipeline. Kamu bisa kustomisasi:

```php
<?php

use Laravel\Fortify\Actions\AttemptToAuthenticate;
use Laravel\Fortify\Actions\CanonicalizeUsername;
use Laravel\Fortify\Actions\EnsureLoginIsNotThrottled;
use Laravel\Fortify\Actions\PrepareAuthenticatedSession;
use Laravel\Fortify\Actions\RedirectIfTwoFactorAuthenticatable;
use Laravel\Fortify\Features;
use Laravel\Fortify\Fortify;
use Illuminate\Http\Request;

Fortify::authenticateThrough(function (Request $request) {
    return array_filter([
        config('fortify.limiters.login') ? null : EnsureLoginIsNotThrottled::class,
        config('fortify.lowercase_usernames') ? CanonicalizeUsername::class : null,
        Features::enabled(Features::twoFactorAuthentication())
            ? RedirectIfTwoFactorAuthenticatable::class
            : null,
        AttemptToAuthenticate::class,
        PrepareAuthenticatedSession::class,
    ]);
});
```

### Rate Limiting

Fortify otomatis throttle percobaan login. Kustomisasi di `FortifyServiceProvider`:

```php
<?php

use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Cache\RateLimiting\Limit;

RateLimiter::for('login', function ($request) {
    return Limit::perMinute(5)->by($request->email . $request->ip());
});
```

---

## Two-Factor Authentication

### Persiapan Model

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Fortify\TwoFactorAuthenticatable;

class User extends Authenticatable
{
    use Notifiable, TwoFactorAuthenticatable;
}
```

### Enable 2FA

**1. User klik "Enable 2FA"** → POST ke `/user/two-factor-authentication`

**2. Tampilkan QR Code** (Blade):

```php
{{ $request->user()->twoFactorQrCodeSvg() }}
```

Atau via API (JavaScript):

```js
fetch('/user/two-factor-qr-code')
    .then(res => res.json())
    .then(data => console.log(data.svg));
```

**3. User input kode dari Google Authenticator** → POST ke `/user/confirmed-two-factor-authentication`

**4. Tampilkan Recovery Codes** (Blade):

```php
@foreach ((array) $request->user()->recoveryCodes() as $code)
    <code>{{ $code }}</code><br>
@endforeach
```

### Login dengan 2FA

Setelah login sukses, Fortify otomatis redirect ke `/two-factor-challenge`. User input kode TOTP atau recovery code.

### Disable 2FA

DELETE request ke `/user/two-factor-authentication`.

---

## Passkeys (WebAuthn)

Fortify mendukung login tanpa password menggunakan Face ID, Touch ID, Windows Hello, atau hardware security key.

### Persiapan Model

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Fortify\Contracts\PasskeyUser;
use Laravel\Fortify\PasskeyAuthenticatable;

class User extends Authenticatable implements PasskeyUser
{
    use Notifiable, PasskeyAuthenticatable;
}
```

### Konfigurasi

Di `config/fortify.php`:

```php
'passkeys' => [
    'relying_party_id' => parse_url(config('app.url'), PHP_URL_HOST),
    'allowed_origins' => [config('app.url')],
    'user_handle_secret' => config('app.key'),
    'timeout' => 60000,
],
```

### JavaScript Client

Install package:

```bash
npm install @laravel/passkeys
```

**Register passkey:**

```js
import { Passkeys } from "@laravel/passkeys";

await Passkeys.register({ name: "MacBook Pro" });
```

**Login dengan passkey:**

```js
await Passkeys.verify();
```

**Confirm password dengan passkey:**

```js
await Passkeys.verify({
    routes: {
        options: "/passkeys/confirm/options",
        submit: "/passkeys/confirm",
    },
});
```

### React / Vue / Svelte Helpers

```js
// React
import { usePasskeys } from "@laravel/passkeys/react";

// Vue
import { usePasskeys } from "@laravel/passkeys/vue";

// Svelte
import { usePasskeys } from "@laravel/passkeys/svelte";
```

### Endpoint Passkeys

| Method | Endpoint | Fungsi |
|--------|----------|--------|
| GET | `/passkeys/login/options` | Ambil challenge untuk login |
| POST | `/passkeys/login` | Login dengan passkey |
| GET | `/passkeys/confirm/options` | Ambil challenge untuk konfirmasi |
| POST | `/passkeys/confirm` | Konfirmasi password dengan passkey |
| GET | `/user/passkeys/options` | Ambil challenge untuk registrasi |
| POST | `/user/passkeys` | Register passkey baru |
| DELETE | `/user/passkeys/{passkey}` | Hapus passkey |

---

## Fortify + Sanctum (Stack SPA)

Ini stack paling umum untuk SPA (Next.js, Vue, React) + Laravel backend:

| Komponen | Tugas |
|----------|-------|
| **Fortify** | Route & logic: register, login, reset password, verifikasi email, 2FA |
| **Sanctum** | Session auth (cookie) untuk SPA, API token untuk mobile |
| **Frontend SPA** | UI auth sendiri (React/Vue) + konsumsi route Fortify |

### Konfigurasi

```php
// config/fortify.php — disable views karena SPA handle UI sendiri
'views' => false,
```

```php
// bootstrap/app.php — aktifkan stateful API Sanctum
->withMiddleware(function (Middleware $middleware): void {
    $middleware->statefulApi();
})
```

### Alur Auth SPA + Fortify + Sanctum

```
SPA (React)                    Laravel (Fortify + Sanctum)
   │                                   │
   │── GET /sanctum/csrf-cookie ──────→│ CSRF protection
   │←── Set cookie XSRF-TOKEN ─────────│
   │                                   │
   │── POST /register ────────────────→│ Fortify: buat user
   │   (name, email, password)         │ Sanctum: session cookie
   │←── 201 + Set session cookie ──────│
   │                                   │
   │── POST /login ───────────────────→│ Fortify: auth
   │   (email, password)               │ Sanctum: session cookie
   │←── 200 + Set session cookie ──────│
   │                                   │
   │── GET /api/user ─────────────────→│ Sanctum: cek session
   │   (cookie otomatis)               │
   │←── { user: ... } ────────────────│
```

### Route Fortify (Yang Tersedia Otomatis)

| Method | Route | Deskripsi |
|--------|-------|-----------|
| GET | `/login` | Tampilkan form login |
| POST | `/login` | Login |
| POST | `/logout` | Logout |
| GET | `/register` | Tampilkan form registrasi |
| POST | `/register` | Registrasi |
| GET | `/forgot-password` | Form lupa password |
| POST | `/forgot-password` | Kirim link reset |
| GET | `/reset-password/{token}` | Form reset password |
| POST | `/reset-password` | Reset password |
| GET | `/email/verify` | Notifikasi verifikasi email |
| GET | `/email/verify/{id}/{hash}` | Verifikasi email |
| POST | `/email/verification-notification` | Kirim ulang verifikasi |
| GET | `/user/confirm-password` | Form konfirmasi password |
| POST | `/user/confirm-password` | Konfirmasi password |
| GET | `/two-factor-challenge` | Form 2FA |
| POST | `/two-factor-challenge` | Verifikasi 2FA |
| POST | `/user/two-factor-authentication` | Enable 2FA |
| DELETE | `/user/two-factor-authentication` | Disable 2FA |
| GET | `/user/two-factor-qr-code` | QR code 2FA |
| GET | `/user/two-factor-recovery-codes` | Recovery codes |
| POST | `/user/two-factor-recovery-codes` | Regenerate recovery codes |
| POST | `/user/confirmed-two-factor-authentication` | Confirm 2FA |

---

## Testing Fortify

### PHPUnit

```php
<?php

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class AuthenticationTest extends TestCase
{
    use RefreshDatabase;

    public function test_users_can_authenticate(): void
    {
        $user = User::factory()->create([
            'email' => 'user@example.com',
            'password' => bcrypt('password'),
        ]);

        $response = $this->post('/login', [
            'email' => 'user@example.com',
            'password' => 'password',
        ]);

        $this->assertAuthenticated();
        $response->assertRedirect('/dashboard');
    }

    public function test_users_can_not_authenticate_with_invalid_password(): void
    {
        $user = User::factory()->create([
            'email' => 'user@example.com',
            'password' => bcrypt('password'),
        ]);

        $this->post('/login', [
            'email' => 'user@example.com',
            'password' => 'wrong-password',
        ]);

        $this->assertGuest();
    }

    public function test_users_can_register(): void
    {
        $response = $this->post('/register', [
            'name' => 'Test User',
            'email' => 'test@example.com',
            'password' => 'password',
            'password_confirmation' => 'password',
        ]);

        $this->assertAuthenticated();
        $response->assertRedirect('/dashboard');
    }

    public function test_email_verification(): void
    {
        $user = User::factory()->create([
            'email_verified_at' => null,
        ]);

        $response = $this->actingAs($user)
            ->post('/email/verification-notification');

        $response->assertSessionHas('status', 'verification-link-sent');
    }

    public function test_protected_route_requires_authentication(): void
    {
        $response = $this->get('/dashboard');

        $response->assertRedirect('/login');
    }
}
```

### Pest

```php
<?php

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('users can authenticate', function () {
    $user = User::factory()->create([
        'email' => 'user@example.com',
        'password' => bcrypt('password'),
    ]);

    $response = $this->post('/login', [
        'email' => 'user@example.com',
        'password' => 'password',
    ]);

    $this->assertAuthenticated();
    $response->assertRedirect('/dashboard');
});

test('users cannot authenticate with invalid password', function () {
    $user = User::factory()->create([
        'email' => 'user@example.com',
        'password' => bcrypt('password'),
    ]);

    $this->post('/login', [
        'email' => 'user@example.com',
        'password' => 'wrong-password',
    ]);

    $this->assertGuest();
});

test('users can register', function () {
    $response = $this->post('/register', [
        'name' => 'Test User',
        'email' => 'test@example.com',
        'password' => 'password',
        'password_confirmation' => 'password',
    ]);

    $this->assertAuthenticated();
    $response->assertRedirect('/dashboard');
});

test('unauthenticated user is redirected to login', function () {
    $response = $this->get('/dashboard');

    $response->assertRedirect('/login');
});
```

---

## Cheatsheet Fortify

### 📦 Instalasi

```bash
composer require laravel/fortify
php artisan fortify:install
php artisan migrate
```

### ⚙️ Features (di `config/fortify.php`)

```php
'features' => [
    Features::registration(),
    Features::resetPasswords(),
    Features::emailVerification(),
    Features::twoFactorAuthentication(['confirm' => true, 'confirmPassword' => true]),
    Features::passkeys(['confirmPassword' => true]),
],
```

### 🎨 View Customization (di `FortifyServiceProvider`)

| Method | View | Route |
|--------|------|-------|
| `Fortify::loginView()` | `auth.login` | GET `/login` |
| `Fortify::registerView()` | `auth.register` | GET `/register` |
| `Fortify::requestPasswordResetLinkView()` | `auth.forgot-password` | GET `/forgot-password` |
| `Fortify::resetPasswordView()` | `auth.reset-password` | GET `/reset-password/{token}` |
| `Fortify::verifyEmailView()` | `auth.verify-email` | GET `/email/verify` |
| `Fortify::confirmPasswordView()` | `auth.confirm-password` | GET `/user/confirm-password` |
| `Fortify::twoFactorChallengeView()` | `auth.two-factor-challenge` | GET `/two-factor-challenge` |

### 🧩 Custom Actions (`app/Actions/Fortify/`)

| Class | Fungsi |
|-------|--------|
| `CreateNewUser` | Validasi & buat user baru |
| `ResetUserPassword` | Validasi & update password |
| `PasswordValidationRules` | Aturan validasi password |
| `Fortify::authenticateUsing()` | Custom logic autentikasi |

### 🛡️ 2FA Endpoints

| Method | Route | Fungsi |
|--------|-------|--------|
| POST | `/user/two-factor-authentication` | Enable 2FA |
| DELETE | `/user/two-factor-authentication` | Disable 2FA |
| POST | `/user/confirmed-two-factor-authentication` | Confirm 2FA |
| GET | `/user/two-factor-qr-code` | QR code SVG |
| GET | `/user/two-factor-recovery-codes` | Recovery codes |
| POST | `/user/two-factor-recovery-codes` | Regenerate codes |

### 🔑 Passkey Endpoints

| Method | Route | Fungsi |
|--------|-------|--------|
| GET | `/passkeys/login/options` | Challenge untuk login |
| POST | `/passkeys/login` | Login dengan passkey |
| POST | `/user/passkeys` | Register passkey baru |
| DELETE | `/user/passkeys/{passkey}` | Hapus passkey |

### 🧪 Testing

```php
// Auth user
$this->actingAs($user);

// Cek authenticated
$this->assertAuthenticated();
$this->assertGuest();

// Test register
$this->post('/register', [
    'name' => 'Test',
    'email' => 'test@example.com',
    'password' => 'password',
    'password_confirmation' => 'password',
])->assertRedirect('/dashboard');
```

### ⚙️ Konfigurasi Penting

| File | Konfigurasi |
|------|------------|
| `config/fortify.php` | `features`, `home`, `views`, `limiters` |
| `config/auth.php` | Guard `web` untuk session |
| `.env` | `APP_URL` (untuk passkey `relying_party_id`) |
| `app/Providers/FortifyServiceProvider.php` | View, actions, pipeline |

---

> **Panduan Konseptual**: Untuk penjelasan tentang apa itu Fortify, perbandingan dengan Sanctum, dan kapan menggunakannya, lihat file [`auth-dan-laravel.md`](./auth-dan-laravel.md).

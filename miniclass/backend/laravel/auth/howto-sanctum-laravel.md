# Cara Implementasi Laravel Sanctum

> **Panduan langkah demi langkah** implementasi Sanctum — hybrid authentication untuk API, SPA, dan mobile apps.

---

## Instalasi Sanctum

### Cara 1: Via Artisan (Laravel 13+) — Termudah

```bash
php artisan install:api
```

Perintah ini akan:
- Menginstal package Sanctum via Composer ✅
- Menerbitkan konfigurasi `config/sanctum.php` ✅
- Membuat migration tabel `personal_access_tokens` ✅
- Menambahkan middleware Sanctum ke `bootstrap/app.php` ✅

### Cara 2: Manual via Composer

```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

---

## Persiapan Model User

Tambahkan trait `HasApiTokens` ke model `User`:

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    // ... properti dan method lainnya
}
```

---

## API Token Authentication

> **Catatan**: Jangan gunakan API token untuk SPA first-party. Gunakan [SPA Authentication](#spa-authentication-cookie-based).

### Membuat Token

Buat endpoint untuk generate token:

```php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::post('/tokens/create', function (Request $request) {
    $token = $request->user()->createToken($request->token_name);

    return response()->json([
        'token' => $token->plainTextToken,
        'message' => 'Token berhasil dibuat'
    ]);
})->middleware('auth:sanctum');
```

**Penting:**
- Token di-hash SHA-256 sebelum disimpan di database
- `plainTextToken` hanya muncul **sekali** saat pembuatan — simpan baik-baik!
- Bisa diberi **abilities** untuk mengontrol akses

### Melindungi Route

Gunakan middleware `auth:sanctum`:

```php
<?php

use Illuminate\Http\Request;

Route::get('/user', function (Request $request) {
    return $request->user();
})->middleware('auth:sanctum');
```

Dari sisi client, kirim token di header:

```
Authorization: Bearer 1|abc123def456...
```

### Melihat Semua Token User

```php
// Di controller atau route
$tokens = $request->user()->tokens; // collection dari semua token

foreach ($tokens as $token) {
    echo $token->name . ' - ' . $token->created_at;
}
```

### Revoke Token

```php
// Revoke SEMUA token user
$request->user()->tokens()->delete();

// Revoke token yang sedang dipakai
$request->user()->currentAccessToken()->delete();

// Revoke token spesifik berdasarkan ID
$request->user()->tokens()->where('id', $tokenId)->delete();
```

### Token Expiration

**Global — atur di `config/sanctum.php`:**

```php
'expiration' => 525600, // dalam menit (1 tahun = 525.600 menit)
```

**Per-token — kasih argumen ketiga `createToken`:**

```php
use Carbon\Carbon;

$token = $user->createToken(
    'token-name',
    ['*'],
    Carbon::now()->addDays(7)  // expired 7 hari
);

return $token->plainTextToken;
```

**Prune otomatis token expired — di `routes/console.php`:**

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('sanctum:prune-expired --hours=24')->daily();
```

---

## Token Abilities (Scopes)

Sanctum mendukung "abilities" — mirip OAuth scopes — untuk mengontrol apa yang bisa dilakukan token.

### Membuat Token dengan Abilities

```php
// Token hanya bisa update server
$token = $user->createToken('deploy-token', ['server:update']);

// Token bisa create DAN update server
$token = $user->createToken('devops-token', ['server:create', 'server:update']);

// Token dengan semua abilities (super token)
$token = $user->createToken('admin-token', ['*']);
```

### Cek Abilities di Controller

```php
<?php

use Illuminate\Http\Request;

Route::put('/server/{id}', function (Request $request, $id) {
    $server = Server::findOrFail($id);

    if ($request->user()->tokenCan('server:update')) {
        $server->update($request->all());
        return response()->json($server);
    }

    return response()->json(['message' => 'Forbidden'], 403);
})->middleware('auth:sanctum');
```

Method yang tersedia:

| Method | Fungsi |
|--------|--------|
| `$user->tokenCan('ability')` | Cek apakah token punya ability |
| `$user->tokenCant('ability')` | Kebalikan dari `tokenCan` |

### Middleware Abilities

Daftarkan middleware abilities di `bootstrap/app.php`:

```php
<?php

use Laravel\Sanctum\Http\Middleware\CheckAbilities;
use Laravel\Sanctum\Http\Middleware\CheckForAnyAbility;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->alias([
        'abilities' => CheckAbilities::class,  // HARUS punya SEMUA
        'ability' => CheckForAnyAbility::class, // punya SETIDAKNYA SATU
    ]);
})
```

**Penggunaan di route:**

```php
// HARUS punya "check-status" DAN "place-orders"
Route::get('/orders', function () {
    // ...
})->middleware(['auth:sanctum', 'abilities:check-status,place-orders']);

// CUKUP punya salah satu: "check-status" ATAU "place-orders"
Route::get('/orders', function () {
    // ...
})->middleware(['auth:sanctum', 'ability:check-status,place-orders']);
```

### Kasus Khusus: First-Party SPA

Untuk request dari SPA first-party (via session cookie), method `tokenCan` akan **selalu return `true`**. Ini karena SPA sudah terautentikasi via session.

Namun, Anda tetap harus mengecek authorization policies:

```php
public function update(Request $request, Server $server)
{
    // Cek kepemilikan + ability
    return $request->user()->id === $server->user_id
        && $request->user()->tokenCan('server:update');
}
```

---

## SPA Authentication (Cookie-Based)

Untuk SPA yang terpisah dari backend Laravel (Next.js, Nuxt, React, Vue), Sanctum menggunakan **session cookie** — bukan token.

### Prasyarat

| Syarat | Keterangan |
|--------|-----------|
| **Domain** | SPA dan API harus share top-level domain yang sama (boleh beda subdomain) |
| **Headers** | Setiap request harus kirim `Accept: application/json` + `Referer` atau `Origin` |
| **Credentials** | Cookie harus dikirim (`withCredentials: true` di frontend) |

### Konfigurasi Backend

**Step 1 — Atur stateful domains di `config/sanctum.php`:**

```php
'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', sprintf(
    '%s%s',
    'localhost,localhost:3000,localhost:5173,127.0.0.1,127.0.0.1:8000,::1',
    Sanctum::currentApplicationUrlWithPort()
))),
```

Tambahkan di `.env`:
```
SANCTUM_STATEFUL_DOMAINS=localhost:3000,app.example.com
```

**Step 2 — Aktifkan stateful API middleware di `bootstrap/app.php`:**

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->statefulApi();
})
```

**Step 3 — Konfigurasi CORS:**

```bash
php artisan config:publish cors
```

Set `config/cors.php`:
```php
'supports_credentials' => true,
```

**Step 4 — Session cookie domain di `config/session.php`:**

```php
'domain' => '.domain.com',  // titik diawal = semua subdomain
```

### Konfigurasi Frontend (Axios)

```js
// resources/js/app.js atau file main frontend
axios.defaults.withCredentials = true;
axios.defaults.withXSRFToken = true;
```

### Alur Autentikasi SPA

**Step 1 — Inisialisasi CSRF:**
```js
await axios.get('/sanctum/csrf-cookie');
```

Ini akan meng-set cookie `XSRF-TOKEN`. Axios akan otomatis mengirimnya sebagai header `X-XSRF-TOKEN` di request berikutnya.

**Step 2 — Login:**
```js
const response = await axios.post('/login', {
    email: 'user@example.com',
    password: 'password',
});
// Session cookie sudah tersimpan otomatis
```

**Step 3 — Akses route terproteksi:**
```js
const user = await axios.get('/api/user');
// Cookie otomatis terkirim, user terautentikasi
```

### Endpoint Login Manual (Backend)

Buat controller untuk login SPA:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Auth;

class AuthController extends Controller
{
    public function login(Request $request): JsonResponse
    {
        $credentials = $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required'],
        ]);

        if (Auth::attempt($credentials)) {
            $request->session()->regenerate();

            return response()->json([
                'user' => $request->user(),
                'message' => 'Login berhasil'
            ]);
        }

        return response()->json([
            'message' => 'Email atau password salah'
        ], 401);
    }

    public function logout(Request $request): JsonResponse
    {
        Auth::logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return response()->json([
            'message' => 'Logout berhasil'
        ]);
    }

    public function user(Request $request): JsonResponse
    {
        return response()->json($request->user());
    }
}
```

### Route SPA

```php
// routes/api.php — untuk SPA
Route::post('/login', [AuthController::class, 'login']);
Route::post('/logout', [AuthController::class, 'logout'])->middleware('auth:sanctum');
Route::get('/user', [AuthController::class, 'user'])->middleware('auth:sanctum');
```

---

## Mobile Application Authentication

Untuk mobile app (iOS/Android), Sanctum menggunakan **API token**.

### Endpoint Login Mobile

```php
<?php

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;
use Illuminate\Support\Facades\Route;

Route::post('/sanctum/token', function (Request $request) {
    $request->validate([
        'email' => 'required|email',
        'password' => 'required',
        'device_name' => 'required', // misal: "Nuno's iPhone 17"
    ]);

    $user = User::where('email', $request->email)->first();

    if (! $user || ! Hash::check($request->password, $user->password)) {
        throw ValidationException::withMessages([
            'email' => ['The provided credentials are incorrect.'],
        ]);
    }

    return response()->json([
        'token' => $user->createToken($request->device_name)->plainTextToken,
        'user' => $user
    ]);
});
```

### Request dari Mobile App

```http
POST /sanctum/token
Content-Type: application/json

{
    "email": "user@example.com",
    "password": "secret123",
    "device_name": "iPhone 17"
}

Response:
{
    "token": "2|a1b2c3d4e5f6...",
    "user": { ... }
}
```

Setiap request berikutnya:

```http
GET /api/user
Authorization: Bearer 2|a1b2c3d4e5f6...
Accept: application/json
```

### Tampilkan & Revoke Token (dari Web)

Di halaman settings web, tampilkan daftar token user:

```php
// Di controller
public function tokens(Request $request)
{
    return response()->json($request->user()->tokens);
}

public function revokeToken(Request $request, $tokenId)
{
    $request->user()->tokens()->where('id', $tokenId)->delete();

    return response()->json(['message' => 'Token revoked']);
}
```

---

## Testing Sanctum

Gunakan `Sanctum::actingAs` untuk mensimulasikan user terautentikasi:

### PHPUnit

```php
<?php

namespace Tests\Feature;

use App\Models\User;
use Laravel\Sanctum\Sanctum;
use Tests\TestCase;

class TaskTest extends TestCase
{
    public function test_task_list_can_be_retrieved(): void
    {
        Sanctum::actingAs(
            User::factory()->create(),
            ['view-tasks'] // abilities
        );

        $response = $this->getJson('/api/task');

        $response->assertOk();
    }

    public function test_admin_can_create_task(): void
    {
        Sanctum::actingAs(
            User::factory()->create(['role' => 'admin']),
            ['*'] // all abilities
        );

        $response = $this->postJson('/api/task', [
            'title' => 'New Task',
        ]);

        $response->assertCreated();
    }

    public function test_unauthenticated_user_cannot_access_tasks(): void
    {
        $response = $this->getJson('/api/task');

        $response->assertUnauthorized();
    }
}
```

### Pest

```php
<?php

use App\Models\User;
use Laravel\Sanctum\Sanctum;

test('task list can be retrieved', function () {
    Sanctum::actingAs(
        User::factory()->create(),
        ['view-tasks']
    );

    $response = $this->getJson('/api/task');

    $response->assertOk();
});

test('unauthenticated user cannot access tasks', function () {
    $response = $this->getJson('/api/task');

    $response->assertUnauthorized();
});

test('admin can create task', function () {
    Sanctum::actingAs(
        User::factory()->create(['role' => 'admin']),
        ['*']
    );

    $response = $this->postJson('/api/task', [
        'title' => 'New Task',
    ]);

    $response->assertCreated();
});
```

---

## Testing via View Sederhana

Untuk testing authentication tanpa perlu tools seperti Postman, kamu bisa bikin view sederhana:

### Route

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\TestAuthController;

Route::get('/test-login', [TestAuthController::class, 'showLogin']);
Route::post('/test-login', [TestAuthController::class, 'login']);
Route::get('/test-user', [TestAuthController::class, 'user'])->middleware('auth:sanctum');
Route::get('/test-tokens', [TestAuthController::class, 'tokens'])->middleware('auth:sanctum');
Route::delete('/test-tokens/{id}', [TestAuthController::class, 'revokeToken'])->middleware('auth:sanctum');
```

### Controller

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Models\User;
use Illuminate\Support\Facades\Hash;

class TestAuthController extends Controller
{
    public function showLogin()
    {
        return view('test-login');
    }

    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            return back()->with('error', 'Email atau password salah');
        }

        $token = $user->createToken('test-token')->plainTextToken;

        return back()->with([
            'success' => 'Login berhasil! Token dibuat.',
            'token' => $token,
            'user' => $user
        ]);
    }

    public function user(Request $request)
    {
        return response()->json($request->user());
    }

    public function tokens(Request $request)
    {
        return response()->json($request->user()->tokens);
    }

    public function revokeToken(Request $request, $tokenId)
    {
        $request->user()->tokens()->where('id', $tokenId)->delete();
        return back()->with('success', 'Token revoked');
    }
}
```

### View `resources/views/test-login.blade.php`

```html
<!DOCTYPE html>
<html>
<head>
    <title>Test Auth Sanctum</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body { font-family: system-ui, sans-serif; max-width: 600px; margin: 40px auto; padding: 0 20px; }
        .card { border: 1px solid #ddd; border-radius: 8px; padding: 20px; margin-bottom: 20px; }
        input, button { width: 100%; padding: 10px; margin-bottom: 10px; border: 1px solid #ddd; border-radius: 4px; box-sizing: border-box; }
        button { background: #6366f1; color: white; border: none; font-weight: bold; cursor: pointer; }
        .success { background: #d1fae5; color: #065f46; padding: 10px; border-radius: 4px; margin-bottom: 10px; }
        .error { background: #fee2e2; color: #991b1b; padding: 10px; border-radius: 4px; margin-bottom: 10px; }
        code { background: #f3f4f6; padding: 2px 6px; border-radius: 3px; font-size: 0.9em; word-break: break-all; }
        h2 { margin-top: 0; }
    </style>
</head>
<body>
    <h1>🔐 Test Auth Sanctum</h1>

    <div class="card">
        <h2>Login & Buat Token</h2>

        @if(session('success'))
            <div class="success">{{ session('success') }}</div>
        @endif

        @if(session('error'))
            <div class="error">{{ session('error') }}</div>
        @endif

        <form method="POST" action="/test-login">
            @csrf
            <input type="email" name="email" placeholder="Email" required>
            <input type="password" name="password" placeholder="Password" required>
            <button type="submit">Login & Generate Token</button>
        </form>

        @if(session('token'))
            <div class="card" style="background: #f0f9ff;">
                <h3>✅ Token Berhasil Dibuat</h3>
                <p><strong>User:</strong> {{ session('user')->email }}</p>
                <p><strong>Token:</strong></p>
                <code>{{ session('token') }}</code>
                <p style="font-size: 0.85em; color: #666;">
                    ⚠️ Token ini hanya muncul sekali. Simpan untuk testing API.
                </p>
            </div>
        @endif
    </div>

    <div class="card">
        <h2>Testing Endpoint</h2>
        <p>Gunakan token di atas untuk test via terminal:</p>
        <code>curl -H "Authorization: Bearer YOUR_TOKEN" http://localhost:8000/api/user</code>
        <br><br>
        <p>Atau akses endpoint ini (perlu token di header):</p>
        <ul>
            <li><code>GET /api/user</code> — Lihat user</li>
            <li><code>GET /api/tokens</code> — Lihat semua token</li>
            <li><code>POST /api/tokens/create</code> — Buat token baru</li>
        </ul>
    </div>

    @if(session('token'))
    <div class="card">
        <h2>🔗 Link Cepat (Via Query Parameter)</h2>
        <p>
            <a href="/api/user?token={{ session('token') }}" target="_blank">
                /api/user?token={{ session('token') }}
            </a>
        </p>
    </div>
    @endif
</body>
</html>
```

Buka `http://localhost:8000/test-login` di browser, login, dan langsung dapat token untuk testing.

---

## Cheatsheet Sanctum

### 📦 Instalasi

```bash
php artisan install:api
```

### 🧩 Model User

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

### 🔐 API Token

| Aksi | Kode |
|------|------|
| **Buat token** | `$user->createToken('name', ['ability'])->plainTextToken` |
| **Cek ability** | `$user->tokenCan('ability')` |
| **Cek tidak punya** | `$user->tokenCant('ability')` |
| **Revoke semua** | `$user->tokens()->delete()` |
| **Revoke current** | `$request->user()->currentAccessToken()->delete()` |
| **Revoke spesifik** | `$user->tokens()->where('id', $id)->delete()` |
| **Ambil semua token** | `$user->tokens` |
| **Token dengan expiry** | `$user->createToken('name', ['*'], now()->addDays(7))` |

### 🛡️ Middleware

```php
// Melindungi route
Route::get('/user', fn(Request $r) => $r->user())->middleware('auth:sanctum');

// Abilities — harus punya SEMUA
Route::get('/orders', fn() => ...)->middleware(['auth:sanctum', 'abilities:read,write']);

// Abilities — cukup punya SALAH SATU
Route::get('/orders', fn() => ...)->middleware(['auth:sanctum', 'ability:read,write']);
```

### 🌐 SPA Authentication

```js
// Frontend (Axios)
axios.defaults.withCredentials = true;
axios.defaults.withXSRFToken = true;

// Step 1: CSRF
await axios.get('/sanctum/csrf-cookie');

// Step 2: Login
await axios.post('/login', { email, password });

// Step 3: Akses API
const user = await axios.get('/api/user');
```

### 📱 Mobile Authentication

```http
POST /sanctum/token
Body: { email, password, device_name }
Response: { token: "1|abc123...", user: {...} }

Request API:
GET /api/user
Authorization: Bearer 1|abc123...
```

### 🧪 Testing

```php
// Auth user with abilities
Sanctum::actingAs(User::factory()->create(), ['view-tasks']);

// Auth user with ALL abilities
Sanctum::actingAs(User::factory()->create(), ['*']);
```

### ⚙️ Konfigurasi Penting

| File | Konfigurasi |
|------|------------|
| `config/sanctum.php` | `stateful` (domain SPA), `expiration` (menit) |
| `config/cors.php` | `supports_credentials => true` |
| `config/session.php` | `domain => '.domain.com'` (untuk subdomain) |
| `.env` | `SANCTUM_STATEFUL_DOMAINS=localhost:3000` |
| `bootstrap/app.php` | `$middleware->statefulApi()` |

---

> **Panduan Konseptual**: Untuk penjelasan tentang apa itu Sanctum, bagaimana cara kerjanya, dan kapan menggunakannya, lihat file [`auth-dan-laravel.md`](./auth-dan-laravel.md).

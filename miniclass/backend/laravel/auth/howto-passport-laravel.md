# Cara Implementasi Laravel Passport

> **Panduan langkah demi langkah** implementasi Passport — OAuth2 server untuk Laravel.

---

## Instalasi Passport

### Cara 1: Via Artisan (Laravel 13+) — Termudah

```bash
php artisan install:api --passport
```

Perintah ini akan:
- Menginstal package Passport via Composer ✅
- Membuat migration tabel OAuth2 (clients, tokens, auth codes, refresh tokens) ✅
- Membuat encryption keys ✅
- Menerbitkan konfigurasi `config/passport.php` ✅

### Cara 2: Manual via Composer

```bash
composer require laravel/passport
php artisan migrate
php artisan passport:install
```

`passport:install` akan membuat:
- **Encryption keys** — untuk sign JWT
- **Personal Access Client** — untuk personal access tokens
- **Password Grant Client** — opsional, jika pakai password grant

### Deploy Keys ke Production

Saat deploy pertama kali, generate keys:

```bash
php artisan passport:keys
```

Jika perlu, tentukan path kustom untuk keys:

```php
// App\Providers\AppServiceProvider.php
public function boot(): void
{
    Passport::loadKeysFrom(__DIR__.'/../secrets/oauth');
}
```

Atau simpan keys di environment variables (lebih aman):

```bash
php artisan vendor:publish --tag=passport-config
```

Lalu set di `.env`:

```ini
PASSPORT_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----
<private key here>
-----END RSA PRIVATE KEY-----"

PASSPORT_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----
<public key here>
-----END PUBLIC KEY-----"
```

---

## Persiapan Model User & Config

### Update Model User

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Passport\Contracts\OAuthenticatable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable implements OAuthenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

> **Penting**: Trait `HasApiTokens` dari Passport berbeda dengan Sanctum — namespace `Laravel\Passport` vs `Laravel\Sanctum`.

### Konfigurasi Auth Guard

Di `config/auth.php`, atur guard `api` ke driver `passport`:

```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
],
```

### Atur Token Lifetime (Opsional)

Di `App\Providers\AppServiceProvider.php`:

```php
<?php

namespace App\Providers;

use Carbon\CarbonInterval;
use Laravel\Passport\Passport;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Passport::tokensExpireIn(CarbonInterval::days(15));
        Passport::refreshTokensExpireIn(CarbonInterval::days(30));
        Passport::personalAccessTokensExpireIn(CarbonInterval::months(6));
    }
}
```

---

## Authorization Code Grant

Flow OAuth2 klasik — user redirect ke halaman login, setuju, dapat token.

### Membuat Client

**First-party client (via Artisan):**

```bash
php artisan passport:client
```

Ikuti prompt untuk memasukkan nama client dan redirect URI.

**Third-party client (via code — misal dari halaman settings user):**

```php
<?php

use App\Models\User;
use Laravel\Passport\ClientRepository;

$user = User::find($userId);

$client = app(ClientRepository::class)->createAuthorizationCodeGrantClient(
    user: $user,
    name: 'Example App',
    redirectUris: ['https://third-party-app.com/callback'],
    confidential: false,
);

// Tampilkan ke user
echo "Client ID: " . $client->id;
echo "Client Secret: " . $client->plainSecret;
```

### Custom Halaman Authorization (Opsional)

Di `App\Providers\AppServiceProvider.php`:

```php
<?php

use Laravel\Passport\Passport;

public function boot(): void
{
    Passport::authorizationView('auth.oauth.authorize');
}
```

Buat view `resources/views/auth/oauth/authorize.blade.php` dengan form approve/deny.

Atau pakai closure (untuk Inertia/React):

```php
Passport::authorizationView(
    fn ($parameters) => Inertia::render('Auth/OAuth/Authorize', [
        'request' => $parameters['request'],
        'authToken' => $parameters['authToken'],
        'client' => $parameters['client'],
        'user' => $parameters['user'],
        'scopes' => $parameters['scopes'],
    ])
);
```

### Redirect User ke Halaman Otorisasi (Dari Third-Party App)

```php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Str;

Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));

    $query = http_build_query([
        'client_id' => 'your-client-id',
        'redirect_uri' => 'https://third-party-app.com/callback',
        'response_type' => 'code',
        'scope' => 'user:read orders:create',
        'state' => $state,
    ]);

    return redirect('https://passport-app.test/oauth/authorize?'.$query);
});
```

Parameter `prompt` (opsional):

| Value | Behavior |
|-------|----------|
| `none` | Gagal jika user belum login |
| `consent` | Selalu tampilkan halaman persetujuan |
| `login` | Paksa user login ulang |

### Tukar Authorization Code dengan Access Token

```php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;

Route::get('/callback', function (Request $request) {
    $state = $request->session()->pull('state');

    throw_unless(
        strlen($state) > 0 && $state === $request->state,
        InvalidArgumentException::class,
        'Invalid state value.'
    );

    $response = Http::asForm()->post('https://passport-app.test/oauth/token', [
        'grant_type' => 'authorization_code',
        'client_id' => 'your-client-id',
        'client_secret' => 'your-client-secret',
        'redirect_uri' => 'https://third-party-app.com/callback',
        'code' => $request->code,
    ]);

    return $response->json();
});
```

**Response sukses:**
```json
{
    "token_type": "Bearer",
    "expires_in": 3600,
    "access_token": "eyJ0eXAiOiJKV1Qi...",
    "refresh_token": "def50200..."
}
```

### Refresh Token

```php
<?php

use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/token', [
    'grant_type' => 'refresh_token',
    'refresh_token' => 'the-refresh-token',
    'client_id' => 'your-client-id',
    'client_secret' => 'your-client-secret', // hanya untuk confidential clients
    'scope' => 'user:read orders:create',
]);

return $response->json();
```

### Revoke Token

```php
<?php

use Laravel\Passport\Passport;
use Laravel\Passport\Token;

$token = Passport::token()->find($tokenId);

// Revoke access token
$token->revoke();

// Revoke refresh token juga
$token->refreshToken?->revoke();

// Revoke semua token user
User::find($userId)->tokens()->each(function (Token $token) {
    $token->revoke();
    $token->refreshToken?->revoke();
});
```

### Purge Token Expired

```bash
# Purge semua token revoked & expired
php artisan passport:purge

# Purge token expired lebih dari 6 jam
php artisan passport:purge --hours=6

# Purge hanya token revoked
php artisan passport:purge --revoked

# Purge hanya token expired
php artisan passport:purge --expired
```

Atur jadwal otomatis di `routes/console.php`:

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('passport:purge')->hourly();
```

### Managing Tokens (Dashboard User)

Tampilkan daftar koneksi third-party ke user:

```php
<?php

use App\Models\User;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Support\Facades\Date;
use Laravel\Passport\Token;

$user = User::find($userId);

// Semua token valid milik user
$tokens = $user->tokens()
    ->where('revoked', false)
    ->where('expires_at', '>', Date::now())
    ->get();

// Koneksi ke third-party OAuth apps (bukan first-party)
$connections = $tokens->load('client')
    ->reject(fn (Token $token) => $token->client->firstParty())
    ->groupBy('client_id')
    ->map(fn (Collection $tokens) => [
        'client' => $tokens->first()->client,
        'scopes' => $tokens->pluck('scopes')->flatten()->unique()->values()->all(),
        'tokens_count' => $tokens->count(),
    ])
    ->values();
```

### Skip Authorization untuk First-Party Client

Agar first-party client tidak perlu menampilkan halaman persetujuan:

```php
<?php

namespace App\Models\Passport;

use Illuminate\Contracts\Auth\Authenticatable;
use Laravel\Passport\Client as BaseClient;

class Client extends BaseClient
{
    public function skipsAuthorization(Authenticatable $user, array $scopes): bool
    {
        return $this->firstParty();
    }
}
```

### Overriding Default Models

Kustomisasi model Passport (misal tambah relasi ke model lain):

```php
<?php

use App\Models\Passport\AuthCode;
use App\Models\Passport\Client;
use App\Models\Passport\DeviceCode;
use App\Models\Passport\RefreshToken;
use App\Models\Passport\Token;
use Laravel\Passport\Passport;

// Di AppServiceProvider::boot()
Passport::useTokenModel(Token::class);
Passport::useRefreshTokenModel(RefreshToken::class);
Passport::useAuthCodeModel(AuthCode::class);
Passport::useClientModel(Client::class);
Passport::useDeviceCodeModel(DeviceCode::class);
```

### Overriding Routes

Jika perlu kostumasi route Passport:

```php
<?php

use Laravel\Passport\Passport;

// Di AppServiceProvider::register()
Passport::ignoreRoutes();
```

Lalu salin route dari [routes file Passport](https://github.com/laravel/passport/blob/master/routes/web.php) ke `routes/web.php`.

---

## Authorization Code Grant dengan PKCE

Untuk SPA & mobile apps yang **tidak bisa menyimpan client secret**.

### Buat Public Client

```bash
php artisan passport:client --public
```

### Generate Code Verifier & Code Challenge

```php
<?php

$codeVerifier = Str::random(128);

$codeChallenge = strtr(rtrim(
    base64_encode(hash('sha256', $codeVerifier, true))
, '='), '+/', '-_');
```

### Redirect User

```php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Str;

Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));
    $request->session()->put('code_verifier', $codeVerifier = Str::random(128));

    $codeChallenge = strtr(rtrim(
        base64_encode(hash('sha256', $codeVerifier, true))
    , '='), '+/', '-_');

    $query = http_build_query([
        'client_id' => 'your-client-id',
        'redirect_uri' => 'https://third-party-app.com/callback',
        'response_type' => 'code',
        'scope' => 'user:read orders:create',
        'state' => $state,
        'code_challenge' => $codeChallenge,
        'code_challenge_method' => 'S256',
    ]);

    return redirect('https://passport-app.test/oauth/authorize?'.$query);
});
```

### Tukar Code dengan Token (Tanpa Client Secret!)

```php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;

Route::get('/callback', function (Request $request) {
    $state = $request->session()->pull('state');
    $codeVerifier = $request->session()->pull('code_verifier');

    throw_unless(
        strlen($state) > 0 && $state === $request->state,
        InvalidArgumentException::class
    );

    $response = Http::asForm()->post('https://passport-app.test/oauth/token', [
        'grant_type' => 'authorization_code',
        'client_id' => 'your-client-id',
        'redirect_uri' => 'https://third-party-app.com/callback',
        'code_verifier' => $codeVerifier,
        'code' => $request->code,
    ]);

    return $response->json();
});
```

---

## Client Credentials Grant (Machine-to-Machine)

Untuk komunikasi server ke server (cron job, internal API) — tanpa melibatkan user.

### Buat Client

```bash
php artisan passport:client --client
```

### Minta Token

```php
<?php

use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/token', [
    'grant_type' => 'client_credentials',
    'client_id' => 'your-client-id',
    'client_secret' => 'your-client-secret',
    'scope' => 'servers:read servers:create',
]);

$accessToken = $response->json()['access_token'];
```

### Melindungi Route untuk Client Credentials

Gunakan middleware khusus `EnsureClientIsResourceOwner`:

```php
<?php

use Laravel\Passport\Http\Middleware\EnsureClientIsResourceOwner;

Route::get('/orders', function (Request $request) {
    // Client terautentikasi sebagai resource owner
})->middleware(EnsureClientIsResourceOwner::class);

// Dengan scope tertentu
Route::get('/orders', function (Request $request) {
    // Client punya scope "servers:read" DAN "servers:create"
})->middleware(EnsureClientIsResourceOwner::using('servers:read', 'servers:create'));
```

> **⚠️ UUID Warning**: OAuth2 server set `sub` claim ke client ID untuk client credentials token. Passport default pakai UUID, jadi tidak bentrok dengan user ID integer. Jika kamu set `Passport::$clientUuids = false`, client credentials token bisa不经意 resolve user yang ID-nya sama dengan client ID.

---

## Device Authorization Grant

Untuk browserless / limited input devices (TV, game console).

### Buat Device Client

```bash
php artisan passport:client --device
```

### Minta Device Code

```php
<?php

use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/device/code', [
    'client_id' => 'your-client-id',
    'scope' => 'user:read orders:create',
]);

return $response->json();
// Response: { device_code, user_code, verification_uri, interval, expires_in }
```

### Polling Token

Device menampilkan `verification_uri` + `user_code` ke user, lalu polling `/oauth/token`:

```php
<?php

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Sleep;

$interval = 5;

do {
    Sleep::for($interval)->seconds();

    $response = Http::asForm()->post('https://passport-app.test/oauth/token', [
        'grant_type' => 'urn:ietf:params:oauth:grant-type:device_code',
        'client_id' => 'your-client-id',
        'client_secret' => 'your-client-secret',
        'device_code' => 'the-device-code',
    ]);

    if ($response->json('error') === 'slow_down') {
        $interval += 5;
    }
} while (in_array($response->json('error'), ['authorization_pending', 'slow_down']));

return $response->json();
```

---

## Password Grant

> **⚠️ Sudah tidak direkomendasikan.** Pilih grant type lain seperti [Authorization Code + PKCE](#authorization-code-grant-dengan-pkce) atau [Client Credentials](#client-credentials-grant-machine-to-machine).

Aktifkan password grant di `AppServiceProvider`:

```php
public function boot(): void
{
    Passport::enablePasswordGrant();
}
```

### Buat Password Grant Client

```bash
php artisan passport:client --password
```

### Request Token

```php
<?php

use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://passport-app.test/oauth/token', [
    'grant_type' => 'password',
    'client_id' => 'your-client-id',
    'client_secret' => 'your-client-secret',
    'username' => 'user@example.com',
    'password' => 'my-password',
    'scope' => 'user:read orders:create',
]);

return $response->json();
```

### Custom Username Field

Ganti field email dengan field lain (misal `username`):

```php
<?php

namespace App\Models;

use Laravel\Passport\Bridge\Client;
use Laravel\Passport\Contracts\OAuthenticatable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable implements OAuthenticatable
{
    use HasApiTokens;

    public function findForPassport(string $username, Client $client): User
    {
        return $this->where('username', $username)->first();
    }
}
```

### Custom Password Validation

```php
<?php

namespace App\Models;

use Illuminate\Support\Facades\Hash;
use Laravel\Passport\Contracts\OAuthenticatable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable implements OAuthenticatable
{
    use HasApiTokens;

    public function validateForPassportPasswordGrant(string $password): bool
    {
        return Hash::check($password, $this->password);
    }
}
```

---

## Implicit Grant

> **⚠️ Sudah tidak direkomendasikan.** Gunakan [Authorization Code + PKCE](#authorization-code-grant-dengan-pkce) sebagai gantinya.

Aktifkan:

```php
public function boot(): void
{
    Passport::enableImplicitGrant();
}
```

Buat client:

```bash
php artisan passport:client --implicit
```

Redirect user (token langsung didapat tanpa exchange code):

```php
Route::get('/redirect', function (Request $request) {
    $request->session()->put('state', $state = Str::random(40));

    $query = http_build_query([
        'client_id' => 'your-client-id',
        'redirect_uri' => 'https://third-party-app.com/callback',
        'response_type' => 'token',
        'scope' => 'user:read orders:create',
        'state' => $state,
    ]);

    return redirect('https://passport-app.test/oauth/authorize?'.$query);
});
```

---

## Personal Access Tokens

Mirip Sanctum — user membuat token dari UI untuk eksperimen API.

### Buat Personal Access Client

```bash
php artisan passport:client --personal
```

(Lewati jika sudah `passport:install`)

### Membuat Token

```php
<?php

use App\Models\User;

$user = User::find($userId);

// Token tanpa scope
$token = $user->createToken('My Token')->accessToken;

// Token dengan scope
$token = $user->createToken('My Token', ['user:read', 'orders:create'])->accessToken;

// Token dengan semua scope
$token = $user->createToken('My Token', ['*'])->accessToken;
```

### Melihat Semua Personal Access Token

```php
<?php

use Illuminate\Support\Facades\Date;
use Laravel\Passport\Token;

$tokens = $user->tokens()
    ->with('client')
    ->where('revoked', false)
    ->where('expires_at', '>', Date::now())
    ->get()
    ->filter(fn (Token $token) => $token->client->hasGrantType('personal_access'));
```

---

## Token Scopes

### Mendefinisikan Scopes

Di `App\Providers\AppServiceProvider.php`:

```php
<?php

use Laravel\Passport\Passport;

public function boot(): void
{
    Passport::tokensCan([
        'user:read' => 'Melihat data user',
        'orders:create' => 'Membuat pesanan',
        'orders:read:status' => 'Cek status pesanan',
    ]);

    // Default scope jika client tidak request scope spesifik
    Passport::defaultScopes([
        'user:read',
    ]);
}
```

### Cek Scope di Route

Via middleware:

```php
<?php

use Laravel\Passport\Http\Middleware\CheckToken;
use Laravel\Passport\Http\Middleware\CheckTokenForAnyScope;

// HARUS punya kedua scope
Route::get('/orders', function () {
    // ...
})->middleware(['auth:api', CheckToken::using('orders:read', 'orders:create')]);

// CUKUP punya salah satu
Route::get('/orders', function () {
    // ...
})->middleware(['auth:api', CheckTokenForAnyScope::using('orders:read', 'orders:create')]);
```

Via PHP Attribute (Laravel 13+):

```php
<?php

namespace App\Http\Controllers;

use Laravel\Passport\Attributes\AuthorizeToken;

#[AuthorizeToken('orders:read')]
class OrderController
{
    // Semua method harus punya scope 'orders:read'

    #[AuthorizeToken('orders:create')]
    public function store()
    {
        // Method ini harus punya 'orders:read' DAN 'orders:create'
    }

    #[AuthorizeToken(['orders:read', 'orders:create'], anyScope: true)]
    public function index()
    {
        // Method ini cukup punya salah satu scope
    }
}
```

### Cek Scope di Controller

```php
<?php

use Illuminate\Http\Request;

Route::get('/orders', function (Request $request) {
    if ($request->user()->tokenCan('orders:create')) {
        // Boleh membuat pesanan
    }
});
```

### Scope Utility Methods

```php
<?php

use Laravel\Passport\Passport;

// Semua scope IDs
Passport::scopeIds(); // ['user:read', 'orders:create', ...]

// Semua scope objects
Passport::scopes();

// Scope objects untuk IDs tertentu
Passport::scopesFor(['user:read', 'orders:create']);

// Cek apakah scope sudah didefinisikan
Passport::hasScope('orders:create');
```

---

## Melindungi Route

### Sanctum

```php
Route::get('/user', fn(Request $r) => $r->user())->middleware('auth:sanctum');
```

### Passport

```php
Route::get('/user', fn(Request $r) => $r->user())->middleware('auth:api');
```

### Multiple Guards (Multi User Type)

Jika punya multiple user types (users + customers), definisikan di `config/auth.php`:

```php
'guards' => [
    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
    'api-customers' => [
        'driver' => 'passport',
        'provider' => 'customers',
    ],
],
```

Lalu:

```php
Route::get('/customer', function () {
    // Dilindungi guard api-customers
})->middleware('auth:api-customers');
```

### Passing Access Token dari Client

```php
<?php

use Illuminate\Support\Facades\Http;

$response = Http::withHeaders([
    'Accept' => 'application/json',
    'Authorization' => "Bearer $accessToken",
])->get('https://passport-app.test/api/user');

return $response->json();
```

---

## SPA Authentication dengan Passport

Passport juga bisa melayani SPA first-party via session cookie — menggunakan middleware `CreateFreshApiToken`.

### Tambahkan Middleware

Di `bootstrap/app.php`:

```php
<?php

use Laravel\Passport\Http\Middleware\CreateFreshApiToken;

->withMiddleware(function (Middleware $middleware): void {
    $middleware->web(append: [
        CreateFreshApiToken::class, // HARUS paling akhir
    ]);
})
```

Middleware ini akan menambahkan cookie `laravel_token` (JWT terenkripsi) ke setiap response.

### Custom Nama Cookie (Opsional)

```php
Passport::cookie('custom_name');
```

### Akses API dari SPA

```js
// Axios — cookie otomatis terkirim
axios.get('/api/user')
    .then(response => {
        console.log(response.data);
    });
```

Tanpa perlu kirim token manual!

### CSRF Protection

Pastikan setiap request menyertakan CSRF token. Axios dari starter kit Laravel sudah otomatis mengirim `X-XSRF-TOKEN` dari cookie `XSRF-TOKEN`.

---

## Events

Passport memunculkan events yang bisa kamu listen untuk pruning atau revoke token:

| Event |
|-------|
| `Laravel\Passport\Events\AccessTokenCreated` |
| `Laravel\Passport\Events\AccessTokenRevoked` |
| `Laravel\Passport\Events\RefreshTokenCreated` |

Contoh listener:

```php
<?php

namespace App\Listeners;

use Laravel\Passport\Events\AccessTokenCreated;

class RevokeOldTokens
{
    public function handle(AccessTokenCreated $event): void
    {
        // Revoke token lama saat token baru dibuat
    }
}
```

---

## Testing Passport

### PHPUnit

```php
<?php

namespace Tests\Feature;

use App\Models\User;
use Laravel\Passport\Passport;
use Tests\TestCase;

class OrderTest extends TestCase
{
    public function test_orders_can_be_created(): void
    {
        Passport::actingAs(
            User::factory()->create(),
            ['orders:create']
        );

        $response = $this->postJson('/api/orders');

        $response->assertStatus(201);
    }
}
```

### Pest

```php
<?php

use App\Models\User;
use Laravel\Passport\Passport;

test('orders can be created', function () {
    Passport::actingAs(
        User::factory()->create(),
        ['orders:create']
    );

    $response = $this->post('/api/orders');

    $response->assertStatus(201);
});
```

### Testing Client Credentials

```php
<?php

use Laravel\Passport\Client;
use Laravel\Passport\Passport;

test('servers can be retrieved', function () {
    Passport::actingAsClient(
        Client::factory()->create(),
        ['servers:read']
    );

    $response = $this->get('/api/servers');

    $response->assertStatus(200);
});
```

---

## Testing via View Sederhana

Untuk testing Authorization Code flow tanpa setup third-party app, kamu bisa buat view simulasi:

### Route

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\TestOAuthController;

Route::get('/test-oauth/login', [TestOAuthController::class, 'showLogin']);
Route::post('/test-oauth/login', [TestOAuthController::class, 'login']);
Route::get('/test-oauth/authorize', [TestOAuthController::class, 'showAuthorize']);
Route::post('/test-oauth/approve', [TestOAuthController::class, 'approve']);
Route::get('/test-oauth/callback', [TestOAuthController::class, 'callback']);
```

### Controller

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Models\User;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;
use Illuminate\Support\Facades\Http;

class TestOAuthController extends Controller
{
    public function showLogin()
    {
        return view('test-oauth-login');
    }

    public function login(Request $request)
    {
        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            return back()->with('error', 'Email atau password salah');
        }

        // Simulasi login — simpan user di session
        session(['test_oauth_user' => $user->id]);

        return redirect('/test-oauth/authorize');
    }

    public function showAuthorize()
    {
        if (!session('test_oauth_user')) {
            return redirect('/test-oauth/login');
        }

        return view('test-oauth-authorize', [
            'client_name' => 'Test App',
            'scopes' => ['user:read', 'orders:create']
        ]);
    }

    public function approve(Request $request)
    {
        $user = User::find(session('test_oauth_user'));

        // Buat personal access token sebagai simulasi
        $token = $user->createToken('oauth-test', ['user:read', 'orders:create'])->accessToken;

        session()->forget('test_oauth_user');

        return view('test-oauth-callback', [
            'token' => $token,
            'user' => $user
        ]);
    }

    public function callback()
    {
        return view('test-oauth-callback');
    }
}
```

### View Login `resources/views/test-oauth-login.blade.php`

```html
<!DOCTYPE html>
<html>
<head>
    <title>Test OAuth — Login</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body { font-family: system-ui, sans-serif; max-width: 500px; margin: 40px auto; padding: 0 20px; }
        .card { border: 1px solid #ddd; border-radius: 8px; padding: 24px; }
        input, button { width: 100%; padding: 10px; margin-bottom: 12px; border: 1px solid #ddd; border-radius: 4px; box-sizing: border-box; }
        button { background: #6366f1; color: white; border: none; font-weight: bold; cursor: pointer; }
        .error { background: #fee2e2; color: #991b1b; padding: 10px; border-radius: 4px; }
    </style>
</head>
<body>
    <div class="card">
        <h1>🔑 Test OAuth — Login</h1>
        <p><strong>Test App</strong> ingin mengakses akun Anda.</p>

        @if(session('error'))
            <div class="error">{{ session('error') }}</div>
        @endif

        <form method="POST">
            @csrf
            <input type="email" name="email" placeholder="Email" required>
            <input type="password" name="password" placeholder="Password" required>
            <button type="submit">Login</button>
        </form>
    </div>
</body>
</html>
```

### View Authorize `resources/views/test-oauth-authorize.blade.php`

```html
<!DOCTYPE html>
<html>
<head>
    <title>Test OAuth — Authorize</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body { font-family: system-ui, sans-serif; max-width: 500px; margin: 40px auto; padding: 0 20px; }
        .card { border: 1px solid #ddd; border-radius: 8px; padding: 24px; }
        button { width: 100%; padding: 10px; margin-bottom: 8px; border: none; border-radius: 4px; font-weight: bold; cursor: pointer; }
        .approve { background: #059669; color: white; }
        .deny { background: #dc2626; color: white; }
        ul { background: #f3f4f6; padding: 12px 24px; border-radius: 4px; }
    </style>
</head>
<body>
    <div class="card">
        <h1>🔐 Otorisasi</h1>
        <p><strong>{{ $client_name }}</strong> meminta akses ke:</p>
        <ul>
            @foreach($scopes as $scope)
                <li>{{ $scope }}</li>
            @endforeach
        </ul>
        <form method="POST" action="/test-oauth/approve">
            @csrf
            <button type="submit" class="approve">Setujui</button>
        </form>
    </div>
</body>
</html>
```

### View Callback `resources/views/test-oauth-callback.blade.php`

```html
<!DOCTYPE html>
<html>
<head>
    <title>Test OAuth — Success</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body { font-family: system-ui, sans-serif; max-width: 600px; margin: 40px auto; padding: 0 20px; }
        .card { border: 1px solid #ddd; border-radius: 8px; padding: 24px; }
        code { background: #f3f4f6; padding: 2px 6px; border-radius: 3px; font-size: 0.9em; word-break: break-all; }
        .success { background: #d1fae5; color: #065f46; padding: 10px; border-radius: 4px; }
        .box { background: #f0f9ff; border: 1px solid #bae6fd; padding: 16px; border-radius: 4px; margin-top: 16px; }
    </style>
</head>
<body>
    <div class="card">
        <h1>✅ Otorisasi Berhasil</h1>

        @if(isset($token))
            <div class="success">Access token berhasil didapatkan!</div>

            <div class="box">
                <h3>Access Token (JWT):</h3>
                <code>{{ $token }}</code>
            </div>

            <div class="box">
                <h3>Testing:</h3>
                <p>Gunakan token untuk akses API:</p>
                <code>curl -H "Authorization: Bearer {{ $token }}" http://localhost:8000/api/user</code>
            </div>

            <div class="box">
                <h3>🔗 Link Cepat:</h3>
                <p><a href="/api/user" target="_blank">/api/user</a> (via session)</p>
            </div>
        @else
            <p>Otorisasi ditolak.</p>
        @endif
    </div>
</body>
</html>
```

Alur: Buka `/test-oauth/login` → login → approve → dapat token JWT.

---

## Cheatsheet Passport

### 📦 Instalasi

```bash
php artisan install:api --passport
php artisan passport:keys    # untuk deploy
```

### 🔑 Manajemen Client

| Aksi | Perintah / Kode |
|------|----------------|
| **Buat client** | `php artisan passport:client` |
| **Buat public client (PKCE)** | `php artisan passport:client --public` |
| **Buat client credentials** | `php artisan passport:client --client` |
| **Buat personal access** | `php artisan passport:client --personal` |
| **Buat device grant** | `php artisan passport:client --device` |
| **Custom third-party client** | `ClientRepository::createAuthorizationCodeGrantClient(...)` |

### 🔐 Token & Endpoint

| Aksi | Endpoint / Kode |
|------|----------------|
| **Minta token** | `POST /oauth/token` dengan grant_type sesuai |
| **Refresh token** | `POST /oauth/token` dengan `grant_type=refresh_token` |
| **Revoke access token** | `$token->revoke()` |
| **Revoke refresh token** | `$token->refreshToken?->revoke()` |
| **Purge expired** | `php artisan passport:purge` |
| **Generate keys** | `php artisan passport:keys` |

### 🛡️ Melindungi Route

```php
// Passport
Route::get('/user', fn() => ...)->middleware('auth:api');

// Client credentials — pastikan client adalah resource owner
Route::get('/orders', fn() => ...)->middleware(EnsureClientIsResourceOwner::class);

// Dengan scope — HARUS SEMUA
Route::get('/orders', fn() => ...)->middleware(['auth:api', CheckToken::using('read', 'write')]);

// Dengan scope — CUKUP SATU
Route::get('/orders', fn() => ...)->middleware(['auth:api', CheckTokenForAnyScope::using('read', 'write')]);
```

### 🧪 Testing

```php
// Auth user dengan scope
Passport::actingAs(User::factory()->create(), ['orders:create']);

// Auth client dengan scope (client credentials)
Passport::actingAsClient(Client::factory()->create(), ['servers:read']);
```

### ⚙️ Konfigurasi Penting

| File | Konfigurasi |
|------|------------|
| `config/auth.php` | Guard `api` → `driver => 'passport'` |
| `config/passport.php` | Token lifetime, client UUID |
| `AppServiceProvider` | `Passport::tokensCan(...)`, `Passport::tokensExpireIn(...)` |
| `.env` | `PASSPORT_PRIVATE_KEY`, `PASSPORT_PUBLIC_KEY` |
| `bootstrap/app.php` | `CreateFreshApiToken` middleware (untuk SPA) |

---

> **Panduan Konseptual**: Untuk penjelasan tentang apa itu Passport, bagaimana cara kerjanya, dan kapan menggunakannya, lihat file [`auth-dan-laravel.md`](./auth-dan-laravel.md).

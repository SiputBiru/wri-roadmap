# Authentication di Laravel

Laravel menyediakan sistem authentication yang **lengkap, aman, dan mudah digunakan**. Alih-alih menulis kode authentication dari nol, Laravel sudah menyiapkan berbagai komponen siap pakai — dari session-based auth untuk web monolith hingga token-based auth untuk API dan mobile apps.

---

## Arsitektur Authentication Laravel

Authentication Laravel dibangun di atas dua konsep utama:

| Konsep | Penjelasan |
|---|---|
| **Guards** | Mendefinisikan **bagaimana** user diautentikasi di setiap request. Contoh: `session` guard (menyimpan state via session & cookie), `sanctum` guard (via token). |
| **Providers** | Mendefinisikan **dari mana** data user diambil (database, Eloquent, API eksternal, dll). |

Konfigurasi authentication Laravel ada di file `config/auth.php`.

---

## Authentication Bawaan (Browser)

Untuk aplikasi web monolith (Laravel tradisional), Laravel menggunakan **session-based authentication**. Cara kerjanya:

```
Browser                    Server
   │                         │
   │── POST /login ─────────→│ Verifikasi kredensial
   │   (email + password)    │
   │                         │── Cocok? Simpan user_id di session
   │←─ Set cookie (session) ─│
   │                         │
   │── GET /dashboard ──────→│ Cocokkan session ID
   │   (cookie otomatis)     │── User terautentikasi?
   │←─ 200 OK ───────────────│
```

**Ciri utama session-based auth:**
- **Stateful** — data login disimpan di **server** (session database atau Redis)
- **Cookie** — browser hanya menyimpan session ID, bukan data user
- **CSRF rawan** — karena cookie otomatis terkirim, perlu CSRF protection
- **Mudah di-revoke** — tinggal hapus session dari database, user langsung logout

### Starter Kits

Untuk mempercepat development, Laravel menyediakan **starter kits** — aplikasi siap pakai yang langsung membangun seluruh sistem authentication (registrasi, login, lupa password, verifikasi email) dalam hitungan menit.

> **Perubahan mulai  di Laravel 12**: Sebelumnya Laravel menggunakan paket terpisah bernama **Laravel Breeze** yang diinstall via `composer require laravel/breeze`. Mulai Laravel 11, Breeze/starter-kits terintegrasi langsung ke dalam perintah **`laravel new`** — ketika membuat project baru, kamu akan diminta memilih stack frontend (Blade, React, Vue, Svelte, Livewire) beserta opsi WorkOS AuthKit secara interaktif. Konsep dan kustomisasinya tetap sama, hanya cara instalasinya yang berubah.

| Starter Kit | Frontend | Cocok Untuk |
|---|---|---|
| **Breeze/starter-kits (built-in)** | Blade / React / Vue / Svelte / Livewire | Aplikasi sederhana & ringan |
| **Breeze/starter-kits + WorkOS** | React / Vue / Svelte / Livewire | Butuh social login, SSO, magic auth |
| **Jetstream** (Laravel 11-) | Livewire / Inertia | Aplikasi kompleks (tim, 2FA, API tokens) |

---

## Sanctum — Hybrid Authentication

### Apa itu Sanctum?

**Laravel Sanctum** adalah paket authentication hybrid ringan yang **direkomendasikan oleh Laravel** untuk sebagian besar aplikasi. Sanctum dirancang agar bisa menangani **dua skenario** authentication dalam satu paket:

1. **API Token Authentication** — untuk mobile apps dan third-party API
2. **SPA Authentication** — untuk first-party SPA (Next.js, Vue, React)

### Bagaimana Sanctum Bekerja?

Sanctum secara otomatis memilih metode autentikasi berdasarkan request yang masuk:

```
Request Masuk
    │
    ├── Ada session cookie? → Ya → Autentikasi via SESSION (cookie)
    │                              (untuk SPA di domain yang sama)
    │
    └── Tidak ada cookie → Cek header Authorization: Bearer
                           │
                           ├── Token valid → Autentikasi via TOKEN
                           │                (untuk mobile / third-party)
                           │
                           └── Token tidak valid → 401 Unauthorized
```

**Dua mode authentication Sanctum:**

| Mode | Media | State | Cocok Untuk |
|------|-------|-------|-------------|
| **Session (cookie)** | Cookie `XSRF-TOKEN` + session | Stateful | SPA first-party (domain sama) |
| **API Token** | Header `Authorization: Bearer` | Stateless | Mobile apps, third-party API |

### Karakteristik Token Sanctum

- **Opaque token** — bukan JWT. Token berupa string acak seperti `1|abc123def456...`
- **Di-hash SHA-256** sebelum disimpan di database
- **Bisa di-revoke kapan saja** — cukup hapus dari database, token langsung tidak valid
- **Bisa diberi abilities (scopes)** — untuk mengontrol akses token
- **Tidak ada refresh token** — token berlaku panjang, revoke manual jika perlu

### Konsep Penting Sanctum

| Konsep | Penjelasan |
|--------|-----------|
| **Stateful domains** | Daftar domain SPA yang diizinkan menggunakan session auth. Diatur di `config/sanctum.php` |
| **Abilities** | Izin yang melekat pada token (contoh: `server:update`, `orders:create`) |
| **CSRF Protection** | Untuk SPA, harus ambil `/sanctum/csrf-cookie` dulu sebelum login |
| **Token expiry** | Opsional. Default: tidak pernah expired (kecuali diatur) |

---

## Fortify — Headless Auth Backend

### Apa itu Fortify?

**Laravel Fortify** adalah package authentication **frontend-agnostic** untuk Laravel. Fortify menyediakan **semua route dan controller backend** untuk login, registrasi, reset password, verifikasi email, konfirmasi password, two-factor authentication (2FA), hingga passkeys (WebAuthn) — **tanpa UI**.

Fortify adalah **package** yang bisa diinstall via Composer (`laravel/fortify`). Di belakang layar, **Starter Kits** (Breeze, Jetstream) menggunakan Fortify untuk backend logic auth-nya.

```
┌────────────────────────────────────────────────────┐
│                   STARTER KITS                     │
│  (Breeze / Jetstream)                              │
│  ┌──────────────────┐  ┌─────────────────────────┐ │
│  │  UI (tampak)     │  │  Fortify (di belakang)  │ │
│  │  Blade/React/Vue │  │  route + logic auth     │ │
│  └──────────────────┘  └─────────────────────────┘ │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  FORTIFY (standalone package)                      │
│  └── Route: /login, /register, /forgot-password,   │
│  │   /email/verify, /two-factor-challenge, dll     │
│  └── Tanpa UI — kamu bikin frontend sendiri        │
└────────────────────────────────────────────────────┘
```

### Kapan Pakai Fortify?

| Skenario | Pakai |
|----------|-------|
| Mau auth langsung jadi (web app) | **Starter Kit** (pilih stack via `laravel new`) — sudah包含 Fortify |
| Backend API + frontend SPA sendiri (Next.js/React/Vue) | **Fortify** (register/login/reset) + **Sanctum** (session/token) |
| Mau bikin auth manual dari awal, tanpa package | **Built-in Auth** (`Auth::attempt()`) — tidak perlu Fortify |

### Fortify vs Sanctum — Bukan Saingan

Fortify dan Sanctum **bukan competing packages** — mereka solve masalah berbeda:

| Fortify | Sanctum |
|---------|---------|
| Menyediakan **route untuk register, login, reset password, verifikasi email, 2FA, passkeys** | Menyediakan **API token management + session authentication** |
| Frontend-agnostic (kamu bikin UI sendiri) | Hybrid (cookie untuk SPA, token untuk API/mobile) |
| Bisa dipasangkan dengan Sanctum | Bisa dipasangkan dengan Fortify |

> **Fortify + Sanctum adalah stack yang umum**: Fortify untuk register/login/reset password, Sanctum untuk session auth & API token.

### Fitur Fortify

| Fitur | Route |
|-------|-------|
| Registrasi | `POST /register` |
| Login | `POST /login` |
| Logout | `POST /logout` |
| Lupa password | `POST /forgot-password` |
| Reset password | `POST /reset-password` |
| Verifikasi email | `POST /email/verification-notification` |
| Konfirmasi password | `POST /user/confirm-password` |
| 2FA (TOTP) | `POST /user/two-factor-authentication` |
| Passkeys (WebAuthn) | `POST /passkeys/login`, `POST /user/passkeys` |

---

## Passport — OAuth2 Server

### Apa itu Passport?

**Laravel Passport** adalah implementasi **OAuth 2.0** penuh untuk Laravel. Passport mengubah aplikasi Laravel menjadi **OAuth2 server** yang bisa mengeluarkan access token untuk client pihak ketiga — seperti halnya Twitter API, GitHub API, atau Google API.

Passport dibangun di atas [League OAuth2 server](https://github.com/thephpleague/oauth2-server), library OAuth2 standar untuk PHP.

### Bagaimana Passport Bekerja?

Passport mengimplementasikan **flow OAuth2 standar**. Secara umum, alurnya:

```
Third-Party App              Laravel (Passport)              User
      │                            │                         │
      │── Redirect user ke ───────→│                         │
      │   /oauth/authorize         │                         │
      │                            │── Tampilkan halaman ───→│
      │                            │   persetujuan ("Allow") │
      │                            │←── User menyetujui ─────│
      │                            │                         │
      │←── Redirect ke callback ───│                         │
      │    (dengan authorization   │                         │
      │     code)                  │                         │
      │                            │                         │
      │── POST /oauth/token ──────→│                         │
      │   (code + client_id +      │ Tukar code → JWT        │
      │    client_secret)          │                         │
      │←── Access Token + ─────────│                         │
      │    Refresh Token           │                         │
      │                            │                         │
      │── GET /api/resource ──────→│ Verifikasi JWT          │
      │   Authorization: Bearer    │ (tanpa query DB!)       │
      │←── 200 OK ─────────────────│                         │
```

### Karakteristik Passport

- **JWT (JSON Web Token)** — token berisi payload terenkripsi yang bisa diverifikasi tanpa database
- **OAuth 2.0 compliant** — mengikuti standar OAuth2 resmi
- **Multi-client** — bisa punya banyak client (web, mobile, desktop) dengan konfigurasi berbeda
- **Refresh token** — access token bisa di-refresh tanpa login ulang
- **Scopes formal** — izin token sesuai standar OAuth2

### Grant Types OAuth2 di Passport

Passport mendukung beberapa **grant type** — yaitu cara client mendapatkan token:

| Grant Type | Cara Kerja | Cocok Untuk |
|-----------|-----------|-------------|
| **Authorization Code** | User redirect ke halaman login → setuju → dapat authorization code → tukar dengan access token | Aplikasi web dengan server backend (confidential clients) |
| **Authorization Code + PKCE** | Sama seperti di atas, tapi pakai kode verifikasi. **Tidak perlu client secret** | SPA & mobile apps (public clients) |
| **Client Credentials** | Client langsung minta token pakai client ID + secret sendiri | Machine-to-machine (server ke server, cron job) |
| **Personal Access Tokens** | User membuat token langsung dari UI (mirip Sanctum) | Developer tools, API eksperimen |
| **Password Grant** ⚠️ | Login dengan email + password langsung dapat token | First-party client (tidak direkomendasikan lagi) |
| **Device Authorization** | User masukin kode dari TV/game console ke device lain | Browserless devices (TV, game console) |

### Konsep Penting Passport

| Konsep | Penjelasan |
|--------|-----------|
| **Client** | Aplikasi pihak ketiga yang minta akses ke API Anda. Punya `client_id` dan `client_secret` |
| **Confidential client** | Client yang bisa menyimpan rahasia (client secret) — biasanya server backend |
| **Public client** | Client yang tidak bisa menyimpan rahasia — SPA, mobile app (pakai PKCE) |
| **Scopes** | Izin spesifik yang diminta client (contoh: `user:read`, `orders:create`) |
| **Authorization code** | Kode sementara yang ditukar dengan access token |
| **Access token** | Token (JWT) yang digunakan untuk akses API |
| **Refresh token** | Token untuk mendapatkan access token baru tanpa login ulang |

### Passport vs Sanctum — Kapan OAuth2 Diperlukan?

Passport diperlukan jika Anda butuh flow OAuth2 yang lengkap, terutama jika **pihak ketiga** (bukan aplikasi Anda sendiri) perlu mengakses API. Sanctum tidak mendukung OAuth2 — lebih sederhana tapi kurang fleksibel untuk third-party.

---

## Socialite — OAuth Login via Provider Eksternal

### Apa itu Socialite?

**Laravel Socialite** adalah package untuk **OAuth authentication** via provider eksternal — Google, GitHub, Facebook, Apple, Twitter, LinkedIn, GitLab, Bitbucket, dan lainnya. Socialite menangani kerumitan OAuth flow (redirect, callback, token exchange) sehingga kamu tinggal fokus ke logic aplikasi.

### Peran Socialite dalam Auth Laravel

Socialite melengkapi stack authentication Laravel — ia tidak menggantikan Sanctum/Fortify/Passport, tapi bekerja **bersamaan** dengan mereka:

```
┌─────────────────────────────────────────────────────────────────┐
│                     STACK AUTH LARAVEL                          │
│                                                                 │
│   Sanctum / Fortify / Passport                                  │
│   (email & password, session, token, 2FA, OAuth2 server)        │
│         ┼                                                       │
│   Socialite                                                     │
│   (login via Google, GitHub, Facebook, Apple, dll)              │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ User bisa login dengan:                                  │   │
│   │ • Email + password (via Sanctum/Fortify/auth bawaan)     │   │
│   │ • Google / GitHub / dll   (via Socialite)                │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Konsep Penting Socialite

| Konsep | Penjelasan |
|--------|-----------|
| **Provider** | Layanan eksternal yang menyediakan OAuth (Google, GitHub, Facebook, Apple, dll) |
| **Redirect** | Arahkan user ke halaman login provider (Google/GitHub/dll) |
| **Callback** | URL yang dipanggil provider setelah user setuju — berisi authorization code |
| **Social Account** | Tabel pivot yang menghubungkan user Laravel dengan akun provider (`provider` + `provider_id`) |
| **Stateless** | Mode tanpa session — cocok untuk API / SPA yang handle redirect sendiri |

### Socialite vs Passport — Keduanya OAuth, Tapi Berbeda

| Aspek | Socialite | Passport |
|-------|-----------|----------|
| **Arah** | **Client** — aplikasi kamu login via provider lain (Google, GitHub) | **Server** — aplikasi kamu jadi provider OAuth2 untuk aplikasi lain |
| **Fungsi** | "Login dengan Google/GitHub" | "Bikin API yang bisa diakses aplikasi lain via OAuth2" |
| **Flow** | Redirect ke provider → callback → login | Aplikasi lain redirect ke kamu → setuju → kasih token |

> **Intinya**: Socialite untuk **login via akun Google/GitHub**. Passport untuk **jadikan aplikasi kamu sebagai server OAuth2** yang bisa dipakai aplikasi lain.

Untuk penjelasan lebih detail tentang konsep OAuth, SSO, dan teknologi authentication secara umum, lihat file [`teknologi-auth.md`](./teknologi-auth.md).

---

## Perbandingan Sanctum vs Passport

| Aspek | Sanctum | Passport |
|-------|---------|----------|
| **Jenis Token** | Opaque token (string acak, SHA-256) | JWT (JSON Web Token) |
| **Protokol** | Hybrid (session cookie + token) | OAuth 2.0 penuh |
| **Cek Database?** | ✅ Setiap request (cocokkan hash token) | ❌ Tidak perlu (verifikasi signature JWT) |
| **Revoke Token** | ✅ Instan — hapus dari DB, langsung mati | ⚠️ Perlu blocklist atau tunggu expired |
| **Refresh Token** | ❌ Tidak ada | ✅ Ada |
| **SPA Auth** | ✅ Built-in via session cookie | ✅ Via `CreateFreshApiToken` middleware atau PKCE |
| **Mobile Auth** | ✅ Sederhana — login → dapat token | ✅ Via PKCE flow |
| **Third-Party Clients** | ❌ Terbatas (hanya API token sederhana) | ✅ OAuth2 standar, multi-client |
| **Client Management** | ❌ Tidak ada | ✅ Multi-client dengan client ID & secret |
| **Scopes/Abilities** | ✅ Abilities (sederhana) | ✅ Scopes (formal, OAuth2) |
| **Kompleksitas** | Rendah — sedikit konfigurasi | Tinggi — banyak komponen & konfigurasi |
| **Rekomendasi Laravel** | ✅ **Direkomendasikan** untuk sebagian besar kasus | ❌ Hanya untuk kasus spesifik OAuth2 |

### Flow Diagram: Sanctum vs Passport

**Sanctum (Sederhana):**
```
Mobile App                     Laravel (Sanctum)
   │                                  │
   │── POST /sanctum/token ──────────→│ Cek email + password
   │   (email + password)             │
   │←── { token: "1|abc123..." } ─────│ Buat token, hash, simpan
   │                                  │
   │── GET /api/user ────────────────→│ Cari token di DB
   │   Authorization: Bearer 1|abc    │ Hash → cocokkan
   │←── { user: ... } ────────────────│
```

**Passport (OAuth2):**
```
Third-Party App                  Laravel (Passport)
   │                                     │
   │── Redirect user ───────────────────→│ /oauth/authorize
   │   (client_id + redirect_uri)        │ User login + setuju
   │←── Redirect dengan code ────────────│
   │                                     │
   │── POST /oauth/token ───────────────→│ Tukar code → JWT
   │   (code + client_secret)            │ (sign, verify signature)
   │←── { access_token, refresh_token } ─│
   │                                     │
   │── GET /api/user ───────────────────→│ Verifikasi JWT
   │   Authorization: Bearer JWT...      │ (tanpa DB query!)
   │←── { user: ... } ───────────────────│
```

### Kelebihan & Kekurangan

| | Sanctum | Passport |
|---|---|---|
| ✅ Kelebihan | Sederhana, hybrid, revoke instan, recommended | OAuth2 lengkap, JWT stateless, refresh token, multi-client |
| ❌ Kekurangan | Tidak ada refresh token, query DB tiap request, tidak cocok third-party | Kompleks, revoke sulit, overkill untuk API sederhana |

---

## Kapan Memilih Sanctum vs Passport?

| Kondisi | Pilihan |
|---------|---------|
| API untuk SPA sendiri (Next.js, Vue, React) | **Sanctum** ✅ |
| API untuk mobile app (iOS/Android) | **Sanctum** ✅ |
| API sederhana untuk first-party frontend | **Sanctum** ✅ |
| Headless auth backend (register, login, 2FA tanpa UI) | **Fortify** ✅ |
| SPA + backend auth lengkap | **Fortify** + **Sanctum** ✅ |
| Butuh refresh token | **Passport** ✅ |
| API publik untuk third-party developers | **Passport** ✅ |
| Machine-to-machine (internal API) | **Passport** (Client Credentials) ✅ |
| Butuh OAuth2 standar (Authorization Code) | **Passport** ✅ |
| Developer API / personal tokens | **Sanctum** ✅ (lebih sederhana) |
| Aplikasi monolith + API | **Sanctum** ✅ |
| Microservices yang butuh JWT stateless | **Passport** ✅ |

### Aturan Praktis

1. **Default ke Sanctum** — 90% kasus cukup pakai Sanctum
2. **Pakai Starter Kit** (`laravel new`) jika ingin semuanya jadi (UI + backend) dalam 5 menit
3. **Pakai Fortify** jika butuh backend auth (register, login, 2FA) tanpa ingin bikin dari nol — biasanya dipasangkan dengan Sanctum untuk SPA
4. **Pilih Passport** hanya jika Anda benar-benar membutuhkan OAuth2 — yaitu ketika **pihak ketiga** (bukan aplikasi Anda sendiri) perlu mengakses API dengan flow OAuth2 standar
5. **Kalau ragu, pakai Sanctum** — lebih sederhana, recommended, dan cukup untuk sebagian besar skenario

> **Saran dari Laravel**: Untuk sebagian besar aplikasi, **Sanctum sudah lebih dari cukup**. Passport hanya diperlukan jika Anda benar-benar membutuhkan OAuth2 server untuk third-party clients.

---

## Ringkasan Stack Authentication Laravel

| Jenis Aplikasi | Stack yang Direkomendasikan |
|---|---|---|
| Monolith (web browser) | Built-in Auth + Starter Kit (Breeze/Jetstream) |
| Monolith + API | Built-in Auth + **Sanctum** |
| SPA terpisah + Laravel backend | **Fortify** + **Sanctum** |
| API untuk mobile app | **Sanctum** |
| API publik untuk third-party | **Passport** (OAuth2 penuh) |
| Machine-to-machine (internal) | **Passport** (Client Credentials) |
| Headless auth (tanpa UI) | **Fortify** (bisa ditambah **Sanctum** untuk API) |
| Login via Google/GitHub/dll | **Socialite** (dipasangkan dengan auth stack manapun) |

### Arsitektur Umum Berdasarkan Stack

**Monolith + Sanctum:**
```
Browser ←→ Laravel (session cookie)
Mobile  ←→ Laravel (Bearer token)
```

**SPA + Sanctum:**
```
Next.js/Vue ←→ Laravel API (session cookie, same domain)
Mobile      ←→ Laravel API (Bearer token)
```

**API Publik + Passport:**
```
Third-Party App ──OAuth2──→ Laravel API (JWT)
Mobile (PKCE)    ──OAuth2──→ Laravel API (JWT)
Own SPA (PKCE)   ──OAuth2──→ Laravel API (JWT)
```

> **Intinya**: Laravel menyediakan empat pendekatan authentication — **Starter Kits** (session, langsung jadi via `laravel new`), **Sanctum** (hybrid, recommended untuk API modern), **Fortify** (headless auth backend package), dan **Passport** (OAuth2 penuh untuk third-party). Ditambah **Socialite** untuk OAuth login via provider eksternal (Google, GitHub, dll). Pilih sesuai kebutuhan, dan **default ke Sanctum jika ragu**.

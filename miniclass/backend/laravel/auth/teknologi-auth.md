# Istilah & Teknologi dalam Authentication

Sistem authentication memiliki banyak istilah dan teknologi yang perlu dipahami. Berikut penjelasan lengkapnya.

---

## Daftar Istilah Penting

### Identity & Credentials

| Istilah | Penjelasan |
|---|---|
| **Identity** | Identitas unik seorang pengguna (bisa berupa ID, email, username). |
| **Credentials** | Kredensial yang membuktikan identitas, kombinasi dari identifier + secret (misal: email + password). |
| **Password** | Rahasia yang hanya diketahui user, digunakan sebagai bukti kepemilikan identitas. |
| **Password Hash** | Password yang sudah dienkripsi satu arah (bcrypt, argon2) sebelum disimpan di database. |
| **Salt** | Data acak yang ditambahkan ke password sebelum di-hash untuk mencegah serangan rainbow table. |

### Session & Token

| Istilah | Penjelasan |
|---|---|
| **Session** | Data sementara di server yang menyimpan status login user. Biasanya dirujuk via cookie di browser. |
| **Session ID** | Identitas unik session yang dikirim sebagai cookie ke browser. |
| **Cookie** | File kecil di browser yang menyimpan session ID atau token. |
| **Token** | String unik yang mewakili otorisasi user, dikirim di setiap request (biasanya di header `Authorization`). |
| **JWT (JSON Web Token)** | Token yang berisi data terenkode (payload) yang bisa diverifikasi tanpa perlu database. Format: `header.payload.signature`. |
| **Access Token** | Token yang digunakan untuk mengakses resource (biasanya berlaku pendek: 15-60 menit). |
| **Refresh Token** | Token untuk mendapatkan access token baru tanpa login ulang (berlaku lebih lama: hari/bulan). |
| **CSRF Token** | Token anti serangan Cross-Site Request Forgery. |

### Authentication Flow

| Istilah | Penjelasan |
|---|---|
| **Login** | Proses memasukkan credentials untuk memulai session terautentikasi. |
| **Logout** | Proses mengakhiri session dan menghapus data authentication. |
| **Register / Sign Up** | Proses pembuatan akun baru. |
| **Remember Me** | Fitur agar user tetap login meskipun browser ditutup (menggunakan token di cookie). |
| **Password Reset** | Proses mengganti password yang lupa melalui email atau mekanisme lain. |
| **Email Verification** | Verifikasi bahwa user benar-benar memiliki email yang didaftarkan. |
| **Rate Limiting / Throttling** | Pembatasan jumlah percobaan login untuk mencegah brute force attack. |
| **Session Fixation** | Serangan di mana penyerang memaksa user menggunakan session ID yang sudah diketahui. |

### Security Concepts

| Istilah | Penjelasan |
|---|---|
| **Brute Force Attack** | Percobaan login berulang dengan berbagai kombinasi password. |
| **Brute Force Protection** | Mekanisme yang memblokir user setelah beberapa kali gagal login. |
| **MFA / 2FA (Multi-Factor Authentication)** | Authentication menggunakan dua atau lebih faktor (password + OTP, dll). |
| **OTP (One-Time Password)** | Password sekali pakai yang dikirim via SMS/email/authenticator app. |
| **TOTP (Time-based OTP)** | OTP yang berubah setiap 30-60 detik (Google Authenticator, Authy). |
| **Biometric Authentication** | Authentication menggunakan sidik jari, wajah, suara, atau retina. |
| **OAuth (Open Authorization)** | Protokol yang memungkinkan login menggunakan akun pihak ketiga (Google, GitHub, dll). |
| **SSO (Single Sign-On)** | Satu kali login bisa mengakses banyak aplikasi (contoh: login Google untuk Gmail, Drive, YouTube). |
| **Magic Link** | Link ajaib yang dikirim ke email — sekali klik langsung login tanpa password. |
| **Passwordless Authentication** | Authentication tanpa password (magic link, OTP, biometric). |

---

## Teknologi dalam Authentication System

### Session-Based Authentication

Cara klasik dan masih paling umum untuk aplikasi web monolith.

**Prinsip**: *Authentication state dikelola di server (stateful).*

#### Alur Lengkap

<img src="/miniclass/backend/assets/auth/session-auth.jpg" style="border-radius: 5px;" alt="gambar-sessionauth-laravel" width=640px></img>

#### Bagaimana Database Menyimpan Session?

Session disimpan di **server / database**, bukan di client. Berikut contoh tabel session di database:

```js
TABLE: sessions
┌──────────────┬─────────────┬────────────────────┬──────────────────────┐
│ id (PK)      │ user_id     │ payload            │ expires_at           │
├──────────────┼─────────────┼────────────────────┼──────────────────────┤
│ abc123...    │ 42          │ {ip: ...,          │ 2025-06-01 13:00:00  │
│              │             │  user_agent: ...}  │                      │
│ def456...    │ 42          │ {ip: ...,          │ 2025-06-01 13:30:00  │
│              │             │  user_agent: ...}  │                      │
│ ghi789...    │ 17          │ {ip: ...,          │ 2025-06-01 14:00:00  │
│              │             │  user_agent: ...}  │                      │
└──────────────┴─────────────┴────────────────────┴──────────────────────┘
```

Tempat penyimpanan session bisa:
- **In-memory** (RAM) — cepat, tapi hilang saat server restart ❌
- **Database relational** (MySQL/PostgreSQL) — persisten ✅
- **Redis / Memcached** — cepat dan persisten, paling umum di production ✅
- **File system** — jarang dipakai di production

#### Kelebihan

- ✅ Mudah di-invalidate (tinggal hapus session dari database → user langsung logout)
- ✅ Aman dari token hijacking (session ID acak, tidak mengandung data user)
- ✅ State dikelola server — lebih mudah dikontrol

#### Kekurangan

- ❌ **Rentan CSRF (Cross-Site Request Forgery)** — karena cookie otomatis terkirim, penyerang bisa memanfaatkannya
- ❌ **Scaling sulit** — server harus tetap menyimpan session data. Di cloud terdistribusi, session jadi *bottleneck*. Solusi: **sticky session** atau **Redis terpusat**
- ❌ **Memory server terbebani** — makin banyak user aktif, makin besar session data yang harus dikelola

#### Tools Populer

Laravel Auth, Express Session, Django Session, Spring Security, ASP.NET Core Identity.

---

### Token-Based Authentication

Solusi untuk mengatasi masalah scaling pada session-based auth. Sangat cocok untuk API, SPA, dan mobile apps.

**Prinsip**: *Authentication state dikelola di client (stateless).*

#### Alur Lengkap (dengan JWT)

<img src="/miniclass/backend/assets/auth/token-auth.jpg" style="border-radius: 5px;" alt="gambar-token-auth-laravel" width=640px></img>

#### Kenapa JWT Tidak Perlu Database?

JWT berisi **semua informasi yang dibutuhkan** di dalam token itu sendiri:

```js
JWT = base64(header) + "." + base64(payload) + "." + signature
```

Contoh payload (didecode):
```json
{
  "sub": "42",           ← user ID
  "name": "John Doe",
  "role": "admin",
  "iat": 1717200000,     ← issued at
  "exp": 1717203600      ← expires (60 menit)
}
```

Server cukup memverifikasi **signature** menggunakan **private key / secret**. Jika signature valid, data di payload bisa dipercaya — **tanpa perlu query database sama sekali**. Ini yang membuat JWT sangat efisien untuk distributed systems.

#### Jenis Token

| Jenis | Verifikasi | Database? | Contoh |
|---|---|---|---|
| **JWT** | Signature (HS256/RS256) | ❌ Tidak perlu | `eyJhbGci...` |
| **Opaque Token** | Acak, harus dicocokkan | ✅ Wajib | `sk_live_3f8a...` |
| **Sanctum Token** | Opaque (milik Laravel) | ✅ Dicocokkan di DB | `1\|abc123...` |
| **Refresh Token** | Opaque / JWT | ✅ Biasanya disimpan | `rft_xyz...` |

#### Kelebihan

- ✅ **Scaling effortless** — server tidak perlu menyimpan state, cocok untuk cloud terdistribusi
- ✅ **Tidak ada CSRF** — token dikirim manual via header, bukan otomatis seperti cookie
- ✅ **Lebih ringan untuk server** — tidak perlu query database di setiap request
- ✅ **Cocok untuk SPA / Mobile** — token bisa disimpan dan dikirim dari mana saja

#### Kekurangan

- ❌ **Token bisa di-hijack** — jika token dicuri, penyerang bisa akses sampai token expired
- ❌ **Sulit di-invalidate** — tidak bisa "logout paksa" sebelum token expired (kecuali pakai blocklist/deny list)
- ❌ **Payload bisa dibaca** — isi JWT hanya di-sign, bukan di-encrypt. Jangan simpan data sensitif di payload!
- ❌ **Tidak cocok untuk background auth** — jika ada proses server-side yang butuh akses user (cron job, queue), JWT yang sudah expire jadi masalah

#### Kapan Pake Yang Mana?

| Skenario | Pilih |
|---|---|
| Web monolith tradisional | Session-Based |
| API untuk SPA / Mobile | Token-Based |
| Distributed cloud / microservices | Token-Based (JWT) |
| Butuh kontrol penuh (force logout cepat) | Session-Based |
| Butuh scalability tinggi | Token-Based |

### OAuth 2.0 & SSO

Protokol standar industri untuk delegasi akses.

```
User ──> Aplikasi ──> Google/GitHub
                  │
                  ├── "Bolehkah user ini login?"
                  ├── User menyetujui
                  └── Dapat token akses
```

**Penyedia OAuth populer**: Google, GitHub, Facebook, Apple, Microsoft.

### Passwordless Authentication

Autentikasi tanpa password tradisional:

- **Magic Link**: klik link di email → langsung login.
- **OTP**: kode sekali pakai via SMS/email.
- **WebAuthn / Passkeys**: standar baru dari FIDO Alliance menggunakan biometric atau hardware token.
- **Push Notification**: notifikasi ke HP "Apakah ini kamu?" → tap approve.

---

## Tools & Library Authentication Populer (2025)

Berikut teknologi authentication yang banyak digunakan di ekosistem web development saat ini:

### 🔷 Laravel Ecosystem

| Tool | Deskripsi |
|---|---|
| **Laravel Breeze** | Starter kit minimalis untuk authentication Laravel (Blade, Livewire, React, Vue). Ringan dan sederhana. |
| **Laravel Jetstream** | Starter kit lebih lengkap dengan fitur team management, 2FA, API tokens via Sanctum. |
| **Laravel Sanctum** | Paket authentication hybrid untuk Laravel — otomatis pakai cookie untuk web dan token untuk API. **Recommended** untuk API Laravel. |
| **Laravel Passport** | Implementasi OAuth2 penuh untuk Laravel. Cocok jika benar-benar butuh OAuth2. |
| **Laravel Fortify** | Headless authentication backend — menyediakan route & controller auth tanpa UI. Cocok dipasangkan dengan frontend SPA. |
| **Laravel Socialite** | OAuth login via provider eksternal (Google, GitHub, Facebook, Twitter, dll). |

### 🔷 PHP / Laravel (Pihak Ketiga)

| Tool | Deskripsi |
|---|---|
| **Laravel Permission (Spatie)** | Manajemen role & permission untuk Laravel (authorization, bukan authentication langsung). |
| **Sentinel (Cartalyst)** | Authentication & authorization framework untuk PHP. Cocok untuk legacy project atau non-Laravel. |

### 🔷 Multi-Platform / Lintas Framework

| Tool | Deskripsi |
|---|---|
| **Better Auth** | Library authentication modern untuk berbagai framework (termasuk Laravel). Mendukung passwordless, MFA, session management, webhooks, dan banyak fitur siap pakai. Cocok jika ingin solusi all-in-one di luar ekosistem Laravel murni. |
| **Auth0** | Platform authentication cloud (SaaS). Mendukung login sosial, MFA, passwordless, SSO, user management dashboards. Tinggal integrasi API, urusan security dikelola Auth0. |
| **Clerk** | Authentication & user management modern. Fokus pada developer experience, UI komponen siap pakai untuk React, Next.js, Vue. |
| **Supabase Auth** | Authentication bawaan Supabase (Firebase alternatif). Mendukung email/password, OAuth, magic link, MFA. Terintegrasi dengan database PostgreSQL. |
| **Firebase Authentication** | Authentication dari Google. Gratis untuk skala kecil-menengah. Mendukung email, OAuth, phone auth, anonymous auth. |
| **NextAuth.js (Auth.js)** | Authentication untuk Next.js dan framework modern lainnya. Support OAuth, credentials, email, database adapters. |
| **Lucia** | Library authentication lightweight untuk JavaScript/TypeScript. Sederhana dan fleksibel, bisa dipakai dengan berbagai database. |
| **Kinde** | Platform auth modern dengan fokus kemudahan integrasi. Menyediakan pre-built login pages. |

### 🔷 .NET Ecosystem

| Tool | Deskripsi |
|---|---|
| **ASP.NET Core Identity** | Authentication & authorization bawaan .NET. Mendukung login, register, role management, 2FA, external login. Lengkap dan terintegrasi dengan Entity Framework. |
| **Duende IdentityServer** | Implementasi OAuth 2.0 & OpenID Connect untuk .NET. Standar industri untuk SSO di ekosistem .NET. |

### 🔷 Python Ecosystem

| Tool | Deskripsi |
|---|---|
| **Django Authentication** | Sistem auth bawaan Django — lengkap dengan user model, session, permissions, admin panel. |
| **Flask-Login** | Session-based authentication untuk Flask. |
| **FastAPI Users** | Authentication & user management untuk FastAPI. Support SQLAlchemy, MongoDB, OAuth, JWT. |

### 🔷 Node.js Ecosystem

| Tool | Deskripsi |
|---|---|
| **Passport.js** | Middleware authentication modular untuk Express. Ratusan strategi (local, OAuth, JWT, dll). |
| **JWT (jsonwebtoken)** | Library untuk membuat & memverifikasi JWT. Bukan auth lengkap, tapi komponen penting. |
| **bcrypt** | Library untuk hashing password (di semua bahasa). |

### 🔷 Go Ecosystem

| Tool | Deskripsi |
|---|---|
| **Goth** | OAuth provider untuk Go. Multi-strategy (sama seperti Passport.js). |
| **Casbin** | Authorization library yang bisa dipasangkan dengan authentication system manapun. |

### 🔷 Java Ecosystem

| Tool | Deskripsi |
|---|---|
| **Spring Security** | Framework authentication & authorization paling populer di Java. Mendukung LDAP, OAuth2, JWT, form login, SSO. |
| **Keycloak** | Platform auth open-source lengkap (SSO, OAuth2, SAML, user federation). Bisa di-self-host atau cloud. |

---

## Tips Memilih Teknologi Authentication

1. **Aplikasi Laravel monolith** → pakai **Breeze** atau **Jetstream** (built-in auth sudah cukup).
2. **Laravel + API/SPA** → pakai **Sanctum**.
3. **Butuh OAuth2 penuh** (third-party client) → pakai **Passport**.
4. **Butuh cloud solution** (enggan urus infrastruktur auth) → **Auth0**, **Clerk**, **Supabase Auth**, atau **Firebase Auth**.
5. **Aplikasi non-Laravel** yang butuh solusi all-in-one → **Better Auth**, **Auth.js**, atau **Lucia**.
6. **Passwordless / modern auth** → **Magic link**, **OTP**, atau **WebAuthn/Passkeys**.
7. **Multi-platform / ingin fleksibel** → **Better Auth** bisa jadi pilihan lintas framework.

> Pada akhirnya, pilihan teknologi authentication tergantung pada kebutuhan: seberapa besar skala, apakah butuh OAuth, berapa banyak developer, dan apakah ingin manage sendiri atau pakai SaaS.

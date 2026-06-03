# Apa itu Authentication?

**Authentication** (otentikasi) adalah proses memverifikasi identitas seseorang atau sistem. Sederhananya, authentication menjawab pertanyaan: **"Siapa kamu?"** — yaitu memastikan bahwa pengguna benar-benar merupakan orang yang mereka klaim.

## Cara Kerja Authentication

Secara umum, authentication bekerja dengan memeriksa satu atau lebih dari tiga faktor berikut:

1. **Something You Know** — sesuatu yang hanya diketahui pengguna, seperti password, PIN, atau jawaban pertanyaan keamanan.
2. **Something You Have** — sesuatu yang dimiliki pengguna, seperti smartphone (OTP), token fisik, atau kartu akses.
3. **Something You Are** — sesuatu yang melekat pada diri pengguna, seperti sidik jari, wajah (*face recognition*), atau retina mata.

Semakin banyak faktor yang digunakan, semakin aman sistem authentication-nya (dikenal sebagai **Multi-Factor Authentication / MFA**).

## Authentication vs Authorization

<img src="/miniclass/backend/assets/auth/authentication_and_authorization.webp" style="border-radius: 5px;" alt="gambar-authentication-vs-authorization" width=640px></img>

Kedua istilah ini sering tertukar, padahal berbeda:

| Authentication | Authorization |
|---|---|
| Memverifikasi identitas | Menentukan akses/izin |
| "Siapa kamu?" | "Apa yang boleh kamu lakukan?" |
| Dilakukan pertama kali | Dilakukan setelah authentication |
| Contoh: login dengan email & password | Contoh: user biasa vs admin punya akses berbeda |

## Contoh dalam Kehidupan Sehari-hari

- **Login ke aplikasi**: memasukkan email dan password → authentication.
- **Scan wajah buka HP**: Face ID → authentication.
- **OTP dari bank saat transaksi**: something you have → authentication.

## Authentication dalam Web Development

Di pengembangan web, authentication biasanya diimplementasikan menggunakan:

- **Session-based**: data login disimpan di server (session), cookie dikirim ke browser.
- **Token-based**: token (seperti JWT) dikirim ke client dan disertakan di setiap request.
- **OAuth / SSO**: login menggunakan akun pihak ketiga seperti Google, GitHub, atau Facebook.

Intinya, **authentication adalah gerbang pertama keamanan aplikasi** — tanpa authentication yang baik, siapa pun bisa mengakses data dan fitur sensitif.


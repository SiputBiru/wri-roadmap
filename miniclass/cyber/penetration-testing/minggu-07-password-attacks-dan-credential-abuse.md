# Chapter 7: Password Attacks dan Credential Abuse

## Password & Kredensial

![cryptography](/miniclass/cyber/assets/penetration-testing/cryptography.jpg)

**Password** adalah sekumpulan karakter rahasia yang digunakan untuk membuktikan identitas pengguna pada saat proses autentikasi. Sementara itu, **kredensial** dapat berupa paket informasi yang lebih luas, biasanya terdiri dari **identitas** (seperti username atau email) dan **bukti rahasia** (password, token, atau biometrik) yang fungsinya sebagai pengidentifikasi unik.

Dalam konteks penetration testing, serangan terhadap password atau kredensial umumnya berkaitan dengan teknik **brute-force** untuk memperoleh akses ke informasi terenkripsi atau untuk mendapatkan user dengan hak akses yang lebih tinggi (privilege escalation). Oleh karena itu, serangan pada tahap ini sangat krusial, karena jika autentikasi berhasil ditembus, penyerang pada dasarnya telah memperoleh “kunci resmi” untuk masuk ke dalam sistem tersebut. 

Kredensial umumnya bersifat **RAHASIA**. Namun apakah dengan "rahasia" saja cukup?

Mari kita bahas lebih lanjut.

---

## Common Risks

Terdapat berbagai jenis resiko kerentanan yang sering dimanfaatkan oleh penyerang untuk melewati sistem autentikasi atau bahkan mendekripsi suatu file tanpa harus mengetahui password atau kredensial yang sebenarnya:

### 1. Default Credentials

Credential bawaan vendor atau instalasi default belum diubah.

::: warning Studi Kasus
Sebuah kampus baru saja meluncurkan sistem akademik baru. Untuk mempermudah aktivasi akun bagi ribuan pengguna, pihak IT menetapkan NIDN (Nomor Induk Dosen Nasional) sebagai **username dan password default**. Penyerang yang mengetahui pola ini dapat dengan mudah mencari data NIDN dan nama dosen melalui laman publik profil kampus atau PDDIKTI untuk melakukan pengambilalihan akun secara massal sebelum dosen sempat mengganti password mereka.
:::

### 2. Weak Credentials

Password terlalu sederhana, mudah ditebak, atau mengikuti pola yang umum.

::: warning Studi Kasus
Dalam sebuah audit, ditemukan akun `admin` yang menggunakan password `Admin#1234`. Password ini sangat mudah ditebak melalui teknik bruteforce menggunakan common password karena mengikuti pola nama organisasi dan tahun yang sering digunakan oleh pengguna untuk memudahkan ingatan.
:::

### 3. Credential Reuse

Satu kredensial dipakai ulang pada beberapa service atau akun.

::: warning Studi Kasus
Seorang pengguna menggunakan **password yang sama** untuk akun LinkedIn, Instagram, dan akun di sistem kantor. Ketika sistem kantor mengalami kebocoran data, penyerang mengambil database tersebut dan mencoba kredensial yang sama untuk menyusup ke platform/sistem yang lain.
:::

### 4. Session Leak

Disebabkan implementasi sistem yang buruk yang berujung pada **session hijacking**.

::: warning Studi Kasus
Sebuah aplikasi web memiliki fitur "Testimoni" yang tidak melakukan sanitasi input, sehingga rentan terhadap ***Stored XSS***. Penyerang mengunggah skrip JavaScript berbahaya yang dirancang untuk mencuri session cookie. Penyerang kemudian menggunakan token tersebut untuk melakukan **session hijacking**, mendapatkan akses penuh ke akun admin tanpa perlu melakukan serangan terhadap password sama sekali.
:::

---

## Password Attack

Teknik yang digunakan penyerang untuk mendapatkan akses tidak sah ke akun pengguna adalah dengan cara menebak semua kemungkinan kata sandi. Serangan ini sebenarnya memanfaatkan **kelemahan manusia dalam memilih kata sandi yang mudah diingat namun lemah secara keamanan**. Dalam fase eksploitasi, ini adalah cara paling efektif untuk mendapatkan *initial access* ke dalam infrastruktur target.

Berikut merupakan beberapa metode umum yang sering digunakan:

1. **Bruteforce Attack**

   Intinya **semua dicobain**, dari `0000` sampai `9999`, atau `aaaaa` sampai `ZZZZZ`. Meskipun secara teoritis dijamin akan berhasil menemukan password apa pun, serangan ini sangat tidak reliabel dan membutuhkan sumber daya komputasi yang sangat besar dan waktu yang sangat lama jika kata sandi target cukup panjang dan kompleks.
2. **Dictionary Attack**

   Metode ini dapat dikatakan bruteforce juga, namun alih-alih menghasilkan kombinasi secara iteratif, metode ini menggunakan **daftar kata yang sudah ditentukan sebelumnya (wordlist)**. Daftar ini berisi ribuan hingga jutaan kata yang sering digunakan sebagai password, seperti kata-kata dari kamus, nama populer, hingga hasil kebocoran data (data breach) sebelumnya. Serangan ini jauh lebih efisien dan cepat daripada bruteforce karena hanya mencoba kata-kata dengan probabilitas tinggi.
3. **Rainbow Table Attack**

   Hampir mirip dengan sebelumnya, serangan ini menggunakan **tabel hasil perhitungan hash yang sudah dihitung sebelumnya (pre-computed hash)**. Penyerang tidak perlu menghitung hash saat menyerang. Mereka cukup mencocokkan hash target dengan tabel untuk menemukan password aslinya secara instan. Kelemahannya, serangan ini memerlukan ruang penyimpanan yang sangat besar dan tidak efektif terhadap hash yang menggunakan salt (karakter acak tambahan).
4. **Hybrid Attack**

   Gabungan antara **dictionary attack** dengan **sedikit logika bruteforce**. Penyerang mengambil kata dasar dari wordlist dan menambahkan variasi seperti angka, simbol, atau perubahan kapitalisasi di depan atau di belakang kata tersebut (misalnya: password menjadi P@ssw0rd123). Teknik ini sangat efektif karena merefleksikan kebiasaan umum manusia dalam membuat password yang "terlihat rumit" namun sebenarnya berpola.

---

## Common Credentials

Common Credentials merujuk pada daftar username dan password yang paling sering digunakan secara global (seperti `123456`, `password`, dan `Admin#1234`). Penyerang tidak menebak secara acak, melainkan menggunakan daftar ini untuk melakukan automasi dan mempercepat serangan.

![SecLists](/miniclass/cyber/assets/penetration-testing/SecLists.png)

Salah satu repositori yang paling terkenal untuk kebutuhan ini adalah [**SecLists**](https://github.com/danielmiessler/seclists). Ini adalah koleksi berbagai jenis daftar kata (wordlists) yang digunakan selama pengujian keamanan, mulai dari daftar username, password, hingga payload untuk web attack.

---

## Hash Cracking

Ketika penyerang berhasil menemukan file/data sensitif seperti `/etc/shadow` atau dump database, password tidak serta merta tersimpan dalam teks biasa, melainkan dalam bentuk **hash**.

### Apa itu hash?

**Hash** adalah hasil dari algoritma matematika yang mengubah data (input) dengan ukuran apa pun menjadi deretan karakter dengan panjang yang tetap (fixed length). Dalam konteks keamanan sistem, hashing digunakan untuk menyimpan kata sandi sehingga jika database bocor, penyerang tidak langsung mendapatkan kata sandi dalam bentuk teks asli. Beberapa algoritma hash yang umum dikenal seperti `MD5`, `SHA256`, `bcrypt`, dan `CRC32`.

![hash](/miniclass/cyber/assets/penetration-testing/hash.png)

**Karakteristik utama hash:**

1. **Satu Arah (*One-Way*)**

   Sangat mudah untuk membuat hash dari sebuah input, tetapi secara matematis sangat sulit (hampir mustahil) untuk mengembalikan hash tersebut menjadi input aslinya.
2. **Deterministik**

   Input yang sama akan selalu menghasilkan hash yang sama.
3. **Efek *Avalanche***

   Perubahan sekecil apa pun pada input (misal mengganti satu huruf kecil menjadi huruf besar) akan menghasilkan hash yang benar-benar berbeda.
4. **Tahan Tabrakan (*Collision Resistant*)**

   Sangat sulit untuk menemukan dua input berbeda yang menghasilkan satu hash yang sama. Walaupun ada kemungkinan kecil, namun selama jenis algoritma yang digunakan tidak usang kemungkinan terjadi collision sangat kecil (nyaris 0%).

### Crack The Hash

Walaupun bersifat *one-way*, selama hash tersebut berasal dari *weak password* atau *common password* ada kemungkinan hash tersebut bisa di-crack.

**Cracking** adalah proses mencoba menemukan input asli dari suatu hash menggunakan metode [Password Attack](/miniclass/cyber/penetration-testing/minggu-07-password-attacks-dan-credential-abuse.html#password-attack) yang telah dijelaskan sebelumnya. Umumnya, penyerang melakukan crack menggunakan tools yang telah didesain untuk mengautomasi tugas ini.

Berikut adalah tools yang umum digunakan untuk melakukan cracking:

### 1. Online Tools

![crackstation](/miniclass/cyber/assets/penetration-testing/crackstation.png)

Ini adalah langkah awal paling mudah untuk mengecek apakah hash tersebut merupakan **common password**. Salah satu tools yang dapat digunakan adalah [CrackStation](https://crackstation.net/).

::: info Info
Tidak semua hash ***common passwords*** bisa di-crack menggunakan tools ini.

Tools ini juga tidak bekerja pada ***salted hash***.
:::

### 2. John the Ripper (John)

![john](/miniclass/cyber/assets/penetration-testing/john.png)

`john` sangat bagus untuk mendeteksi tipe hash secara otomatis dan melakukan cracking cepat. Berikut adalah contoh penggunaannya:

```bash
# Cracking menggunakan wordlist
john --wordlist=path/to/wordlist.txt myhashes.txt

# Melihat hasil yang berhasil di-crack
john --show myhashes.txt

# List algoritma hash yang dapat digunakan
john --list=formats

# Mem-spesifikan jenis algoritma yang digunakan
john --format=sha512crypt --wordlist=path/to/wordlist.txt hash_file.txt

# Tanpa dictionary (increment mode)
# Contoh kombinasi yang dihasilkan:
# 0, 1, 2, ..., 9, 00, 01, ..., 09, 10, 11, ..., 99, 000, 001, ...
# Beberapa mode incremental bawaan: digits, alpha, alnum, dll
john --incremental=digits hash.txt
```

::: tip Tips
Selain *raw hash*, `john` dapat digunakan untuk crack format yang lain, seperti file terenkripsi atau format kredensial yang lain (misal: zip, rar, ssh/id_rsa, pem, openssl, dsb).

Agar melakukan hal tersebut, kita harus menginstall `john` **versi jumbo**.

Link: [https://github.com/openwall/john/blob/bleeding-jumbo/doc/INSTALL](https://github.com/openwall/john/blob/bleeding-jumbo/doc/INSTALL)

More Source: [https://www.stationx.net/how-to-use-john-the-ripper/](https://www.stationx.net/how-to-use-john-the-ripper/)
:::

### 3. Hashcat

![hashcat](/miniclass/cyber/assets/penetration-testing/hashcat.png)

`hashcat` memanfaatkan kekuatan GPU dan sangat efektif untuk volume hash yang besar atau algoritma yang berat.

```bash
# Help command dan list mode yang tersedia
hashcat -h

# [-m] Common Modes:
# 0     -> MD5
# 10    -> Md5(Password+Salt)
# 20    -> Md5(Salt+Password)
# 500   -> md5crypt, MD5 (Unix), Cisco-IOS $1$ (MD5)
# 3200  -> bcrypt $2*$, Blowfish (Unix)
# 7400  -> sha256crypt $5$, SHA256 (Unix)
# 1800  -> sha512crypt $6$, SHA512 (Unix)
# 100   -> SHA1
# 110   -> SHA1(Password+Salt)
# 400   -> WordPress/phpass
 
# Contoh Crack SHA-512
hashcat -m 1800 -a 0 path/to/sha512_hashlist.txt path/to/password_wordlist.txt

# Contoh Crack MD5 (With Output Redirection)
hashcat -m 500 -a 0 md5_hashes.list passwords.txt -o cracked_passwords.txt

# Hashcat akan menyimpan hash yang pernah di-crack
# Biasanya disimpan di ~/.local/share/hashcat/hashcat.potfile
# Untuk menampilkannya kembali, gunakan flag --show
hashcat -m 0 -a 0 hash.txt rockyou.txt --show

# -a 0 = Straight Dictionary Attack
hashcat -m 500 -a 0 hash.txt dict.txt

# -a 1 = Combination Attack
hashcat -m 500 -a 1 hash.txt dict1.txt dict2.txt

# -a 3 = Bruteforce Attack (Mask)
hashcat -m 500 -a 3 hash.txt ?l?d?u

# Mask Character Sets:
# ?l   -> abcdefghijklmnopqrstuvwxyz
# ?u   -> ABCDEFGHIJKLMNOPQRSTUVWXYZ
# ?d   -> 123456789
# ?h   -> 0123456789abcdef
# ?H   -> 0123456789ABCDEF
# ?s   -> !”#$%&'()*+,-./:;<=>?@[\]^_`{|}~
# ?a   -> ?l?u?d?s (semua karakter)
# ?b   -> 0x00 – 0xff

# -a 6 = Hybrid Wordlist + Mask
hashcat -m 500 -a 6 hash.txt wordlist.txt ?d?s

# -a 7 = Mask + Wordlist
hashcat -m 500 -a 7 hash.txt ?d?s wordlist.txt
```

::: tip Tips
Hashcat Cheatsheet: [https://www.blackhillsinfosec.com/hashcat-cheatsheet/](https://www.blackhillsinfosec.com/hashcat-cheatsheet/)
:::

---

## Langkah Mitigasi

- Ubah *default credential* (jika ada)
- Hindari menyimpan password secara eksplisit. Ubah ke dalam bentuk hash
- Gunakan algoritma hash terbaru (`SHA256` atau `bcrypt`)
- Pakai *password policy* yang lebih kuat (Untuk menghindari *weak password*)
- Terapkan *rate limiting* atau *lockout* pada API Auth
- Terapkan MFA jika relevan
- Audit credential reuse jika perlu

---

## Hands-on

- Tryhackme: [Brute IT](https://tryhackme.com/room/bruteit)

# 🚀 Laravel Artisan Cheatsheet

[`php artisan`](https://laravel.com/docs/13.x/artisan) adalah Command Line Interface (CLI) bawaan Laravel yang akan menjadi sahabat terbaikmu selama development. Berikut adalah daftar perintah yang akan paling sering kamu gunakan!

## 🌟 1. Perintah Dasar (Wajib Tahu)

Perintah ini digunakan untuk operasional dasar aplikasi Laravel-mu.

| Perintah | Deskripsi |
| :--- | :--- |
| `php artisan serve` | Menjalankan development server lokal (biasanya di localhost:8000). |
| `php artisan list` | Menampilkan daftar SEMUA perintah artisan yang tersedia. |
| `php artisan help [perintah]` | Menampilkan panduan/bantuan untuk perintah tertentu (contoh: `php artisan help make:controller`). |
| `php artisan key:generate` | Membuat APP_KEY baru di file .env. (Wajib dijalankan pertama kali setelah clone project dari GitHub). |
| `php artisan storage:link` | Membuat symlink folder storage ke public. (Dibutuhkan jika kamu ingin menampilkan gambar/file yang di-upload oleh user). |

## 🏗️ 2. Generator (Make Commands)

Jangan buat file manual! Gunakan perintah `make` agar Laravel membuatkan boilerplate (struktur kode dasar) secara otomatis sesuai standar.

| Perintah | Deskripsi |
| :--- | :--- |
| `php artisan make:controller NamaController` | Membuat file controller kosong. |
| `php artisan make:controller NamaController -r` | Membuat controller lengkap dengan fungsi CRUD dasar (index, create, store, dll). |
| `php artisan make:model NamaModel` | Membuat file Model. (Tips: Nama model gunakan bahasa Inggris dan Singular/Tunggal). |
| `php artisan make:model NamaModel -m` | Membuat Model sekaligus file Migration-nya. (Sangat disarankan!). |
| `php artisan make:model NamaModel -mcr` | **Combo Maut!** Membuat Model, Migration, dan Controller Resource sekaligus. |
| `php artisan make:migration create_nama_tabel_table` | Membuat file migration baru untuk membuat tabel di database. |
| `php artisan make:seeder NamaSeeder` | Membuat file seeder untuk mengisi data awal. |

## 🗄️ 3. Database & Migrations

Perintah untuk mengatur skema database kamu lewat code.

| Perintah | Deskripsi |
| :--- | :--- |
| `php artisan migrate` | Menjalankan semua file migration yang belum dieksekusi (membuat tabel di DB). |
| `php artisan migrate:rollback` | Membatalkan/mundur dari proses migrasi yang paling terakhir dilakukan. |
| `php artisan migrate:fresh` | **HATI-HATI!** Menghapus (drop) semua tabel yang ada, lalu menjalankan ulang semua migration dari awal. |
| `php artisan db:seed` | Menjalankan file DatabaseSeeder untuk mengisi dummy data. |
| `php artisan migrate:fresh --seed` | Menghapus semua tabel, migrasi ulang, lalu langsung mengisi data dari seeder. (Sangat berguna saat fase development!). |

## 🛣️ 4. Routing

Saat rute/URL aplikasi tidak bisa diakses, periksa lewat perintah ini.

| Perintah | Deskripsi |
| :--- | :--- |
| `php artisan route:list` | Menampilkan tabel berisi semua URL/Route yang terdaftar di aplikasi beserta Method dan Controller-nya. |
| `php artisan route:clear` | Menghapus cache pada route. (Gunakan jika kamu baru saja menambahkan route baru tapi memunculkan error 404). |

## 🛠️ 5. Troubleshooting & Clear Cache ("Obat Sakit Kepala")

**Pro-Tip:** Jika kodemu dirasa sudah 100% benar tapi Laravel masih memunculkan error yang aneh, atau perubahan kodemu (seperti di .env atau view) tidak muncul di browser, jalankan perintah ini!

| Perintah | Deskripsi |
| :--- | :--- |
| `php artisan optimize:clear` | **Sapu Jagat!** Menghapus semua cache sekaligus (view, route, config, dan cache aplikasi). |
| `php artisan config:clear` | Menghapus cache konfigurasi. (Wajib setelah mengubah isi file .env). |
| `php artisan view:clear` | Menghapus cache tampilan blade. |

## 💻 6. Interaksi Database Cepat (Tinker)

Tinker memungkinkanmu berinteraksi langsung dengan aplikasi dan databasemu lewat terminal tanpa harus membuat route/controller.

| Perintah | Deskripsi |
| :--- | :--- |
| `php artisan tinker` | Membuka REPL (Read-Eval-Print Loop). Kamu bisa mengetikkan kode PHP/Laravel di sini (contoh: `User::all()`) untuk melihat data. |

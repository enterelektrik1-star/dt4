# Web File Manager - PHP Native

Aplikasi file manager berbasis PHP native, tanpa database dan tanpa framework.

## Persyaratan
- PHP 8.0+ (disarankan PHP 8.1 atau lebih baru)
- Ekstensi Fileinfo direkomendasikan
- Apache/cPanel atau server PHP lain
- Izin tulis pada folder `storage`

## Instalasi di hosting
1. Ekstrak ZIP.
2. Upload seluruh isi folder `php_native_file_manager` ke `public_html/filemanager/` (atau folder/domain yang diinginkan).
3. Pastikan folder `storage` memiliki permission 755. Jika hosting membutuhkan, sesuaikan ke 775 melalui File Manager cPanel.
4. Edit `config.php`, ganti `ADMIN_PASSWORD` dengan password yang kuat.
5. Buka `https://domain-anda.com/filemanager/`.

## Login awal
- Username: `admin`
- Password: `GantiPasswordKuat123!`

Segera ganti password di `config.php` sebelum aplikasi dipublikasikan.

## Menjalankan di localhost
Dari folder aplikasi jalankan:
`php -S localhost:8000`
Lalu buka `http://localhost:8000`.

## Catatan keamanan
- Aplikasi membatasi operasi file ke dalam folder `storage`.
- Folder utama tidak bisa dihapus; folder hanya bisa dihapus jika kosong.
- Symlink disembunyikan.
- Upload dibatasi 50 MB per file (ubah `MAX_UPLOAD_BYTES` di `config.php`).
- Gunakan HTTPS dan password administrator yang kuat.
- Jangan meletakkan aplikasi ini di area publik tanpa mengganti password default.
- Preview HTML dan SVG ditampilkan sebagai teks, bukan dieksekusi.
- Batas upload PHP di server (`upload_max_filesize` dan `post_max_size`) juga harus mencukupi.

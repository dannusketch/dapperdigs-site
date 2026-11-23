---
name: upload-katalog-web
description: Upload foto drop baru dari Google Drive "3. Curated Photos" ke repo dan isi kolom Foto di sheet katalog "Thrift". Pakai saat user bilang "UPLOAD KATALOG WEB" (atau minta upload drop/katalog baru ke web).
---

# UPLOAD KATALOG WEB

Alur untuk memasukkan drop baru ke katalog dapperdigs.store. Web (`index.html`) membaca sheet katalog lewat CSV publik, lalu memuat foto dari root repo berdasarkan nama file di kolom `Foto`.

## Sumber dan tujuan

- **Drive (konektor Google Drive):** folder `3. Curated Photos`, ID `1hbmmYNx9uimlWhpfHD87OlJgZdYfqQAt`.
  - Isinya subfolder per item bernama `N. KODE` (contoh `1. BDC`, `8. BDB`).
  - File di dalamnya bernama `KODE1`, `KODE2`, ... dengan ekstensi campur (`.jpg`, `.JPEG`, `.heic`, `.HEIC`, `.png`).
  - Foto bernomor 1 adalah cover.
- **Repo:** foto disimpan di root repo sebagai `KODEn.jpg` (ekstensi selalu `.jpg` huruf kecil). `CONFIG.FOTO_FOLDER` di `index.html` kosong, jadi jangan pakai subfolder.
- **Sheet katalog (konektor Google Sheets):** spreadsheet "Thrift", ID `11Hu8ID0_kdsZjw30DOsoMHSylnBAGTtgsxV-0YVQim4`, tab `Sheet1`.
  - Header: `Nama | Size | Kondisi | Foto | Status`.
  - Satu baris per item. `Foto` berisi daftar file dipisah koma + spasi, cover dulu: `BDC1.jpg, BDC2.jpg, ...`.

## Langkah

1. **Inventaris.** Daftar semua subfolder di `3. Curated Photos`, lalu daftar file tiap subfolder. `search_files` hanya mengembalikan ±5 hasil per halaman, jadi ikuti `nextPageToken` sampai habis. Cek nomor urut lengkap tanpa bolong; kalau ada yang bolong atau ganda, laporkan ke user sebelum lanjut.
2. **Tentukan item baru.** Item dianggap baru kalau `KODE1.jpg` belum ada di root repo **dan** kodenya belum muncul di kolom `Foto` sheet. Proses item baru saja; jangan sentuh item lama.
3. **Download.** Pakai `download_file_content` per file. Hasilnya tersimpan di folder `tool-results` sesi; salin ke `scratchpad/raw/` dengan nama `KODEn.<ext>` sesuai judul file (cocokkan ukuran byte dengan `fileSize` dari Drive).
4. **Konversi.** ImageMagick di container tidak punya decoder HEVC, jadi HEIC gagal di `convert`.
   - HEIC: `pip install pillow-heif`, buka dengan Pillow + `register_heif_opener()`, `ImageOps.exif_transpose`, simpan sementara sebagai PNG.
   - Semua file (JPG, PNG, PNG hasil HEIC):
     `convert IN -auto-orient -resize '1600x1600>' -strip -sampling-factor 4:2:0 -interlace JPEG -quality 82 OUT.jpg`
     (`-strip` menghapus EXIF termasuk GPS dari foto iPhone.)
   - Buat contact sheet (`montage`) dan lihat untuk memastikan tidak ada foto terbalik atau rusak.
5. **Repo.** Salin JPG ke root repo. Mulai branch kerja dari `origin/main` terbaru. Commit, push, lalu buat PR ke `main` (user sudah memberi izin tetap untuk PR di alur ini). Subscribe aktivitas PR. Merge dilakukan user sendiri.
6. **Sheet.** Baca `Sheet1` dulu. Tambahkan satu baris per item baru di bawah baris terakhir yang terisi, urut sesuai nomor folder. Isi hanya kolom `Foto`; biarkan `Nama`, `Size`, `Kondisi`, `Status` kosong (user mengisi sendiri). Jangan ubah atau hapus baris yang sudah ada. Baca ulang setelah menulis untuk verifikasi.
7. **Laporan.** Sebutkan item yang diproses, jumlah foto per item, link PR, dan baris sheet yang ditambah. Ingatkan bahwa foto baru tampil setelah PR di-merge.

## Catatan

- Jangan hapus foto lama atau baris sheet tanpa diminta.
- Domain `dapperdigs.store` dan `docs.google.com` (CSV publik) diblokir proxy container, jadi tampilan web tidak bisa dicek dari sesi; minta user mengecek.

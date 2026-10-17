# Pixego

## Coba Aplikasinya

[![Buka Aplikasi](https://img.shields.io/badge/Buka%20Aplikasi-Streamlit-FF4B4B?logo=streamlit&logoColor=white)](ISI_LINK_STREAMLIT_DI_SINI)

Klik tombol di atas untuk membuka Pixego langsung di browser, tanpa instalasi.

![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)

Pixego adalah aplikasi steganografi untuk menyembunyikan pesan rahasia di dalam gambar PNG. Pesan dienkripsi dengan AES-256-GCM (kunci dari Argon2id), lalu disisipkan memakai LSB sekuensial atau acak berbasis kunci. Dilengkapi pengukuran MSE dan PSNR, uji chi-square, notebook Jupyter, CLI, dan unit test.

Nama Pixego berasal dari kata *pixel* dan *stego*.

## Fitur

- Enkripsi AES-256-GCM, kunci diturunkan dari password lewat Argon2id dan HKDF-SHA256
- Dua mode penyisipan: sekuensial dan acak berbasis kunci, dengan 1 sampai 3 bit per kanal
- Deteksi password salah atau gambar yang diubah lewat tag autentikasi GCM
- Pengukuran kualitas citra: MSE dan PSNR
- Steganalisis chi-square (seluruh gambar dan progresif)
- Pemuatan cover yang aman: rotasi EXIF, transparansi, pengecilan otomatis
- Notebook Jupyter, CLI, dan 25 unit test

## Struktur Folder

```
pixego/
  README.md
  LICENSE
  requirements.txt
  .gitignore
  notebook/
    pixego.ipynb                 notebook bersih, jalankan sendiri
    pixego_dengan_output.ipynb   contoh hasil eksekusi (gambar sintetis)
  src/
    stego_core.py        inti: kriptografi, embed, extract, MSE, PSNR, chi-square
    cli.py               antarmuka baris perintah
    generate_samples.py  pembuat gambar cover contoh
  tests/
    test_stego.py        25 unit test
  sample_images/         dua gambar cover sintetis (tekstur dan polos)
  hasil_contoh/          CSV hasil eksperimen dari gambar sintetis
```

## Cara Memasang

Butuh Python 3.10 atau lebih baru.

```
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

## Cara Menjalankan

**1. Notebook (disarankan untuk belajar dan laporan)**

```
jupyter notebook notebook/pixego.ipynb
```

Jalankan sel dari atas ke bawah. Untuk memakai gambarmu sendiri, isi `USER_COVER_PATHS` di bagian 6. Notebook juga bisa diunggah ke Google Colab.

**2. Baris perintah**

```
cd src
python cli.py kapasitas ../sample_images/cover_tekstur.png
python cli.py embed -i ../sample_images/cover_tekstur.png -o stego.png -m "pesan rahasia"
python cli.py extract -i stego.png
python cli.py metrik ../sample_images/cover_tekstur.png stego.png
python cli.py chi-square stego.png
```

Contoh dengan foto HP yang besar:

```
python cli.py embed -i foto_hp.jpg -o stego.png -m "halo" --max-side 1024 --save-cover cover_kecil.png
python cli.py metrik cover_kecil.png stego.png
```

Kalau opsi `-p` tidak diisi, password ditanyakan lewat prompt (lebih aman karena tidak tersimpan di riwayat terminal). Opsi lain: `--mode sequential|random`, `--bits 1|2|3`, `--message-file`, `--max-side`, `--save-cover`, `--tanpa-verifikasi`.

Setelah menyimpan stego image, `embed` otomatis membaca ulang file hasil dan memastikan pesannya bisa diekstrak. Kalau output bukan `.png` atau `.bmp`, perintah ditolak karena format lossy merusak pesan.

**3. Unit test**

```
python -m unittest discover -s tests -v
```

## Cara Kerja Singkat

```
EMBED   : password --Argon2id--> master key --HKDF--> kunci AES + kunci posisi
          pesan --AES-256-GCM--> ciphertext --LSB (posisi acak)--> stego image (PNG)

EXTRACT : stego image --> baca header --> turunkan kunci lagi --> baca bit di posisi yang sama
          --> AES-256-GCM decrypt (tag salah = password salah atau gambar diubah)
```

Header 26 byte ditanam sekuensial (1 bit per kanal) di awal gambar. Isinya magic `STG1`, mode, bit per kanal, salt 16 byte, dan panjang body. Body berisi nonce 12 byte, ciphertext, dan tag GCM 16 byte. Header dipakai sebagai associated data GCM sehingga perubahan pada header ikut terdeteksi.

## Format Gambar

**Cover (input)** boleh PNG, JPG, BMP, WebP, dan format lain yang dikenali Pillow. Gambar dibuka lewat `load_cover` yang menangani:

| Kasus | Perlakuan |
|---|---|
| Foto HP dengan rotasi EXIF | Rotasi diterapkan agar tidak miring |
| PNG transparan | Diratakan di atas latar putih |
| GIF atau PNG palet | Diubah ke RGB, GIF animasi hanya frame pertama |
| Gambar sangat besar | Bisa diperkecil dengan `--max-side` |
| HEIC/HEIF atau file rusak | Pesan error yang jelas (konversi ke PNG atau JPG dulu) |
| Metadata (EXIF, profil warna) | Tidak ikut tersimpan di hasil |

**Stego (output)** wajib PNG atau BMP. JPG, WebP, resize, crop, dan filter merusak pesan. Kirim sebagai file atau dokumen, bukan sebagai foto lewat aplikasi chat yang mengompres otomatis.

Mode acak menolak gambar di atas sekitar 8 megapiksel (24 juta kanal) karena boros memori. Perkecil gambar dulu atau pakai mode sequential.

## Catatan Penting

1. **Parameter Argon2id** (64 MiB, 3 iterasi, paralelisme 4) perlu dicocokkan dengan RFC 9106 dan OWASP Password Storage Cheat Sheet sebelum dipakai di laporan.
2. **Folder `hasil_contoh`** berasal dari gambar sintetis. Itu hanya bukti kode berjalan. Untuk laporan, jalankan ulang eksperimen dengan gambar asli.
3. **Hasil chi-square** hanya menggambarkan satu jenis serangan statistik dan sangat bergantung pada karakter gambar. Jangan menyimpulkan keamanan umum dari hasil ini.

## Keterbatasan

- Rapuh terhadap kompresi, resize, crop, dan filter.
- Header tidak disamarkan dan parameter Argon2id tidak disimpan di header.
- Cover berformat JPG punya karakter statistik hasil kompresi, yang bisa memengaruhi hasil chi-square.
- Keamanan tetap bergantung pada kekuatan password.
- Ini kode pembelajaran dan belum diaudit untuk penggunaan produksi.

## Referensi

- Westfeld, A., & Pfitzmann, A. (2000). Attacks on Steganographic Systems. *Information Hiding (IH 1999)*, LNCS, hlm. 61-76. Springer. DOI: 10.1007/10719724_5
- Böhme, R., & Westfeld, A. (2004). Exploiting Preserved Statistics for Steganalysis. *Information Hiding (IH 2004)*, hlm. 82-96. Springer.
- RFC 9106 (2021). Argon2 Memory-Hard Function for Password Hashing and Proof-of-Work Applications. https://www.rfc-editor.org/rfc/rfc9106
- NIST SP 800-38D (AES-GCM) dan RFC 5869 (HKDF): cek langsung ke situs resminya sebelum dikutip.

## Pengembang

| | |
|---|---|
| Nama | Kayla Verda Fawnia Heses |
| NIM | A11.2024.15689 |
| Mata Kuliah | Kriptografi |

Aplikasi ini dibuat untuk memenuhi proyek mata kuliah Kriptografi.

## Lisensi

MIT. Lihat file `LICENSE`.

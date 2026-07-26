# Neraca — Pencatatan Aset, Utang & Arus Kas

Aplikasi keuangan pribadi satu file (`index.html`). Tanpa build, tanpa dependency lokal — cukup buka di browser atau host di GitHub Pages.

---

## Perubahan Proses Bisnis

Versi sebelumnya punya cacat mendasar: **transaksi tidak memengaruhi total harta**. Aset dicatat manual sebagai angka mati, sementara pemasukan/pengeluaran hanya menumpuk di daftar terpisah. Akibatnya gaji masuk tidak menaikkan harta bersih, dan bayar cicilan tidak mengurangi utang.

Sekarang semua saling terhubung:

| Entitas | Perlakuan |
|---|---|
| **Akun kas/bank/investasi** | Punya saldo awal. Saldo kini = saldo awal + seluruh mutasi transaksi. Tidak bisa diedit langsung — hanya lewat transaksi, jadi angkanya selalu bisa diaudit. |
| **Aset tetap** | Properti, kendaraan, logam mulia, piutang. Dinilai manual, bisa diperbarui kapan saja. |
| **Utang** | Punya pokok awal, bunga %/tahun, cicilan/bulan, tanggal jatuh tempo. Sisa pokok berkurang otomatis setiap pembayaran dicatat. |
| **Transaksi** | Wajib memilih akun. Empat jenis, masing-masing dengan efek akuntansi yang benar. |

### Efek tiap jenis transaksi

```
Pemasukan     → saldo akun naik           → harta bersih naik
Pengeluaran   → saldo akun turun          → harta bersih turun
Transfer      → akun A turun, akun B naik → harta bersih TIDAK berubah
Bayar Utang   → saldo akun turun sebesar nominal
                sisa pokok turun sebesar (nominal − porsi bunga)
              → harta bersih turun hanya sebesar porsi bunganya
```

Baris terakhir itu yang paling sering salah di aplikasi keuangan sederhana. Membayar pokok utang bukan kerugian — uang berpindah dari kas ke pengurangan kewajiban, kekayaan bersih tetap. Hanya bunga yang benar-benar hilang.

**Total Harta Bersih** = (saldo semua akun + nilai aset tetap) − sisa seluruh utang.

Semua angka di dasbor dihitung ulang dari riwayat transaksi, bukan disimpan sebagai total. Jadi tidak ada kemungkinan saldo "melenceng" dari mutasinya.

### Pengaman logika

- Nominal harus > 0; akun asal dan tujuan transfer tidak boleh sama.
- Porsi bunga tidak boleh melebihi nominal pembayaran.
- Pembayaran utang ditolak jika melebihi sisa pokok.
- Pokok utang tidak bisa diubah menjadi lebih kecil dari yang sudah dibayar.
- Akun yang masih dipakai transaksi tidak bisa dihapus atau dipindah kelompok.

---

## Fitur

- **Ringkasan** — tren harta bersih 12 bulan, komposisi aset, pemasukan vs pengeluaran, pengeluaran per kategori, saldo akun, utang jatuh tempo terdekat.
- **Rasio utang** dengan penilaian otomatis (sehat < 36%, berisiko > 80%).
- **Anggaran bulanan** per kategori dengan progress bar dan peringatan saat terlampaui.
- **Filter transaksi** per bulan, jenis, akun, dan pencarian teks.
- **Edit penuh** untuk semua entitas, bukan cuma hapus.
- **Ekspor/impor JSON**, plus tombol *Isi Contoh* untuk melihat data lengkap.
- **Migrasi otomatis** dari data versi lama (`bukukas_data_v1`) — aset/utang dipisah ke tempat yang benar, transaksi lama diarahkan ke akun default.

---

## Sinkronisasi Antar Device

`localStorage` **tidak bisa** sinkron antar perangkat — sifatnya menyimpan data hanya di satu browser di satu device. Untuk data yang sama di semua device diperlukan penyimpanan cloud, di sini memakai **Supabase** (gratis).

Arsitekturnya *offline-first*: `localStorage` tetap menjadi cache, jadi app tetap jalan tanpa internet dan menyusul sinkron begitu online.

### Langkah setup

1. Buat akun di [supabase.com](https://supabase.com) → **New project**.
2. Buka **SQL Editor**, jalankan:

```sql
create table if not exists neraca_state (
  space_id   text primary key,
  data       jsonb not null,
  updated_at timestamptz not null default now(),
  device     text
);

alter table neraca_state enable row level security;

create policy "akses via kode ruang" on neraca_state
  for all to anon using (true) with check (true);
```

3. Di dashboard project, buka **Settings** (ikon gerigi di sidebar kiri bawah) → **API Keys**. Salin dua hal:
   - **Project URL** — bentuknya `https://<project-ref>.supabase.co`. Ini alamat project Anda sendiri, dibuat otomatis oleh Supabase saat project dibuat; tidak ada hubungannya dengan repo GitHub.
   - **Publishable key** (`sb_publishable_…`) atau **anon key** (`eyJ…`) — keduanya diterima app ini.

   Jalan pintas: tombol **Connect** di bagian atas dashboard juga menampilkan keduanya. Jangan pakai **service_role** / **secret key** — itu untuk server, bukan browser.
4. Di app, klik pil status di kanan atas (*Lokal saja*) → tempel keduanya.
5. Isi **Kode Ruang** — dipakai sebagai identitas data Anda. Klik *Buat* untuk menghasilkan kode acak.
6. Klik **Uji Koneksi**, lalu **Aktifkan & Sinkron**.
7. Ulangi langkah 4–6 di device lain dengan **Kode Ruang yang sama**.

### Cara kerjanya

- Setiap perubahan didorong ke server 1,2 detik setelah Anda berhenti mengetik.
- App menarik perubahan tiap 15 detik, saat tab kembali aktif, dan saat koneksi pulih.
- Device baru yang datanya masih kosong otomatis mengambil data dari server.
- Jika dua device sama-sama berubah, muncul dialog pilihan (*pakai data server* / *pakai data device ini*) lengkap dengan waktu dan jumlah catatan masing-masing — tidak ada data yang ditimpa diam-diam.
- Indikator status: `Lokal saja` · `Menyinkronkan…` · `Tersinkron` · `Offline` · `Gagal sinkron`.

### Catatan keamanan

Repo ini publik, jadi perlu dipahami:

- **anon / publishable key memang dirancang untuk browser** dan tidak berbahaya bila terlihat — pengamanannya ada di Row Level Security, bukan pada kerahasiaan kunci.
- Supabase sedang memindahkan format kunci: **anon key lama (`eyJ…`) akan dihentikan akhir 2026**, penggantinya **publishable key (`sb_publishable_…`)`**. App ini menerima keduanya, tapi untuk project baru sebaiknya langsung pakai yang publishable.
- **Kode Ruang berfungsi seperti kata sandi.** Policy di atas mengizinkan siapa pun yang tahu kode ruang untuk membaca data ruang itu. Pakai kode panjang dan acak, jangan nama Anda sendiri.
- Kredensial disimpan di `localStorage` device masing-masing, **tidak** ikut ter-commit ke repo.
- Untuk keamanan yang lebih ketat, aktifkan Supabase Auth (login email) dan ubah policy menjadi `using (auth.uid() = user_id)`. Itu menghilangkan ketergantungan pada kerahasiaan kode ruang.

Kalau belum mau pakai cloud: biarkan saja tidak dikonfigurasi. App berjalan penuh secara lokal, dan pindah device bisa lewat **Ekspor** → **Impor** file JSON.

---

## Deploy

Repo ini bisa langsung di-host: **Settings → Pages → Source: main branch**. Karena hanya file statis, tidak ada langkah build.

## Struktur

```
index.html   # seluruh aplikasi (HTML + CSS + JS, ~91 KB)
README.md
```

Library eksternal hanya Chart.js dan Google Fonts, dimuat dari CDN.

# Shipping Label Generator — V1.0

Aplikasi Next.js untuk Shipping Label **4×6 inci** dengan data dari Google Sheets read-only, deploy ke **Vercel**, dan cetak via Chrome ke **Zebra ZD230**.

## Fitur

- Dropdown searchable **ATA → Shipment → Barcode → Batch**.
- Sumber data: tab `Data Base Print` — **A=ATA, B=Shipment, C=Barcode, G=Batch**.
- Total Koli dan No. Palet input manual opsional; **kotaknya tetap tercetak walaupun kosong**.
- Mengubah salah satu dari 4 dropdown mengosongkan kedua input manual.
- Cache backend **maksimal 60 detik per instance** untuk pilihan; tombol **Refresh Data** menembus cache dalam satu snapshot konsisten.
- Sebelum print backend **selalu membaca ulang Google Sheets**; data tidak valid atau koneksi gagal = print diblokir.
- SVG **Code 128** (JsBarcode), nomor human-readable, 4×6 `@page`.
- Aplikasi publik tanpa login. **Google Sheets harus tetap private**.

## Persyaratan

- Node.js 22 (minimal 20.9), npm.
- Project Google Cloud dengan **Google Sheets API diaktifkan**.
- Service Account Google Cloud.
- Spreadsheet yang sudah dibagikan ke alamat email Service Account dengan role **Viewer**.
- Tab persis `Data Base Print`, header di baris 1: A `ATA`, B `Shipment`, C `Barcode`, G `Batch`.

## 1. Menjalankan lokal

```bash
npm install
cp .env.example .env.local
npm run dev
```

Buka `http://localhost:3000`.

**Untuk preview tanpa kredensial:** isi `ENABLE_DEMO_DATA=true` dalam `.env.local`, lalu jalankan `npm run dev`. Mode demo **otomatis ditolak saat production**. Contoh data pada demo adalah data ilustrasi, bukan dari database operasional.

## 2. Menghubungkan Google Sheets

1. Google Cloud Console → buat/ pilih project → aktifkan **Google Sheets API**.
2. Buat **Service Account**, lalu buat kunci JSON melalui mekanisme yang disetujui organisasi. Simpan kunci tersebut secara aman, **jangan di-commit**.
3. Di Google Sheets klik Share, tambahkan email Service Account dengan role **Viewer**.
4. Ambil Spreadsheet ID dari URL di bagian `/d/<ID>/edit`.
5. Isi `.env.local`:

```dotenv
GOOGLE_SPREADSHEET_ID=ISI_SPREADSHEET_ID
GOOGLE_SERVICE_ACCOUNT_EMAIL=nama@project.iam.gserviceaccount.com
GOOGLE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\\nISI_KEY\\n-----END PRIVATE KEY-----\\n"
ENABLE_DEMO_DATA=false
```

`GOOGLE_PRIVATE_KEY` dapat berupa multiline asli atau menggunakan escape `\n`.

6. Jalankan ulang server. Uji beberapa kombinasi ATA/Shipment/Barcode/Batch.

**Penting:** `FORMATTED_VALUE` digunakan agar Barcode tidak kehilangan nol yang memang masih tampak pada sel sumber. Jika digit sudah hilang di Sheets, aplikasi tidak dapat merekonstruksinya. Format tanggal dan header harus diaudit.

## 3. Deploy GitHub → Vercel

1. Buat repo **private** (disarankan), upload seluruh proyek kecuali `.env.local` dan `node_modules`.
2. Import repo di Vercel → Framework: **Next.js**.
3. Isi 3 environment variables (Spreadsheet ID, Service Account Email, Private Key) di **Project Settings → Environment Variables**, berlaku untuk Production (dan Preview jika memang diperlukan).
4. Pastikan `ENABLE_DEMO_DATA` tidak diset `true` pada production.
5. Deploy lalu uji data aktual dan hasil cetak Zebra ZD230.
6. **Wajib** konfigurasi Vercel Firewall/WAF atau rate limiting terdistribusi untuk endpoint API, sesuai kemampuan plan. Pembatasan di kode `lib/rate-limit.ts` hanya **best-effort per-instance**, bukan pengganti WAF.
7. Minta **persetujuan pemilik data** sebelum mempublikasikan. Tanpa login, siapa pun yang membuka URL juga dapat membaca pilihan dropdown melalui API.

## 4. Cara cetak

- Pilih ATA → Shipment → Barcode → Batch.
- Isi Total Koli dan No. Palet jika perlu (boleh kosong).
- Klik **Cetak Shipping Label 4×6**.
- Dialog Chrome: Zebra ZD230, kertas **4×6 inch**, **Scale 100%**, **Margins None**, **Headers and footers OFF**.
- Kalibrasi media lewat driver Zebra bila hasil turun/terpotong. Uji barcode dengan scanner fisik.

## 5. Struktur kode

```text
app/
  api/options/route.ts   daftar dropdown + cache 60 detik + refresh
  api/validate/route.ts  validasi fresh sebelum print
  globals.css            desain UI dan cetak @page 4x6
  layout.tsx             metadata
  page.tsx               halaman aplikasi
components/
  LabelWorkspace.tsx     form, reset, refresh, preview, print
  SearchSelect.tsx       dropdown searchable
  Barcode.tsx            Code 128 SVG
lib/
  sheets.ts              Google Sheets API read-only
  data-utils.ts          mapping, deduplikasi, validasi
  rate-limit.ts          rate limit best effort per instance
  demo-data.ts           data contoh lokal
  types.ts
```

## 6. Catatan batasan dan keamanan

- **Tidak ada** login, write-back, riwayat print, bulk print, atau direct printer access.
- Cache **per server instance**, bukan cache terdistribusi. Validasi cetak selalu melewati cache, tetapi permintaan bersamaan dapat berbagi satu pembacaan yang sedang berjalan.
- Refresh publik memiliki pembatasan request best-effort; tanpa WAF, pengunjung dapat membebani API/Google Sheets.
- Validasi cetak membuktikan data ada saat pemeriksaan, **bukan** bahwa printer berhasil mencetak.
- Backend mengembalikan hanya kolom yang dibutuhkan, namun **data pilihan tetap bisa dibaca publik**; jangan rilis data internal tanpa izin.
- Teks bebas yang terlalu panjang dapat diblokir saat preflight print agar label tidak terpotong; tetap uji media 4×6 fisik.
- Tidak ada credential di frontend. Jangan masukkan Service Account JSON ke folder `public/`.
- Build sukses bukan berarti lulus QA integrasi: untuk produksi perlu audit data aktual dan scan test Zebra ZD230.

## 7. Pemeriksaan

```bash
npm run typecheck
npm run build
npm run test
```

## 8. Checklist Go/No-Go

- [ ] Tab dan header kolom A/B/C/G terverifikasi.
- [ ] Formatted Barcode dan ATA sesuai sumber.
- [ ] Duplikasi dan multi-batch tidak merusak filter.
- [ ] Cache, refresh, dan validasi fresh dites.
- [ ] Permission Service Account hanya Viewer.
- [ ] Akses publik terhadap data disetujui pemilik data.
- [ ] WAF/rate limit terdistribusi diterapkan.
- [ ] Zebra ZD230 mencetak 1 lembar 4×6 tanpa crop.
- [ ] Barcode hasil cetak terbukti terbaca scanner.


## Patch V1.0.1 — Perbaikan tombol cetak

- Pemeriksaan fit label kini melihat batas konten terakhir, bukan `scrollHeight` keseluruhan paper yang dapat keliru karena SVG dan skala preview.
- Ukuran font Shipment, Batch, Barcode dan isian manual mengecil bila konten benar-benar membutuhkan ruang tambahan; teks tetap ditolak jika terlalu panjang.
- Error percobaan cetak tidak lagi mengunci tombol Print secara permanen; operator dapat mencoba ulang.
- Input dinonaktifkan sementara saat validasi server berjalan.

**Penting untuk pengguna tanpa GitHub/VS Code:** Vercel Drop yang digunakan untuk unggah folder biasanya membuat proyek baru. Untuk memperbarui *proyek dan URL Vercel yang sama* gunakan deploy ke project ID yang sama melalui Vercel CLI/API atau minta bantuan integrasi yang mempunyai izin deployment. Jangan menaruh private key pada ZIP ini.

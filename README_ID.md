# PRILLY XITER — Website + QRIS otomatis

Paket ini mempertahankan katalog dan tampilan storefront PRILLY XITER yang sudah ada, lalu mengganti QRIS statis dan konfirmasi screenshot dengan QRIS dinamis Xendit, database pesanan Cloudflare D1, dan webhook pembayaran.

## Isi paket

- `public/index.html` — website storefront, katalog, logo/ilustrasi inline, overlay pembuka, checkout QRIS dinamis, polling status pembayaran.
- `src/index.js` — Cloudflare Worker API. Secret Xendit hanya dibaca di backend.
- `schema.sql` — tabel pesanan D1.
- `wrangler.toml` — konfigurasi Worker + static assets + D1.
- `catalog.json` — salinan katalog untuk pemeriksaan; harga yang dipercaya backend ada di `src/index.js`.

## Sebelum mulai

1. Pastikan akun Xendit sudah aktif untuk menerima QRIS dan API key memiliki izin Money-in yang diperlukan.
2. Gunakan **Test Mode** terlebih dahulu. Jangan langsung memakai Live Mode sebelum tes pembayaran berhasil.
3. Siapkan komputer dengan Node.js dan koneksi internet untuk menjalankan Wrangler. File HTML saja atau GitHub Pages saja tidak dapat menyimpan secret API key dan menerima webhook dengan aman.

## Deploy Cloudflare

1. Buat/login akun Cloudflare: https://dash.cloudflare.com/
2. Install Wrangler di komputer:
   ```bash
   npm install -g wrangler
   wrangler login
   ```
3. Masuk ke folder proyek `PRILLY_XITER_AUTO`.
4. Buat database D1:
   ```bash
   wrangler d1 create prilly-xiter-orders
   ```
5. Salin `database_id` dari hasil perintah tersebut, lalu ganti `REPLACE_WITH_YOUR_D1_DATABASE_ID` di `wrangler.toml`.
6. Buat tabel:
   ```bash
   wrangler d1 execute prilly-xiter-orders --remote --file=./schema.sql
   ```
7. Simpan secret Xendit dan token admin. Masukkan nilai hanya pada prompt terminal Cloudflare, jangan ke HTML atau chat:
   ```bash
   wrangler secret put XENDIT_SECRET_KEY
   wrangler secret put XENDIT_WEBHOOK_TOKEN
   wrangler secret put ADMIN_TOKEN
   ```
   - `XENDIT_SECRET_KEY`: Secret API Key Xendit dari mode Test saat pengujian.
   - `XENDIT_WEBHOOK_TOKEN`: token verifikasi webhook yang ditampilkan/diatur pada Xendit Dashboard untuk webhook Payment Request.
   - `ADMIN_TOKEN`: buat sendiri password/token acak yang panjang. Simpan baik-baik.
8. Deploy:
   ```bash
   wrangler deploy
   ```
9. Cloudflare akan memberi URL Worker `https://prilly-xiter-store.<akun>.workers.dev`. Buka URL itu untuk menguji website.

## Atur webhook Xendit

Di Xendit Dashboard, pada **Test Mode** buka pengaturan webhook untuk Payment Request / Payments API dan daftarkan URL:

`https://NAMA-WORKER-ANDA.workers.dev/api/xendit/webhook`

Gunakan token verifikasi yang sama dengan nilai secret `XENDIT_WEBHOOK_TOKEN`. Pastikan event Payment Capture (`payment.capture`) dikirim ke endpoint tersebut. Webhook harus memakai HTTPS dan endpoint akan memvalidasi header `x-callback-token`.

Setelah Test Mode berhasil, buat Secret API Key Live dan token webhook Live, lalu ganti kedua secret Worker dengan nilai Live sebelum menerima pembayaran sungguhan.

## Cara kerja checkout

1. Harga dihitung ulang oleh Worker dari katalog server, bukan mempercayai nominal dari browser.
2. Worker membuat order `PENDING` di D1.
3. Worker meminta Payment Request Xendit dengan channel `QRIS` dan nominal IDR tepat.
4. QR string dikembalikan ke browser dan digambar menjadi QR yang dapat dipindai.
5. Xendit mengirim webhook. Worker memeriksa token, event, status, mata uang, channel, referensi, dan nominal sebelum mengubah order menjadi `PAID`.
6. Halaman mengecek status order secara berkala. Status `PAID` hanya berasal dari webhook terverifikasi.

## Endpoint

- `POST /api/orders` — buat pesanan + QRIS.
- `GET /api/orders/:id` — lihat status order milik ID tersebut.
- `POST /api/xendit/webhook` — webhook Xendit.
- `GET /api/admin/orders` — daftar maksimal 100 order, perlu header `Authorization: Bearer ADMIN_TOKEN`.

Contoh melihat daftar order dari terminal:

```bash
curl -H "Authorization: Bearer TOKEN_ADMIN_ANDA" \
  https://NAMA-WORKER-ANDA.workers.dev/api/admin/orders
```

## Catatan penting

- Sistem otomatis mengonfirmasi pembayaran, **bukan otomatis mengirim lisensi/key**. Pengiriman key masih perlu dilakukan admin atau integrasi sistem inventori digital yang sah.
- Notifikasi WhatsApp otomatis memerlukan WhatsApp Business Platform/Cloud API dan konfigurasi token + template pesan yang disetujui Meta. Paket awal ini menyediakan daftar pesanan admin yang terlindungi, bukan mengirim WhatsApp otomatis.
- Xendit memproses pembayaran sesuai status aktivasi merchant, channel QRIS, limit, biaya, dan ketentuan akun. QRIS bisa gagal dibuat jika channel belum aktif atau API key tidak memiliki izin.
- Jangan masukkan secret API key ke `public/index.html`, GitHub repository publik, atau pesan chat.
- Harga dan daftar produk di backend mengikuti katalog HTML yang tersedia saat paket ini dibuat. Perubahan harga harus dilakukan pada `src/index.js` juga.
- Pastikan produk digital yang dijual mematuhi hukum, ketentuan Xendit, dan aturan pemilik layanan/perangkat lunak terkait.

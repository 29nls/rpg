# KEBIJAKAN PRIVASI (PRIVACY POLICY)

**DRAF TEMPLATE**

> **DISCLAIMER:** Dokumen ini adalah draf template untuk panduan desain/compliance, BUKAN nasihat hukum final. Wajib ditinjau counsel sebelum launch.

---

## 1. Disclaimer & Tanggal Efektif

Dokumen ini ("Kebijakan Privasi") adalah draf template yang disusun untuk keperluan panduan desain produk dan pemenuhan compliance (GDPR Uni Eropa dan kerangka PDPA gaya Asia). **Ini bukan nasihat hukum final.** Sebelum peluncuran (launch), dokumen wajib ditinjau dan divalidasi oleh counsel/penasihat hukum yang berwenang di yurisdiksi target.

- **Tanggal efektif (rencana):** [TANGGAL_EFEKTIF]
- **Versi:** 0.1 (draft template)
- **Status:** Menunggu tinjauan counsel (target penyelesaian D-20 sebelum launch).

Kebijakan ini melengkapi Syarat & Ketentuan (Terms of Service) untuk game RPG *Fantasy Turn-Based Idle* berbasis website + PWA ("Game", "Layanan", "kami", "kita"). Dengan menggunakan Layanan, pengguna ("Anda", "Pemain") menyetujui pengumpulan dan pemrosesan data sesuai kebijakan ini sejauh diizinkan oleh hukum yang berlaku.

Jika terdapat konflik antara versi bahasa ini dan versi lain, versi yang berlaku adalah yang ditentukan counsel pada saat launch.

---

## 2. Pengontrol Data & Kontak DPO

Pengontrol data pribadi ("Data Controller") adalah:

- **Nama entitas / perusahaan:** [NAMA_PERUSAHAAN]
- **Alamat legal:** [ALAMAT_LEGAL]
- **Jenis badan hukum:** [PT / LLC / Ltd. / lainnya]
- **Nomor pendaftaran bisnis:** [REG_NO]
- **Negara domisili:** [NEGARA_DOMISILI]

**Petugas Pelindungan Data (Data Protection Officer / DPO):**

- **Nama DPO:** [NAMA_DPO]
- **Email DPO:** [EMAIL_DPO]
- **Alamat korespondensi privasi:** [ALAMAT_PRIVASI]

Untuk pertanyaan, permintaan hak pengguna (akses, koreksi, hapus, portabilitas), atau keluhan privasi, hubungi DPO melalui email di atas. Kami akan merespons sesuai tenggat waktu yang diwajibkan hukum (umumnya dalam 30 hari untuk GDPR; lihat Bagian 9). Jika DPO belum ditunjuk saat draft ini disusun, gunakan [EMAIL_SUPPORT] sebagai jalur sementara.

---

## 3. Data yang Kami Kumpulkan

Kami mengumpulkan kategori data berikut semata-mata untuk menjalankan Layanan:

### 3.1. Data Akun (Account Data)
- Alamat email (digunakan untuk login email/password dan verifikasi).
- Hash kata sandi (disimpan terenkripsi; kami tidak pernah menyimpan kata sandi plaintext).
- Profil pemain: nama panggilan (display name), avatar (jika diunggah), zona waktu.

### 3.2. Data Gameplay (Gameplay Data)
- Progresi: level, experience point (XP), pencapaian (achievements).
- Currencies virtual: Gold, Gems, Event Token (saldo & riwayat transaksi).
- Heroes/items yang dimiliki, guild keanggotaan.
- Riwayat pertempuran (battle history) dan riwayat gacha pulls.
- Riwayat transaksi marketplace dalam-game (marketplace tx).

### 3.3. Data Pembayaran — Metadata Saja (Payment Metadata)
- Kami **tidak** menyimpan nomor kartu, CVV, atau detail rekening lengkap.
- Data yang kami simpan: ID transaksi dari [Payment Provider], jenis paket Gem (IAP), jumlah, status, tanggal, dan mata uang.
- Data pembayaran riil diproses oleh pihak ketiga ([Payment Provider] / toko aplikasi platform).
- Bukti transaksi disimpan semata-mata untuk rekonsiliasi & kepatuhan pajak/keuangan.

### 3.4. Data Perangkat & Log Teknis (Device & IP)
- Alamat IP (untuk keamanan, geolokasi regional kasar, dan pencegahan penipuan).
- Jenis perangkat, sistem operasi, versi browser/PWA, identifier perangkat non-persisten.
- Log akses (waktu login, session, alamat IP, user-agent) untuk audit & keamanan.

### 3.5. Data Analytics (Opsional, Opt-In)
- Event interaksi gameplay (hanya jika Anda mengaktifkan analytics).
- Metrik performa & crash (agregat, tidak mengidentifikasi individu secara langsung).
- Tidak diaktifkan tanpa consent eksplisit; dapat ditarik kapan saja.

### 3.6. Penyimpanan Lokal (Client Cache)
- Client menyimpan salinan lokal state via `localStorage` / `IndexedDB` **hanya sebagai cache**.
- **Server adalah otoritas tunggal (server-authoritative):** state resmi disimpan di server; cache lokal bukan sumber kebenaran dan dapat dihapus kapan saja.
- Cache tidak dikirim ke pihak ketiga kecuali untuk pemrosesan Layanan yang sah.

### 3.7. Data yang TIDAK Kami Kumpulkan
- Kami tidak meminta dokumen identitas (KTP/paspor) kecuali diwajibkan hukum tertentu (mis. verifikasi usia/orang tua).
- Kami tidak mengumpulkan data sensitif (kesehatan, agama, politik) melalui Game.

---

## 4. Tujuan & Dasar Hukum Pemrosesan

| Tujuan | Data | Dasar Hukum (GDPR Art. 6) |
|---|---|---|
| Pembuatan & autentikasi akun, keamanan login | Akun, Device/IP | Pelaksanaan kontrak (Art. 6(1)(b)) |
| Penyediaan fitur gameplay & progresi | Gameplay | Pelaksanaan kontrak (Art. 6(1)(b)) |
| Pemrosesan pembayaran IAP | Payment Metadata | Pelaksanaan kontrak (Art. 6(1)(b)) |
| Kepatuhan legal & pencegahan penipuan | Log, Device/IP | Kepentingan sah / kewajiban legal (Art. 6(1)(f)/(c)) |
| Analytics & peningkatan produk | Analytics | **Consent** (Art. 6(1)(a)) — opt-in |
| Pengiriman email (verify, reset pwd) | Akun | Consent / kontrak (Art. 6(1)(a)/(b)) |

Untuk region **PDPA-style (Asia)** sebagai default, dasar pemrosesan mengikuti persetujuan (consent) dan keperluan kontraktual sesuai hukum lokal. Untuk region **EU (GDPR)**, toggle aktivasi disiapkan pada **D-20** (toggle region EU diaktifkan saat rilis).

**Consent wajib saat register:** Anda diminta menyetujui Kebijakan Privasi & Syarat sebelum akun dibuat. **Email verification bersifat soft** (D-05): akun dapat dibuat namun fitur tertentu dibatasi hingga email diverifikasi.

**Minimisasi data:** Kami hanya memproses data yang diperlukan untuk tujuan di atas dan tidak memproses lebih lanjut secara tidak sesuai.

---

## 5. Cookie & Pelacakan

Kami menggunakan mekanisme berikut:

- **Refresh token:** disimpan dalam cookie `httpOnly`, flag `Secure`, dan `SameSite` (Lax/Strict). Cookie ini tidak dapat diakses oleh JavaScript dan melindungi sesi Anda.
- **Session token:** dikelola secara aman di sisi client untuk autentikasi request.
- **Analytics:** **opt-in**. Tidak ada cookie analytics/publikasi pihak ketiga yang diaktifkan tanpa persetujuan eksplisit Anda.
- **PWA Service Worker:** hanya meng-cache aset statis (HTML, JS, CSS, aset gambar) **bukan data pribadi**. Cache lokal tidak digunakan untuk pelacakan.

Tabel ringkas cookie:

| Nama | Tipe | Tujuan | Pihak | Persetujuan |
|---|---|---|---|---|
| refresh_token | httpOnly/Secure/SameSite | Menjaga sesi login | Kami | Wajib (kontrak) |
| analytics_id | Opt-in | Metrik anonim | [Analytics Provider] | Consent |

Anda dapat mengelola preferensi cookie/analytics melalui pengaturan dalam-game ("Settings → Privacy") dan dapat menarik consent kapan saja (lihat Bagian 9). Menonaktifkan cookie esensial dapat memengaruhi fungsi login.

---

## 6. Berbagi Data dengan Pihak Ketiga

Kami **tidak menjual** data pribadi Anda. Kami membagikan data hanya dengan kategori pihak ketiga berikut, semata-mata untuk menjalankan Layanan:

| Kategori Pihak Ketiga | Tujuan | Data Dibagikan |
|---|---|---|
| [Payment Provider] | Pemrosesan IAP Gem pack | Payment metadata (bukan detail kartu) |
| [Email Service] | Verifikasi email, reset password | Alamat email, token sekali pakai |
| [Analytics Provider] | Metrik opt-in (jika diaktifkan) | Event interaksi agregat |
| [CDN Provider] | Distribusi aset & performa | IP (untuk routing), aset statis |

Setiap pihak ketiga diwajibkan melalui perjanjian pemrosesan data (DPA) untuk memproses data sesuai instruksi kami dan standar keamanan yang berlaku. Kami melakukan due diligence terhadap sub-processor sebelum penggunaannya.

---

## 7. Transfer Lintas Batas (Cross-Border Transfer)

- Region default mengikuti kerangka **PDPA-style (Asia)**.
- Saat toggle **EU (GDPR)** diaktifkan (D-20), transfer data warga EU ke luar EEA dilakukan dengan **Klausul Kontrak Standar (Standard Contractual Clauses / SCC)** atau mekanisme pengalihan yang sah lainnya.
- Untuk transfer antara Asia ↔ EU, kami mengandalkan SCC dan/atau pengakuan kecukupan (adequacy) sebagaimana diizinkan hukum.
- Data gameplay & payment metadata dapat diproses di [REGION_DATACENTER] oleh [Payment Provider] dan sub-processor mereka.
- Kami akan memberi tahu Anda jika terjadi perubahan material pada lokasi pemrosesan yang dapat memengaruhi perlindungan data Anda.

---

## 8. Retensi Data

- **Akun aktif:** data profil & gameplay disimpan selama akun aktif dan Layanan berjalan.
- **Penghapusan atas permintaan:** akun dihapus sepenuhnya (anonymize PII + purge inventory) sesuai permintaan (lihat Bagian 9).
- **Log audit:** disimpan selama **12 bulan** untuk keamanan & kepatuhan, lalu dihapus/anonimisasi.
- **Payment metadata:** disimpan selama periode yang diwajibkan hukum perpajakan/keuangan ([RETENTION_PAYMENT] tahun) oleh [Payment Provider].
- **Analytics:** dihapus setelah [RETENTION_ANALYTICS] bulan sejak pengumpulan, atau saat consent ditarik.
- **Cache lokal:** dapat dihapus oleh Anda kapan saja melalui pengaturan browser/PWA; ini tidak menghapus data server.

Kami menerapkan penghapusan otomatis (automated purge) sesuai jadwal retensi di atas bila memungkinkan.

---

## 9. Hak Pengguna

Bergantung pada yurisdiksi Anda (GDPR / PDPA-style), Anda memiliki hak berikut. Cara menggunakannya:

**Cara mengajukan:** (a) melalui menu dalam-game **Settings → Privacy → Data Request**, atau (b) email ke [EMAIL_DPO].

- **Akses (Access):** minta salinan data pribadi Anda.
- **Koreksi (Rectification):** perbaiki data yang tidak akurat (mis. display name).
- **Penghapusan (Erasure / Right to be forgotten):** hapus akun → kami **anonymize PII** dan **purge inventory** (heroes, items, currencies, guild, marketplace tx) secara permanen.
- **Portabilitas / Export:** ekspor data Anda dalam format **JSON** (profil, progresi, currencies, owned heroes/items, guild, battle history, gacha pulls, marketplace tx).
- **Penarikan consent (Withdraw consent):** untuk analytics & email marketing, tarik consent kapan saja tanpa memengaruhi keabsahan pemrosesan sebelumnya.
- **Opt-out analytics:** nonaktifkan analytics via Settings → Privacy (US-CMP di PRD).
- **Pembatasan / keberatan:** Anda dapat keberatan atas pemrosesan berdasar kepentingan sah (GDPR).

Kami akan merespons dalam **30 hari** (GDPR) atau tenggat waktu lokal (PDPA). Identitas Anda akan diverifikasi sebelum pemenuhan permintaan untuk melindungi data. Permintaan yang berlebihan/menetap dapat dikenai biaya sesuai hukum.

---

## 10. Perlindungan Anak

- Game ini **rated 12+** (untuk pemain usia **di atas 12 tahun**).
- Kami **tidak** secara sengaja mengumpulkan data dari anak di bawah batas usia yang diwajibkan hukum (mis. <13/<16 tergantung yurisdiksi) tanpa persetujuan orang tua.
- Pembatasan **IAP** berlaku untuk pemain muda (lihat ToS Bagian 7). Orang tua dapat mengaktifkan **spending limit** dan meminta penghapusan data anak.
- Jika kami mengetahui pengumpulan data dari anak tanpa consent sah, kami akan menghapusnya.
- Fitur verifikasi orang tua ([PARENTAL_VERIFY]) dapat diaktifkan bila diwajibkan platform/hukum.

---

## 11. Keamanan

- **Server-authoritative:** state resmi disimpan & divalidasi di server; client hanya cache.
- **Enkripsi:** kata sandi di-hash (algoritma [HASH_ALGO]); transmisi via TLS/HTTPS; data sensitif dienkripsi saat istirahat (at rest).
- **Token:** refresh token `httpOnly`, `Secure`, `SameSite`; rotasi token berkala.
- **Audit:** log akses & perubahan disimpan 12 bulan; pemantauan anomali/penipuan.
- **Akses internal:** berbasis kebutuhan (need-to-knom), dengan otentikasi ganda untuk admin.
- **Insiden:** prosedur respons insiden & notifikasi (sesuai kewajiban legal, mis. 72 jam untuk GDPR) telah disiapkan.
- **Pengujian:** kami melakukan pengujian keamanan berkala (termasuk penetration test pra-launch).

---

## 12. Penundaan / Penghentian Layanan

Jika Layanan dihentikan secara permanen, kami akan memberi tahu Pengguna minimal **[SHUTDOWN_NOTICE_DAYS] hari** sebelumnya dan menyediakan jendela ekspor data (JSON) sebelum penghapusan data sesuai retensi (Bagian 8).

---

## 13. Perubahan Kebijakan

Kami dapat memperbarui Kebijakan ini. Perubahan material akan diumumkan melalui email dan/atau pengumuman dalam-game minimal **[MIN_NOTICE_DAYS] hari** sebelum berlaku. Versi terbaru akan ditandai tanggal efektifnya. Penggunaan berkelanjutan setelah perubahan berlaku dianggap sebagai persetujuan. Untuk perubahan yang secara material memperluas pemrosesan, kami akan meminta consent kembali bila diwajibkan hukum.

---

## 14. Kontak & Otoritas Pengawas

- **DPO:** [EMAIL_DPO] — [ALAMAT_PRIVASI]
- **Support umum:** [EMAIL_SUPPORT]
- **Otoritas pengawas (EU/GDPR):** otoritas perlindungan data lokal Anda di negara EEA (mis. [SUPERVISORY_AUTHORITY]).
- **Otoritas pengawas (Asia/PDPA):** [PDPA_AUTHORITY] sesuai yurisdiksi.

---

## 15. Keputusan Otomatis & Profiling

Kami menggunakan pemrosesan otomatis (mis. deteksi anomali/anti-cheat berbasis server-authoritative dan penyesuaian balance currency). Pemrosesan ini **tidak** menghasilkan keputusan otomatis yang memiliki efek hukum atau berdampak signifikan bagi Anda tanpa tinjauan manusia. Jika di masa depan kami menerapkan keputusan otomatis sepenuhnya (mis. penolakan login otomatis), Anda berhak meminta intervensi manusia, menyatakan pendapat, dan membela diri (sesuai GDPR Art. 22 bila berlaku). Profiling untuk analytics hanya berjalan dengan consent opt-in dan bersifat agregat/anonim.

## 16. Notifikasi Pelanggaran Data (Data Breach)

Jika terjadi pelanggaran keamanan yang menyangkut data pribadi Anda, kami akan:
- Mengambil tindakan mitigasi segera (rotasi token, penangguhan akses mencurigakan).
- Memberitahu otoritas pengawas dalam tenggat yang diwajibkan hukum (mis. 72 jam untuk GDPR) bila risiko tinggi.
- Memberitahu Anda tanpa penundaan yang tidak wajar bila pelanggaran berisiko tinggi terhadap hak & kebebasan Anda.
- Mencatat insiden dalam log audit (retensi 12 bulan, lihat Bagian 8).

## 17. Hak Spesifik Yurisdiksi Lain

Selain hak di Bagian 9, pengguna di yurisdiksi tertentu mungkin memiliki hak tambahan:
- **California (CCPA/CPRA gaya):** hak mengetahui, menghapus, dan menolak penjualan/berbagi data (kami tidak menjual data; opt-out analytics tersedia).
- **Asia (PDPA-style):** hak akses, koreksi, penarikan consent, dan pengaduan ke [PDPA_AUTHORITY] sesuai hukum lokal.
- **EU (GDPR):** seluruh hak Bab III, termasuk portabilitas dan penarikan consent.

Pastikan counsel melengkapi bagian ini untuk setiap yurisdiksi peluncuran aktual.

---

*— Akhir Kebijakan Privasi (draft template). Wajib review counsel sebelum launch. —*

### Catatan Konsistensi (Internal / CDP + DECISION_REGISTER)
- CDP region default: **PDPA-style (Asia)**; toggle **EU** diaktifkan **D-20**.
- DECISION_REGISTER: **D-05** email-verify soft, **D-06** refund ≤48j & spending limit opt-in, **D-07** Battle Pass pasca-launch.
- Consent wajib saat register. Analytics opt-in (US-CMP di PRD).
- Server-authoritative; cache client `localStorage`/`IndexedDB` hanya salinan.

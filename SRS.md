# SRS — RPG Fantasy Turn-Based Idle (Web Desktop-First + PWA)

**Dokumen:** Software Requirements Specification (SRS)
**Versi:** 1.0 (MVP)
**Tanggal:** 2026-07-14
**Status:** Draft untuk review
**Penulis:** Arsitek Sistem & Game System Designer
**Bahasa:** Bahasa Indonesia

> **Catatan Konsistensi:** Seluruh parameter angka, rarity, rate gacha, tick idle, elemen, status effect, dan skala awal dalam dokumen ini mengikuti *Canonical Design Parameters* (CDP) yang disepakati lintas dokumen. Setiap penyimpangan akan dicatat eksplisit di bagian Asumsi.

---

## Daftar Isi

1. [Scope & System Context](#1-scope--system-context)
2. [Functional Requirements (FR)](#2-functional-requirements-fr)
3. [Non-Functional Requirements (NFR)](#3-non-functional-requirements-nfr)
4. [Arsitektur yang Direkomendasikan](#4-arsitektur-yang-direkomendasikan)
5. [API Spec Tingkat Tinggi](#5-api-spec-tingkat-tinggi)
6. [Aturan Bisnis Kritis](#6-aturan-bisnis-kritis)
7. [Test Plan](#7-test-plan)
8. [Appendix: Glosarium & Traceability](#8-appendix)

---

## 1. Scope & System Context

### 1.1 Tujuan Produk

Membangun game RPG *Fantasy Turn-Based Idle* berbasis web (desktop-first) yang dapat diinstal sebagai PWA. Permainan berjalan secara server-authoritative: seluruh status ekonomi, combat, dan gacha dihitung di server; klien hanya sebagai presentasi dan pengumpul input. Produk menargetkan MVP dengan 10–50 DAU dan dirancang agar *scalable* ke skala lebih besar tanpa perubahan arsitektur mendasar.

### 1.2 Aktor Sistem

| ID Aktor | Nama | Deskripsi | Kepentingan Utama |
|---|---|---|---|
| ACT-01 | Player | Pengguna terdaftar yang memainkan game, membeli item, ber-PvP, berdagang. | Progresi, keseruan, keadilan ekonomi. |
| ACT-02 | Admin / GM | Staf operasional yang mengelola konten, event, ban, grant item. | Stabilitas, integritas, liveops. |
| ACT-03 | Payment Provider | Pihak ketiga (gateway IAP/PSP) yang memproses pembayaran. | Konfirmasi pembayaran, webhook. |
| ACT-04 | Email Service | Layanan pengirim email (transaksional/verifikasi). | Delivery reset password, verifikasi. |
| ACT-05 | Analytics | Layanan/warehouse telemetri (opsional pihak ke-3 atau self-host). | Metrik retensi, funnel, fraud signal. |

### 1.3 Context Diagram (Teks)

```
                       +---------------------------+
                       |      EXTERNAL WORLD       |
                       +---------------------------+
                                  |
        +-----------+         +----+----+          +-----------+
        | Email Svc |<------->|         |<-------->| Payment   |
        | (ACT-04)  |  SMTP  |  RPG    |  Webhook | Provider  |
        +-----------+         |  SYSTEM |          | (ACT-03)  |
                              | (Server)|          +-----------+
        +-----------+  API    |         |    API    +-----------+
        | Player    |<------->|         |<-------->| Analytics |
        | (ACT-01)  |  HTTPS |         |  Events  | (ACT-05)  |
        +-----------+  (PWA)  +----+----+          +-----------+
                                   |
                              Admin UI/API
                                   |
        +-----------+         +----+----+
        | Admin/GM  |<------->|         |
        | (ACT-02)  |  HTTPS |         |
        +-----------+         +---------+
```

**Penjelasan boundary:**
- Segala logika kebenaran (combat resolve, gacha roll, currency ledger, idle accrual) berada **di dalam** batas sistem (server).
- Klien (PWA) berada di luar batas kebenaran; hanya mengirim niat aksi dan menerima snapshot state.
- Payment Provider & Email Service berkomunikasi via webhook/SMTP satu arah + verifikasi signature.

### 1.4 Batasan Sistem (Constraints)

| Kode | Batasan | Dampak |
|---|---|---|
| CON-01 | Desktop-first; PWA untuk install + offline queue, bukan full offline play. | Tidak ada komputasi combat di klien saat offline. |
| CON-02 | Server-authoritative wajib. | Klien tidak boleh menjadi sumber kebenaran (lihat §6.2). |
| CON-03 | MVP pool rarity = C/R/E/L; Mythic (M) hanya banner khusus (bukan MVP). | Gacha MVP tidak menghasilkan M. |
| CON-04 | Auth awal hanya email/password. | Tidak ada OAuth/SSO di MVP. |
| CON-05 | Skala awal 10–50 DAU; desain scalable. | Cukup single-node dengan Redis/Postgres; auto-scale belakangan. |
| CON-06 | Idle tick real-time 5 detik; offline cap 24 jam. | Akrual dihitung server-side. |
| CON-07 | Compliance GDPR/PDPA-style, bukan sertifikasi formal di MVP. | Fitur export/delete wajib; legal copy disiapkan belakangan. |

### 1.5 Asumsi (Eksplisit)

| Kode | Asumsi | Jika salah, dampak |
|---|---|---|
| ASP-01 | Satu akun = satu player; tidak ada multi-akun anti-abuse canggih di MVP (hanya deteksi dasar). | Risiko idle-farming multi-akun; mitigasi: rate-limit + device fingerprint ringan. |
| ASP-02 | Payment Provider mengirim webhook dengan signature HMAC terverifikasi. | Jika tidak, perlu polling status. |
| ASP-03 | Email Service memiliki rate limit wajar; kita implementasi backoff. | Jika gagal, antrian retry internal. |
| ASP-04 | Koneksi internet tersedia saat aksi state-changing (gacha, trade, combat). Offline hanya untuk akrual klaim. | Klaim offline tidak bisa saat benar-benar offline tanpa queue; IndexedDB queue kirim saat online. |
| ASP-05 | Waktu server menggunakan NTP tersinkron; `now` server adalah sumber waktu tunggal. | Jika drift besar, off-line accrual meleset. |
| ASP-06 | Penyimpanan aset statis via CDN. | Jika tidak, fallback ke origin. |
| ASP-07 | Player tidak dapat melihat kode sumber sensitive (server); klien minified. | Obfuscation terbatas; keamanan bergantung server-authoritative, bukan kerahasiaan klien. |
| ASP-08 | Bahasa utama UI Bahasa Indonesia; dokumentasi internal Bahasa Indonesia. | Lokalitas bukan fitur MVP wajib. |
| ASP-09 | Audit RNG gacha cukup transparan via history log + seeded RNG tersimpan; bukan third-party provable fairness di MVP. | Bila regulasi ketat, upgrade ke verifiable RNG. |
| ASP-10 | Marketplace awal hanya antar-player dengan escrow server; tidak ada listing NPC di MVP. | Bila perlu NPC shop terpisah di Shop module. |

### 1.6 Di Luar Cakupan (Out of Scope — MVP)

- Native mobile app (iOS/Android store). PWA diizinkan "Add to Home Screen".
- Voice chat, video chat.
- Real-time multiplayer synchronous combat (raid bersifat co-op instance async/callback, bukan real-time).
- Blockchain / NFT.
- Editor level visual untuk player.
- Localization penuh (>1 bahasa).
- Cross-platform cloud save share antar OS eksternal (cukup web account).

---

## 2. Functional Requirements (FR)

**Konvensi prioritas:**
- **P0** = Must-have MVP (blocker rilis).
- **P1** = Sebaiknya ada MVP (high value).
- **P2** = Post-MVP / nice-to-have.

**Format ID:** `FR-<modul>-<nomor>` di mana modul: AUTH, PROF, HERO, GAC, CMB, PVE, PVP, IDL, INV, MKT, GLD, SHOP, LIVE, ADM.

---

### 2.1 Auth & Account (AUTH)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-AUTH-01 | Sistem MUST mendukung registrasi akun dengan email + password (bcrypt/argon2, cost disetel). | P0 |
| FR-AUTH-02 | Password MUST di-hash di server; tidak pernah dikirim/tersimpan plaintext. | P0 |
| FR-AUTH-03 | Login MUST mengembalikan access token (JWT pendek) + refresh token (httpOnly cookie). | P0 |
| FR-AUTH-04 | Sistem MUST membatasi percobaan login gagal (brute-force protection): lock 5 gagal / 15 menit per email+IP. | P0 |
| FR-AUTH-05 | Sistem MUST mendukung reset password via email token (berlaku < 30 menit, single-use). | P0 |
| FR-AUTH-06 | Sistem SHOULD mengirim verifikasi email opsional pada registrasi (toggle fitur). | P1 |
| FR-AUTH-07 | Session MUST dapat di-revoke (logout, logout-all). | P0 |
| FR-AUTH-08 | Refresh token MUST dapat dirotasi dan masuk denylist saat revoke. | P0 |
| FR-AUTH-09 | Sistem MUST mencegah reuse refresh token (token version / jti tracking). | P0 |
| FR-AUTH-10 | Account MUST dapat di-suspend/ban oleh Admin (lihat FR-ADM). | P0 |

### 2.2 Player Profile & Progression (PROF)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-PROF-01 | Setiap akun memiliki profil: player level, total XP, currencies (Gold, Gems, Event Tokens). | P0 |
| FR-PROF-02 | Player level MUST naik saat XP mencapai threshold; threshold bertambah per level (formula lihat §6). | P0 |
| FR-PROF-03 | Currencies MUST disimpan di ledger terpisah; tidak boleh negatif (invariant). | P0 |
| FR-PROF-04 | Sistem MUST mencatat mutasi currency (sumber, jumlah, timestamp, balance after). | P0 |
| FR-PROF-05 | Player SHOULD memiliki nickname (editable, unik opsional, filter kata kasar dasar). | P1 |
| FR-PROF-06 | Profil MUST menyimpan `last_seen` (epoch ms) untuk idle accrual. | P0 |
| FR-PROF-07 | Player MUST dapat melihat statistik diri (win rate PvP, total summons, dll). | P2 |
| FR-PROF-08 | Sistem SHOULD membatasi perubahan nickname (cooldown 7 hari). | P2 |

### 2.3 Hero / Party Management (HERO)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-HERO-01 | Player MUST dapat memiliki banyak hero; tiap hero punya: id, rarity, level, XP, baseStats, elemen, skill set. | P0 |
| FR-HERO-02 | Rarity MUST salah satu dari C/R/E/L (MVP); M hanya via banner khusus (post-MVP). | P0 |
| FR-HERO-03 | Party MUST memiliki maksimal 5 slot hero; tidak boleh >5. | P0 |
| FR-HERO-04 | Player MUST dapat assign/unassign hero ke party slot. | P0 |
| FR-HERO-05 | Hero MUST dapat di-level-up dengan XP/material; level cap awal 60 (asumsi). | P0 |
| FR-HERO-06 | Hero MUST dapat di-equip gear (weapon/armor/accessory — lihat FR-INV). | P0 |
| FR-HERO-07 | Effective stats hero = baseStats(level) + gear bonuses + status (saat combat). | P0 |
| FR-HERO-08 | Sistem SHOULD mendukung hero dupes → shard/star-up (asumsi mekanik). | P2 |
| FR-HERO-09 | Player MUST tidak dapat menggunakan hero yang sedang di-list marketplace (lock). | P0 |

### 2.4 Gacha / Summon (GAC)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-GAC-01 | Sistem MUST menyediakan pull gacha per banner dengan rate standar: C 60% / R 30% / E 9% / L 1%. | P0 |
| FR-GAC-02 | Pity counter MUST per-banner; tersimpan server-side. | P0 |
| FR-GAC-03 | Setiap 10-pull MUST guarantee minimal rarity R+. | P0 |
| FR-GAC-04 | Hard pity: pull ke-90 MUST guarantee L (per banner). | P0 |
| FR-GAC-05 | Soft pity: dari pull ke-75, probabilitas L meningkat bertahap hingga 90 (lihat §6.4). | P0 |
| FR-GAC-06 | Gacha roll MUST menggunakan seeded RNG tersimpan & auditable (seed + counter disimpan). | P0 |
| FR-GAC-07 | Setiap pull MUST dicatat di history log (akun, banner, timestamp, rarity, hero id, seed). | P0 |
| FR-GAC-08 | Player MUST dapat melihat history gacha (pagination). | P0 |
| FR-GAC-09 | Banner MUST memiliki jadwal (start/end) yang diatur Admin. | P0 |
| FR-GAC-10 | Pull MUST memotong currency (Gems/Event Token tergantung banner) via ledger atomik. | P0 |
| FR-GAC-11 | Sistem MUST mencegah pull bila currency tidak cukup (no negative). | P0 |
| FR-GAC-12 | Sistem SHOULD menampilkan rate & pity tersisa di UI (transparansi). | P1 |
| FR-GAC-13 | Banner Mythic (M) post-MVP: pool terpisah, rate M terpisah, bukan MVP. | P2 |

### 2.5 Combat Engine (CMB)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-CMB-01 | Combat MUST di-resolve di server (server-authoritative). Klien hanya submit intent & render. | P0 |
| FR-CMB-02 | Turn order MUST ditentukan oleh speed: urutan descending speed; tie-break oleh hero id. | P0 |
| FR-CMB-03 | Tiap hero MUST memiliki resource mana/energy; skill membutuhkan cost tertentu. | P0 |
| FR-CMB-04 | Sistem MUST mendukung status effect: Burn, Freeze, Poison, Stun, AtkDown, DefDown, Shield, Regen. | P0 |
| FR-CMB-05 | Elemen MUST: Fire, Water, Earth, Wind, Light, Dark. | P0 |
| FR-CMB-06 | Elemen affinity: damage multiplier 1.5x (strong) / 0.75x (weak) / 1.0x (neutral). | P0 |
| FR-CMB-07 | Combat MUST deterministic di server (seed combat + inputs); replayable untuk audit. | P0 |
| FR-CMB-08 | Sistem MUST menangani end-condition: semua musuh mati = win; semua hero mati = lose. | P0 |
| FR-CMB-09 | Battle resolve SHOULD selesai < 1 detik untuk simulasi penuh (lihat NFR-PERF). | P0 |
| FR-CMB-10 | Combat log MUST dikembalikan ke klien untuk animasi (tanpa reveals state rahasia musuh sebelum waktunya). | P0 |
| FR-CMB-11 | Sistem MUST memvalidasi input klien (skill ada, cukup mana, target valid) sebelum resolve. | P0 |

### 2.6 Wave-based PvE (PVE)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-PVE-01 | Stage MUST tersusun dari beberapa wave; tiap wave sekumpulan musuh. | P0 |
| FR-PVE-02 | Stage progression MUST lock sampai stage sebelumnya clear (atau konfigurabel). | P0 |
| FR-PVE-03 | Musuh per wave MUST di-scale (HP/ATK naik per stage index via formula). | P0 |
| FR-PVE-04 | Clear stage MUST memberikan reward (Gold, XP, material, chance gear). | P0 |
| FR-PVE-05 | Combat PvE MUST menggunakan Combat Engine server (FR-CMB). | P0 |
| FR-PVE-06 | Auto-battle (idle-friendly) MUST didukung: party otomatis serang tiap tick. | P0 |
| FR-PVE-07 | Sistem MUST menyimpan highest stage reached per akun. | P0 |
| FR-PVE-08 | Sweep/fast-clear SHOULD tersedia untuk stage sudah 3-star (asumsi). | P1 |
| FR-PVE-09 | Stamina/energy masuk stage SHOULD ada (asumsi, untuk monetisasi). | P1 |

### 2.7 PvP (PVP)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-PVP-01 | Sistem MUST menyediakan matchmaking berbasis ELO sederhana (start 1000). | P0 |
| FR-PVP-02 | ELO MUST diperbarui pasca-match dengan K-factor (mis. K=32, asumsi). | P0 |
| FR-PVP-03 | Ladder tier MUST diturunkan dari ELO (Bronze…Legend, asumsi threshold). | P0 |
| FR-PVP-04 | PvP combat MUST server-authoritative (reuse FR-CMB). | P0 |
| FR-PVP-05 | Player MAY menentukan defense team (party tersimpan) untuk diserang lawan. | P0 |
| FR-PVP-06 | Matchmaking SHOULD mencari lawan dengan Δ ELO kecil (mis. ±150). | P1 |
| FR-PVP-07 | PvP reward MUST berupa Event Tokens / ranking points per season. | P0 |
| FR-PVP-08 | Season SHOULD reset periodik (asumsi 28 hari) dengan soft reset ELO. | P2 |
| FR-PVP-09 | Sistem MUST mencegah matchmaking saat akun suspended. | P0 |

### 2.8 Idle System (IDL)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-IDL-01 | Sistem MUST menjalankan idle tick real-time setiap 5 detik saat player online. | P0 |
| FR-IDL-02 | Idle accrual offline MUST dihitung server-side dari `(now - last_seen)` clamp [0, 24h]. | P0 |
| FR-IDL-03 | Offline reward MUST di-claim saat player kembali online (endpoint claim). | P0 |
| FR-IDL-04 | Offline reward formula MUST deterministik & auditable (lihat §6.1). | P0 |
| FR-IDL-05 | Cap offline MUST 24 jam; lebih dari itu dihitung sebagai 24 jam. | P0 |
| FR-IDL-06 | Klien PWA SHOULD queue claim via IndexedDB bila offline, kirim saat online. | P1 |
| FR-IDL-07 | Idle reward SHOULD skala dengan stage/progression player. | P1 |
| FR-IDL-08 | Sistem MUST memperbarui `last_seen` tiap aksi state-changing & tiap tick online. | P0 |

### 2.9 Inventory & Equipment (INV)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-INV-01 | Inventory MUST mendukung 3 kategori: Gear, Consumable, Material. | P0 |
| FR-INV-02 | Gear MUST memiliki: jenis (weapon/armor/accessory), rarity, stat bonus. | P0 |
| FR-INV-03 | Hero MUST dapat equip 1 weapon, 1 armor, 1 accessory (slot terbatas). | P0 |
| FR-INV-04 | Equip/unequip MUST memvalidasi ownership & tidak double-equip. | P0 |
| FR-INV-05 | Consumable MUST dapat digunakan (heal, buff) dengan efek terdefinisi. | P0 |
| FR-INV-06 | Material MUST dapat dikonsumsi untuk level-up/upgrade (FR-HERO-05). | P0 |
| FR-INV-07 | Inventory MUST memiliki kapasitas (soft cap; expand via item, asumsi). | P1 |
| FR-INV-08 | Item MUST memiliki id unik instance (terutama Gear) untuk trade. | P0 |
| FR-INV-09 | Sistem MUST mencegah penggunaan item saat tidak dalam kondisi valid (mis. consumable di luar battle). | P0 |
| FR-INV-10 | Gear SHOULD dapat di-enhance/upgrade (asumsi level gear). | P2 |

### 2.10 Marketplace / Trading (MKT)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-MKT-01 | Player MUST dapat membuat listing jual item (Gear/Material, asumsi Consumable excluded). | P0 |
| FR-MKT-02 | Listing MUST masuk escrow: item dikunci (FR-HERO-09) di server saat listing aktif. | P0 |
| FR-MKT-03 | Buyer MUST membayar currency; ledger atomik: transfer item + currency tanpa double-spend. | P0 |
| FR-MKT-04 | Marketplace MUST memotong fee (persen konfigurable, asumsi 5%). | P0 |
| FR-MKT-05 | Seller MUST dapat cancel listing; item dikembalikan dari escrow. | P0 |
| FR-MKT-06 | Sistem MUST mencegah penjualan item yang sedang di-equip/di-lock. | P0 |
| FR-MKT-07 | Listing MUST memiliki durasi (asumsi 24–72 jam) + auto-expire. | P1 |
| FR-MKT-08 | Setiap transaksi MUST tercatat di ledger terpisah (audit). | P0 |
| FR-MKT-09 | Harga MUST dibatasi range (min/max) untuk cegah abuse (asumsi). | P1 |
| FR-MKT-10 | Sistem MUST mencegah self-trade (akun sendiri beli listing sendiri). | P0 |
| FR-MKT-11 | Marketplace SHOULD memiliki search/filter (rarity, elemen, stat). | P1 |

### 2.11 Guild + Raid / Co-op (GLD)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-GLD-01 | Player MUST dapat membuat/mengikuti guild (nama, kapasitas asumsi 30). | P2 |
| FR-GLD-02 | Guild MUST memiliki membership list + role (leader/officer/member). | P2 |
| FR-GLD-03 | Guild chat SHOULD opsional (real-time ringan, asumsi WebSocket ephemeral). | P2 |
| FR-GLD-04 | Guild raid MUST berupa instance co-op: kontribusi damage akumulatif anggota. | P2 |
| FR-GLD-05 | Raid reward MUST dibagi berdasarkan kontribusi (formula asumsi). | P2 |
| FR-GLD-06 | Sistem MUST mencegah double-claim raid reward (idempoten per member per raid). | P2 |

### 2.12 Shop + Monetisasi (SHOP)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-SHOP-01 | Shop MUST menjual paket Gems (hard currency) via Payment Provider. | P0 |
| FR-SHOP-02 | Shop SHOULD menjual Battle Pass (free + premium track, asumsi). | P1 |
| FR-SHOP-03 | Shop MAY menjual Subscription (daily gems, asumsi). | P2 |
| FR-SHOP-04 | Shop SHOULD memiliki IAP one-time bundle (starter pack). | P1 |
| FR-SHOP-05 | Pembayaran MUST dikonfirmasi via webhook Payment Provider sebelum grant. | P0 |
| FR-SHOP-06 | Sistem MUST mencegah grant ganda dari webhook (idempotency key). | P0 |
| FR-SHOP-07 | Ads (rewarded) MAY tersedia untuk bonus idle (asumsi, bila ada SDK). | P2 |
| FR-SHOP-08 | Harga & ketersediaan paket MUST dikonfigurasi Admin. | P0 |

### 2.13 Liveops / Event (LIVE)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-LIVE-01 | Sistem MUST mendukung event berjadwal (start/end, reward). | P1 |
| FR-LIVE-02 | Event SHOULD memberikan Event Tokens / item spesifik. | P1 |
| FR-LIVE-03 | Event MAY berupa login-streak, stage-clear milestone, dll (konfigurable). | P2 |
| FR-LIVE-04 | Admin MUST dapat membuat/mengubah/hapus event (lihat FR-ADM). | P1 |
| FR-LIVE-05 | Sistem MUST menghentikan reward event saat event berakhir (server time). | P1 |

### 2.14 Admin Panel / GM Tools (ADM)

| ID | Requirement | Prioritas |
|---|---|---|
| FR-ADM-01 | Admin MUST login terpisah (role-based) dengan 2FA sederhana (asumsi TOTP). | P0 |
| FR-ADM-02 | Admin MUST dapat CRUD konten: hero, item, stage, banner. | P0 |
| FR-ADM-03 | Admin MUST dapat mengatur drop rate & pool banner gacha. | P0 |
| FR-ADM-04 | Admin MUST dapat grant/revoke item & currency ke akun (dengan reason). | P0 |
| FR-ADM-05 | Admin MUST dapat ban/suspend akun (soft/hard). | P0 |
| FR-ADM-06 | Setiap aksi Admin MUST tercatat di audit log (who/what/when/before-after). | P0 |
| FR-ADM-07 | Admin MUST dapat melihat dashboard metrik (DAU, revenue, fraud signals). | P1 |
| FR-ADM-08 | Admin SHOULD dapat query player state (inventory, ledger) untuk support. | P1 |
| FR-ADM-09 | Sistem MUST membatasi aksi destruktif (delete massal) via confirmasi + log. | P0 |
| FR-ADM-10 | Config (rates, fees, caps) MUST hot-reload tanpa restart (Redis config cache). | P1 |

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Performance (PERF)

| ID | Requirement | Target |
|---|---|---|
| NFR-PERF-01 | API latency p99 (selain battle resolve) | < 300 ms |
| NFR-PERF-02 | Battle resolve (full simulasi server) | < 1 detik |
| NFR-PERF-03 | Gacha pull (single/10) | < 500 ms |
| NFR-PERF-04 | Idle claim | < 500 ms |
| NFR-PERF-05 | Availability MVP | 99.5% bulanan |
| NFR-PERF-06 | Concurrent players (awal) | 50–200 (headroom 10x) |
| NFR-PERF-07 | Database query p95 | < 100 ms (terindeks) |
| NFR-PERF-08 | Static asset load (PWA) via CDN | < 2 s first paint (desktop) |
| NFR-PERF-09 | Scalability | Horizontal stateless API + Redis; DB read-replica bila perlu |
| NFR-PERF-10 | Idle tick scheduler | 5s interval, batch per shard |

**Trade-off:** Target p99 300ms mengasumsikan DB terindeks & cache Redis aktif. Bila query kompleks (leaderboard) melebihi, kita terima p95 lebih longgar untuk endpoint spesifik (dokumentasikan SLA per-endpoint).

### 3.2 Security (SEC)

**Prinsip utama:** *Server-authoritative*. Klien tidak pernah sumber kebenaran. Segala validasi penting dilakukan server.

#### 3.2.1 OWASP Top 10 Baseline (2021)

| OWASP | Kontrol yang Diterapkan |
|---|---|
| A01 Broken Access Control | RBAC (Player/Admin), object-level authz tiap endpoint, deny-by-default. |
| A02 Cryptographic Failures | TLS 1.2+ wajib, hash password argon2id, JWT signed (HS256/RS256), secrets di Vault/Env. |
| A03 Injection | Parameterized query (SQL), schema validasi input (zod/ class-validator),no string concat. |
| A04 Insecure Design | Server-authoritative, anti-cheat by design, threat model di §3.2.4. |
| A05 Security Misconfiguration | Header keamanan (CSP, HSTS, X-Content-Type), disable stack trace prod. |
| A06 Vulnerable Components | Dependency scan (npm audit/ Snyk) di CI, lockfile. |
| A07 Auth Failures | Rate-limit login, lockout, refresh token rotation, MFA admin. |
| A08 Software & Data Integrity | Webhook signature verify, CI signed artifacts, no eval. |
| A09 Logging Failures | Audit log terpusat, no secret logging, tamper-evident. |
| A10 SSRF | Validasi URL webhook/outbound, allowlist domain. |

#### 3.2.2 Daftar Kontrol Keamanan

- [ ] **SEC-01** HTTPS/TLS everywhere (HSTS preload).
- [ ] **SEC-02** Password policy: min 8 char, argon2id (cost tunable).
- [ ] **SEC-03** Rate limiting per-IP & per-account (token bucket) di API gateway/Redis.
- [ ] **SEC-04** Brute-force protection login (FR-AUTH-04).
- [ ] **SEC-05** JWT access short-lived (15 menit), refresh via httpOnly+Secure+SameSite cookie.
- [ ] **SEC-06** Anti-replay: nonce/timestamp + signature tiap request state-changing penting (gacha, trade).
- [ ] **SEC-07** Input validation strict (whitelist, length, type).
- [ ] **SEC-08** Output encoding; tidak ada `innerHTML` sembarang.
- [ ] **SEC-09** CSP untuk cegah XSS; nonces untuk script.
- [ ] **SEC-10** Secrets tidak di-bundle ke klien; aman di env server.
- [ ] **SEC-11** Audit log write-only (append-only), akses terbatas Admin.
- [ ] **SEC-12** Webhook HMAC verify (Payment, Email bounce).
- [ ] **SEC-13** Database least-privilege user; tidak ada akses langsung klien ke DB.
- [ ] **SEC-14** Penetration test ringan pra-rilis (OWASP ZAP).
- [ ] **SEC-15** Timeout koneksi & payload size limit (cegah abuse).
- [ ] **SEC-16** Anti-automation: deteksi pola anomali (banyak akun 1 device, idle-farm).

#### 3.2.3 Anti-Cheat (Desain)

- Klien tidak hitung damage/combat; kirim hanya intent (skill id, target id).
- Server validasi intent (mana cukup? skill ada? target hidup?).
- Tidak ada nilai HP/ currency di klien yang dihormati server.
- RNG gacha & combat di-seed server; klien tidak bisa memilih hasil.
- Replay battle untuk forensik (seed + inputs tersimpan).
- Deteksi: akun dengan win-rate/absurd stat → review manual.

#### 3.2.4 Threat Model (Singkat)

| Threat | Vector | Mitigasi |
|---|---|---|
| Cheat client (memory edit) | Modifikasi state lokal | Server-authoritative; ignore client state. |
| Gacha manipulation | Predict/RNG abuse | Seeded RNG server, pity server-side. |
| Double-spend currency | Race condition | Atomic ledger + DB transaction. |
| Marketplace fraud | Self-trade, item dup | Escrow + idempotency + ownership lock. |
| Brute force | Login spray | Rate-limit + lockout. |
| Replay attack | Resend request | Nonce + timestamp + signature. |
| Payment bypass | Fake webhook | HMAC verify + idempotency. |
| Admin abuse | Rogue GM | RBAC + 2FA + audit log + deny destructive. |

### 3.3 Privacy / Compliance (PRIV)

| ID | Requirement |
|---|---|
| NFR-PRIV-01 | Kumpulkan minimal data: email, password hash, nickname opsional, last_seen. |
| NFR-PRIV-02 | Consent ditampilkan saat registrasi (checkbox kebijakan privasi). |
| NFR-PRIV-03 | Player MUST dapat export data diri (GDPR/PDPA-style). |
| NFR-PRIV-04 | Player MUST dapat delete account (anonymize/erase) dalam SLA (asumsi 30 hari). |
| NFR-PRIV-05 | Retensi log: audit log 1 tahun; telemetri 90 hari (asumsi). |
| NFR-PRIV-06 | Data di-rest saat transit (TLS) & saat istirahat (DB encrypted at rest). |
| NFR-PRIV-07 | Tidak jual data ke pihak ke-3; analytics anonymized. |
| NFR-PRIV-08 | Akses data player oleh Admin MUST tercatat audit log. |

**Trade-off:** Export/delete penuh butuh pipeline ETL; MVP cukup endpoint status + Admin tool manual. Upgrade ke self-service di P1.

### 3.4 Observability (OBS)

| ID | Requirement |
|---|---|
| NFR-OBS-01 | Structured logging (JSON) via central logger (Winston/Pino → Loki/ELK). |
| NFR-OBS-02 | Metrics: request count, latency, error rate, DAU, revenue, gacha pulls (Prometheus). |
| NFR-OBS-03 | Distributed tracing untuk request lintas service (OpenTelemetry). |
| NFR-OBS-04 | Alerting: latency/error spike, fraud signal, payment failure. |
| NFR-OBS-05 | Battle & gacha events di-log untuk audit & debugging. |
| NFR-OBS-06 | Dashboard ops (Grafana) untuk Admin (FR-ADM-07). |
| NFR-OBS-07 | Error tracking (Sentry-like) untuk klien & server. |

---

## 4. Arsitektur yang Direkomendasikan

### 4.1 Stack yang Direkomendasikan

| Layer | Rekomendasi | Alasan | Alternatif |
|---|---|---|---|
| Frontend | React + Vite + vite-plugin-pwa | Ekosistem besar, HMR cepat, PWA plugin siap pakai (SW + manifest). | Vue 3 + Vite; SvelteKit. |
| State Mgmt | Zustand / Redux Toolkit | Ringan, cocok game state snapshot. | Tanstack Query untuk server state. |
| Backend API | Node.js + NestJS (TypeScript) | Struktur modular, decorator, validasi bawaan, cepat dev. | Go (Fiber/Echo) untuk perf tinggi; Rust (Axum). |
| Realtime | WebSocket (ws/socket.io) untuk idle tick & chat opsional | Push tick 5s efisien. | SSE. |
| Database | PostgreSQL 15+ | ACID, ledger transaksional, JSONB untuk state fleksibel. | MySQL; CockroachDB (scale). |
| Cache / Rate-limit | Redis 7 | Session, config hot-reload, rate-limit, queue, pub/sub tick. | Memcached (tanpa persist). |
| Queue / Scheduler | BullMQ (Redis) | Tick batch, email queue, webhook retry, idle accrual job. | Agenda; cron + pg_cron. |
| Auth | JWT (access) + httpOnly refresh cookie; argon2id | Standar, aman. | Session DB; OAuth (post-MVP). |
| File/CDN | CDN statis (Cloudflare/Vercel) | Asset PWA cepat, offline cache. | S3 + CloudFront. |
| Infra | Container (Docker) + orchestrator ringan (single node MVP, k8s bila scale). | Portabel, reproducible. | PM2 (sederhana). |
| CI/CD | GitHub Actions | Build, test, scan, deploy. | GitLab CI. |

**Alasan pemilihan utama:** TypeScript end-to-end (frontend + backend) mengurangi friction tipe; NestJS memberi struktur FR modular; PostgreSQL menjamin atomic ledger (krusial untuk currency/marketplace); Redis menangani tick/rate-limit/queue di satu tempat. Semua stateless kecuali DB/Redis → mudah horizontal scale.

### 4.2 Diagram Arsitektur (ASCII)

```
                         +-------------------+
                         |      Browser      |
                         |  (React PWA SW)   |
                         |  IndexedDB queue  |
                         +---------+---------+
                                   | HTTPS / WS
                         +---------v---------+
                         |   CDN / Edge      |
                         | (static assets)   |
                         +---------+---------+
                                   | /api
                         +---------v---------+
                         |  API Gateway /    |
                         |  Rate Limiter     |
                         |  (Redis bucket)   |
                         +---------+---------+
                                   |
                +------------------+------------------+
                |                  |                  |
        +-------v-------+  +-------v-------+  +-------v-------+
        |  Auth Service |  | Game API (Nest)|  | Admin API    |
        |  (JWT, argon) |  | FR modul semua |  | (RBAC, 2FA)  |
        +-------+-------+  +-------+-------+  +-------+-------+
                |                  |                  |
                +-------+----------+----------+-------+
                        |          |          |
                 +------v-----+ +--v-----+ +--v--------+
                 | PostgreSQL | | Redis  | | BullMQ     |
                 | (ledger,  | |(sess,  | | (tick,     |
                 |  state,    | |config, | | email,     |
                 |  content)  | |ratelimit| | webhook q) |
                 +------+-----+ +--------+ +------------+
                        |
                 +------v-----+        +------------------+
                 |  External   |<------>| Payment Provider |
                 |  Integrations|  webhook (HMAC)   |
                 | (Email,     |        +------------------+
                 |  Analytics) |
                 +-------------+
```

### 4.3 Data Flow (Contoh: Gacha Pull)

1. Klien kirim `POST /api/gacha/pull` + JWT + nonce + signature.
2. Gateway: rate-limit cek Redis → API Game validasi authz + currency cukup (ledger read).
3. Gacha Service: ambil seeded RNG + pity counter (Redis/Postgres) → tentukan rarity & hero.
4. Atomic DB tx: potong Gems, tambah hero ke inventory, tulis history, update pity.
5. Response JSON ke klien + audit event ke log.
6. Bila currency dari IAP, hanya setelah webhook terverifikasi (FR-SHOP-05).

### 4.4 Data Flow (Idle Claim)

1. Klien buka app → `POST /api/idle/claim`.
2. Server hitung `elapsed = clamp(now - last_seen, 0, 24h)` → reward = f(elapsed, stage).
3. Atomic ledger: +Gold/XP/material; update `last_seen = now`.
4. Response reward breakdown.
5. Bila offline saat claim, IndexedDB queue dikirim saat online.

---

## 5. API Spec Tingkat Tinggi

**Konvensi:**
- Base URL: `https://api.<domain>/api/v1`
- Auth: `Authorization: Bearer <access_token>` untuk kebanyakan; refresh via cookie `rt`.
- Semua timestamp epoch ms (UTC).
- Response envelope: `{ "ok": true, "data": ..., "error": null }`.
- Error envelope: `{ "ok": false, "data": null, "error": { "code": "...", "message": "..." } }`.

### 5.1 Auth

#### POST /auth/register
Request:
```json
{ "email": "player@example.com", "password": "Str0ng#Pass", "nickname": "Hero" }
```
Response 201:
```json
{ "ok": true, "data": { "accountId": "acc_123", "requiresVerification": false }, "error": null }
```

#### POST /auth/login
Request:
```json
{ "email": "player@example.com", "password": "Str0ng#Pass" }
```
Response 200:
```json
{
  "ok": true,
  "data": {
    "accessToken": "eyJ...",
    "expiresIn": 900,
    "accountId": "acc_123"
  },
  "error": null
}
```
(Refresh token di-set httpOnly cookie `rt`.)

#### POST /auth/refresh
Response 200 (rotasi):
```json
{ "ok": true, "data": { "accessToken": "eyJ...", "expiresIn": 900 }, "error": null }
```

#### POST /auth/logout
Response 200: `{ "ok": true, "data": null, "error": null }`

#### POST /auth/reset-request
```json
{ "email": "player@example.com" }
```
Response 202: `{ "ok": true, "data": { "sent": true }, "error": null }`

#### POST /auth/reset-confirm
```json
{ "token": "rst_abc", "newPassword": "New#Pass9" }
```
Response 200: `{ "ok": true, "data": null, "error": null }`

### 5.2 Profile & Progression

#### GET /profile
Response 200:
```json
{
  "ok": true,
  "data": {
    "accountId": "acc_123",
    "nickname": "Hero",
    "level": 12,
    "xp": 3400,
    "xpToNext": 1200,
    "currencies": { "gold": 15200, "gems": 320, "eventTokens": 45 },
    "lastSeen": 1752456000000
  },
  "error": null
}
```

#### GET /profile/ledger?currency=gold&limit=50&offset=0
Response 200:
```json
{
  "ok": true,
  "data": {
    "entries": [
      { "id": "tx_1", "type": "idle_claim", "delta": 500, "balanceAfter": 15200, "ts": 1752456000000 }
    ],
    "total": 1
  },
  "error": null
}
```

### 5.3 Hero & Party

#### GET /heroes
Response 200:
```json
{
  "ok": true,
  "data": {
    "heroes": [
      { "heroId": "h_1", "defId": "hero_fire_knight", "rarity": "R", "level": 30, "element": "Fire", "equipped": { "weapon": "itm_w1", "armor": null, "accessory": null } }
    ]
  },
  "error": null
}
```

#### PUT /party
Request:
```json
{ "slots": ["h_1", "h_2", null, "h_4", null] }
```
Response 200:
```json
{ "ok": true, "data": { "party": ["h_1","h_2",null,"h_4",null] }, "error": null }
```
Error 400 (slot >5 / duplikat): `{ "ok": false, "data": null, "error": { "code": "INVALID_PARTY", "message": "Max 5 unique heroes" } }`

#### POST /heroes/equip
```json
{ "heroId": "h_1", "itemId": "itm_w1", "slot": "weapon" }
```
Response 200: `{ "ok": true, "data": { "equipped": true }, "error": null }`

### 5.4 Gacha

#### POST /gacha/pull
Request:
```json
{ "bannerId": "banner_std_01", "count": 10, "nonce": "n_xyz", "sig": "hmac..." }
```
Response 200:
```json
{
  "ok": true,
  "data": {
    "pulls": [
      { "rarity": "R", "heroId": "h_2", "seed": "s_77" },
      { "rarity": "C", "heroId": "h_9", "seed": "s_78" }
    ],
    "pity": { "current": 10, "guaranteedRPlus": true, "hardPityAt": 90 },
    "cost": { "currency": "gems", "amount": 1000 }
  },
  "error": null
}
```
Error 402 (insufficient):
```json
{ "ok": false, "data": null, "error": { "code": "INSUFFICIENT_CURRENCY", "message": "Not enough gems" } }
```

#### GET /gacha/history?bannerId=banner_std_01&limit=20
Response 200:
```json
{
  "ok": true,
  "data": {
    "entries": [
      { "ts": 1752456000000, "bannerId": "banner_std_01", "rarity": "L", "heroId": "h_3", "seed": "s_12", "pullIndex": 90 }
    ]
  },
  "error": null
}
```

### 5.5 Combat

#### POST /combat/start
```json
{ "mode": "pve", "stageId": "stage_10", "party": ["h_1","h_2",null,"h_4",null] }
```
Response 200:
```json
{ "ok": true, "data": { "battleId": "b_99", "status": "resolved", "result": "win", "log": [ "...turn snapshots..." ], "rewards": { "gold": 300, "xp": 120 } }, "error": null }
```
(PvP: `mode":"pvp"`, `opponentId`.)

### 5.6 PvE

#### GET /pve/stages?page=1
Response 200:
```json
{ "ok": true, "data": { "stages": [ { "stageId": "stage_10", "index": 10, "waves": 3, "reward": { "gold": 300 } } ], "highestCleared": 9 }, "error": null }
```

#### POST /pve/sweep  (asumsi, P1)
```json
{ "stageId": "stage_9", "times": 5 }
```

### 5.7 PvP

#### POST /pvp/matchmaking
Response 200:
```json
{ "ok": true, "data": { "opponentId": "acc_456", "opponentElo": 1010, "battleId": "b_100" }, "error": null }
```

#### GET /pvp/rank
Response 200:
```json
{ "ok": true, "data": { "elo": 1000, "tier": "Bronze", "rank": 42 }, "error": null }
```

### 5.8 Idle

#### POST /idle/claim
Response 200:
```json
{
  "ok": true,
  "data": {
    "elapsedSeconds": 43200,
    "cappedAt24h": false,
    "rewards": { "gold": 8640, "xp": 2000, "materials": [{ "itemId": "mat_1", "qty": 5 }] },
    "lastSeen": 1752542400000
  },
  "error": null
}
```
(Error 409 bila sudah claim baru-baru ini tanpa elapsed.)

#### GET /idle/status
Response 200:
```json
{ "ok": true, "data": { "lastSeen": 1752456000000, "projectedGoldPerHour": 720 }, "error": null }
```

### 5.9 Inventory

#### GET /inventory?category=gear
Response 200:
```json
{
  "ok": true,
  "data": {
    "items": [
      { "itemId": "itm_w1", "defId": "gear_sword_r", "category": "gear", "rarity": "R", "stats": { "atk": 40 }, "locked": false }
    ]
  },
  "error": null
}
```

#### POST /inventory/use  (consumable)
```json
{ "itemId": "itm_c1", "targetHeroId": "h_1" }
```

### 5.10 Marketplace

#### POST /marketplace/list
```json
{ "itemId": "itm_w1", "price": { "currency": "gems", "amount": 500 }, "durationHrs": 48 }
```
Response 200:
```json
{ "ok": true, "data": { "listingId": "lst_1", "status": "active", "itemLocked": true }, "error": null }
```

#### GET /marketplace/listings?rarity=R&page=1
Response 200:
```json
{ "ok": true, "data": { "listings": [ { "listingId": "lst_1", "itemId": "itm_w1", "price": { "currency":"gems","amount":500 }, "sellerId": "acc_123" } ] }, "error": null }
```

#### POST /marketplace/buy
```json
{ "listingId": "lst_1", "nonce": "n_2", "sig": "hmac..." }
```
Response 200 (atomic escrow settle):
```json
{
  "ok": true,
  "data": {
    "txId": "tx_mkt_1",
    "itemTransferred": "itm_w1",
    "fee": { "currency": "gems", "amount": 25 },
    "sellerReceived": { "currency": "gems", "amount": 475 }
  },
  "error": null
}
```

#### POST /marketplace/cancel
```json
{ "listingId": "lst_1" }
```
Response 200: `{ "ok": true, "data": { "itemReturned": "itm_w1" }, "error": null }`

### 5.11 Shop

#### POST /shop/purchase
```json
{ "packageId": "pkg_gems_100" }
```
Response 202 (menunggu webhook):
```json
{ "ok": true, "data": { "orderId": "ord_1", "status": "pending" }, "error": null }
```

#### POST /shop/webhook  (dari Payment Provider)
```json
{ "orderId": "ord_1", "status": "paid", "sig": "provider_hmac" }
```
Response 200 (idempoten): `{ "ok": true, "data": { "granted": true }, "error": null }`

### 5.12 Admin (RBAC)

#### POST /admin/auth/login (2FA TOTP)
```json
{ "email": "gm@game", "password": "x", "totp": "123456" }
```

#### POST /admin/grant
```json
{ "accountId": "acc_123", "item": { "defId": "gear_sword_l", "qty": 1 }, "reason": "compensation" }
```
Response 200: `{ "ok": true, "data": { "granted": true, "auditId": "aud_1" }, "error": null }`

#### POST /admin/banner/upsert
```json
{ "bannerId": "banner_std_02", "pool": ["hero_fire_knight","..."], "rates": { "C":0.6,"R":0.3,"E":0.09,"L":0.01 }, "start": 1753000000000, "end": 1754000000000 }
```

#### GET /admin/audit?from=..&to=..&actor=gm1

#### POST /admin/account/ban
```json
{ "accountId": "acc_999", "type": "soft", "reason": "cheat" }
```

---

## 6. Aturan Bisnis Kritis

### 6.1 Offline Reward Calculation

**Prinsip:** Akrual dihitung server saat claim, bukan disimpan progresif di klien.

```
function computeIdleReward(lastSeen, now, player):
    elapsed_ms = now - lastSeen
    elapsed_sec = max(0, elapsed_ms / 1000)
    elapsed_sec = clamp(elapsed_sec, 0, 24*3600)        # cap 24 jam
    hours = elapsed_sec / 3600

    baseGoldPerHour = player.idleBaseGold            # skala dgn stage/progress
    gold = floor(baseGoldPerHour * hours)

    xp = floor(player.idleXpPerHour * hours)
    mats = floor(player.idleMatPerHour * hours)

    return {
      elapsedSeconds: floor(elapsed_sec),
      cappedAt24h: (elapsed_ms/1000) > 24*3600,
      rewards: { gold, xp, materials: mats }
    }

# Invariant: reward NEVER negatif; lastSeen selalu di-update ke now pasca-claim.
# Anti-abuse: bila elapsed < threshold kecil (mis < 60s), reward = 0 (hindari spam claim).
```

**Catatan:** `player.idleBaseGold` diturunkan dari `highestStageReached` (asumsi formula `baseGold = 50 + 10*highestStage`). Admin dapat override via config.

### 6.2 Anti-Cheat Rules

1. **Client is never source of truth.** Seluruh state ekonomi/combat/gacha di server.
2. **Intent-only submission.** Klien kirim `skillId` + `targetId`, bukan `damage`.
3. **Server validates every intent.** Mana cukup? Target valid? Skill on cooldown?
4. **No client RNG.** Semua roll (gacha, combat crit) di-seed server.
5. **Replayable battles.** Simpan `(seed, inputs[])`. Bila perlu, re-simulasi untuk verifikasi.
6. **Ledger invariant.** Currency tidak pernah negatif; semua mutasi via transaksi atomik.
7. **Rate-limit & anomaly.** Deteksi idle-farm multi-akun, win-rate anomali → review.
8. **Signature/nonce** untuk aksi state-changing krusial cegah replay.

### 6.3 Marketplace Integrity (Escrow + Atomic Ledger)

```
function buyListing(buyerId, listingId, nonce, sig):
    assert validSignature(buyerId, nonce, sig)
    assert not buyerId == listing.sellerId          # FR-MKT-10
    listing = lock(listingId)                       # row lock / tx
    assert listing.status == 'active'
    price = listing.price
    fee = floor(price.amount * MARKET_FEE)          # FR-MKT-04
    net = price.amount - fee

    BEGIN TRANSACTION:
        assert buyer.currency[price.currency] >= price.amount
        buyer.currency -= price.amount
        seller.currency += net
        treasury.currency += fee
        transferItemInstance(listing.itemId, from escrow -> buyer)
        listing.status = 'sold'
        writeLedger(buyer, -price, 'market_buy', txId)
        writeLedger(seller, +net, 'market_sell', txId)
        writeLedger(treasury, +fee, 'market_fee', txId)
        writeMarketAudit(txId, ...)
    COMMIT

    return txId
```

- Item masuk **escrow** saat listing (tidak bisa di-equip/di-jual lain).
- Semua mutasi dalam **satu DB transaction** → tidak ada double-spend / item hilang.
- Idempotensi via `txId` → webhook/retry aman.

### 6.4 Gacha Fairness (Pity, Transparency, Auditable RNG)

**Rate standar (per pull, pre-pity):**
| Rarity | Base Rate |
|---|---|
| C | 60% |
| R | 30% |
| E | 9% |
| L | 1% |

**Pity logic (per banner):**
```
function rollGacha(banner, account):
    counter = banner.pity[account]            # per-banner
    counter += 1

    if counter == 10 and not guaranteedThisTen:
        force rarity >= R                      # FR-GAC-03
    if counter >= 90:
        result = 'L'                           # hard pity FR-GAC-04
    else if counter >= 75:
        boost = softPityBoost(counter)         # mis: (counter-74)*0.06 tambahan ke L
        result = weightedRoll(ratesWithBoost)
    else:
        result = weightedRoll(baseRates)

    hero = pickFromPool(result, seededRNG)     # auditable seed
    save seed, counter, history
    return { rarity: result, heroId, seed, pullIndex: counter }
```

- **Soft pity:** dari pull 75, prob L naik linear ke 100% di 90 (asumsi `boost = (counter-74)/15 * (1 - baseL)` ditambah ke L, sisa di-distribusi proporsional).
- **Transparansi:** UI tunjukkan rate & pity tersisa (FR-GAC-12).
- **Auditabilitas:** `seed` + `pullIndex` + `bannerId` tersimpan → player & auditor dapat rekonstruksi.
- **Anti-manipulasi:** counter & seed di server; klien tidak tahu hasil sebelum response.

---

## 7. Test Plan

### 7.1 Level Pengujian

| Level | Tools (rekomendasi) | Cakupan |
|---|---|---|
| Unit | Jest / Vitest | Formula (idle, pity, ELO, ledger), validasi. |
| Integration | Supertest + Testcontainers (PG/Redis) | Endpoint + DB transaction. |
| E2E | Playwright (PWA desktop) | Login → gacha → combat → idle → market. |
| Security | OWASP ZAP, custom abuse scripts | §3.2. |
| Load | k6 / Artillery | Skala 10–50 DAU + spike 10x. |
| Chaos | Tambahan (post-MVP) | DB restart, Redis down. |

### 7.2 Unit Test Kritis

- `computeIdleReward`: elapsed 0 → 0; elapsed 24h → cap; elapsed 48h → dihitung 24h.
- `rollGacha`: pull ke-10 → R+; pull ke-90 → L; soft pity naik.
- `applyElo`: win/lose update sesuai K-factor.
- `ledgerMutate`: reject negatif balance.

### 7.3 Integration Test Kritis

- Gacha pull memotong gems & tambah hero dalam 1 tx.
- Marketplace buy: atomic, fee benar, item pindah, no double-spend (retry idempoten).
- Idle claim update last_seen & tidak bisa claim dua kali tanpa elapsed.
- Admin grant tercatat audit log.

### 7.4 E2E Test Kritis

1. Registrasi → verifikasi → login → dapatkan party.
2. Gacha 10-pull → history tampil → pity benar.
3. PvE stage clear → reward ledger.
4. Offline simulasi (ubah last_seen) → claim → cap 24h.
5. Marketplace list → buy (akun lain) → escrow settle.
6. PvP matchmaking → battle → ELO update.

### 7.5 Security Test

- Login brute-force → lockout aktif.
- SQL injection pada filter marketplace → ditolak.
- Replay request gacha (same nonce) → ditolak.
- Fake payment webhook (bad sig) → tidak grant.
- Akses endpoint Admin tanpa role → 403.
- XSS via nickname → di-encode / CSP blokir.

### 7.6 Load Test (Skala Awal)

| Skenario | Target |
|---|---|
| Steady 50 DAU concurrent | p99 < 300ms, error < 0.1% |
| Spike 500 concurrent (10x) | p99 < 800ms, no data loss |
| Gacha burst 1000 pull/menit | ledger konsisten, no negatif |
| Idle claim burst (login pagi) | queue BullMQ handle, no dup |

**Kriteria lulus:** Tidak ada ledger negatif, tidak ada item duplikat, tidak ada panic OOM.

---

## 8. Appendix

### 8.1 Glosarium

| Term | Arti |
|---|---|
| DAU | Daily Active Users. |
| PWA | Progressive Web App. |
| MVP | Minimum Viable Product. |
| Gacha | Sistem summon acak berbayar. |
| Pity | Penjaminan rarity setelah N pull. |
| Escrow | Penahanan item di server saat listing. |
| Ledger | Buku besar mutasi currency (immutable). |
| ELO | Sistem rating PvP. |
| Soft cap | Batas lembut (bisa dilewati dengan syarat). |

### 8.2 Traceability (CDP → FR/NFR)

| CDP | Pemenuhan |
|---|---|
| Desktop-first + PWA | CON-01, FR-IDL-06, §4.1 |
| Party max 5 | FR-HERO-03, FR-PVE-06 |
| Rarity C/R/E/L + M | FR-HERO-02, FR-GAC-01, FR-GAC-13 |
| Gacha rate & pity | FR-GAC-01..05, §6.4 |
| Idle 5s tick, cap 24h | FR-IDL-01..05, §6.1 |
| Items 3 kategori | FR-INV-01 |
| PvP ELO | FR-PVP-01..03 |
| Currencies Gold/Gems/Token | FR-PROF-01, FR-PROF-03 |
| Combat elemen & status | FR-CMB-02..06 |
| Skala 10–50 DAU | NFR-PERF-06, §7.6 |
| Auth email/password | FR-AUTH-01..10 |
| Server-authoritative | FR-CMB-01, §3.2, §6.2 |

### 8.3 Trade-off Ringkas

- **Server-authoritative vs responsivitas:** Pilih server-authoritative (keamanan > latensi marginal). Battle resolve <1s tetap aman.
- **PostgreSQL vs NoSQL:** Pilih PG untuk atomic ledger & relasi; scale via read-replica bila perlu.
- **Node/NestJS vs Go:** Node dipilih untuk kecepatan dev & TS end-to-end; Go bila CPU-bound terbukti bottleneck.
- **Seeded auditable RNG vs provable fairness:** Mulai seeded (cukup transparansi); upgrade ke verifiable bila regulasi wajib.
- **Escrow marketplace vs instant:** Escrow dipilih untuk integritas; trade-off sedikit kompleksitas.

### 8.4 Open Questions (Perlu Keputusan)

| ID | Pertanyaan | Rekomendasi Awal |
|---|---|---|
| OQ-01 | Level cap hero & stage awal? | 60 / 100 (asumsi). |
| OQ-02 | Stamina masuk PvE? | Ya (P1) untuk monetisasi. |
| OQ-03 | Formula affinity elemen (matrix penuh)? | Lihat §6.6 draft (perlu confirm desainer). |
| OQ-04 | K-factor ELO pasti? | K=32 (asumsi). |
| OQ-05 | Marketplace fee %? | 5% (asumsi). |
| OQ-06 | Hero dupe → star-up? | Ya (P2). |

### 8.5 Draft Matrix Elemen (OQ-03)

| Atk \ Def | Fire | Water | Earth | Wind | Light | Dark |
|---|---|---|---|---|---|---|
| Fire | 1.0 | 0.75 | 1.5 | 1.0 | 1.0 | 1.0 |
| Water | 1.5 | 1.0 | 0.75 | 1.0 | 1.0 | 1.0 |
| Earth | 1.0 | 1.5 | 1.0 | 0.75 | 1.0 | 1.0 |
| Wind | 0.75 | 1.0 | 1.0 | 1.0 | 1.5 | 1.0 |
| Light | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | 1.5 |
| Dark | 1.0 | 1.0 | 1.0 | 1.0 | 1.5 | 1.0 |

> Catatan: Matrix di atas contoh; desainer game harus mengonfirmasi keseimbangan (OQ-03). Multiplier 1.5 (strong) / 0.75 (weak) sesuai CDP.

---

## 9. Data Model & Database Schema (Draft)

> Semua tabel berada di PostgreSQL. Kunci asing & indeks disederhanakan untuk keterbacaan. Ledger bersifat append-only (tidak ada UPDATE/DELETE baris mutasi).

### 9.1 Entity Relationship (Teks)

```
accounts 1──* heroes
accounts 1──* inventory_items
accounts 1──* party_slots (5)
accounts 1──* ledger_entries
accounts 1──* gacha_history
accounts 1──* marketplace_listings (as seller)
accounts 1──* pvp_stats
accounts 1──1 idle_state
accounts 1──* admin_audit_logs (as actor)
banners 1──* gacha_history
banners 1──* banner_pity_counters
listings 1──1 inventory_items (escrow)
stages 1──* stage_waves
guilds 1──* guild_members
```

### 9.2 Tabel Inti

| Tabel | Kolom Kunci | Catatan |
|---|---|---|
| `accounts` | id, email(unique), pw_hash, nickname, level, xp, status(suspend/active/ban), created_at, last_seen | Status akun untuk FR-AUTH-10. |
| `currencies` | account_id (FK), gold, gems, event_tokens, version (optimistic lock) | Invariant >=0 via CHECK constraint. |
| `ledger_entries` | id, account_id, currency, delta, balance_after, type, ref_id, ts | Append-only; sum delta = balance. |
| `heroes` | id, account_id, def_id, rarity, level, xp, element, skill_set(JSONB), equipped(JSONB) | Instance hero. |
| `party_slots` | account_id, slot(1..5), hero_id(nullable) | Max 5 (CHECK). |
| `inventory_items` | id, account_id, def_id, category(gear/consumable/material), rarity, stats(JSONB), locked(bool) | Instance item unik. |
| `banners` | id, name, pool(JSONB), rates(JSONB), start_ts, end_ts, status | FR-GAC-09. |
| `banner_pity_counters` | account_id, banner_id, counter, guaranteed_rplus | Per-banner (FR-GAC-02). |
| `gacha_history` | id, account_id, banner_id, ts, rarity, hero_id, seed, pull_index | Audit (FR-GAC-07). |
| `marketplace_listings` | id, seller_id, item_id(FK inventory_items), price_currency, price_amount, status(active/sold/cancelled/expired), expires_at | Escrow (FR-MKT-02). |
| `market_audits` | id, tx_id, buyer, seller, item_id, price, fee, ts | FR-MKT-08. |
| `pvp_stats` | account_id, elo, tier, rank, wins, losses, season_id | FR-PVP-01. |
| `idle_state` | account_id, last_seen, base_gold_per_hr, base_xp_per_hr | FR-IDL-02. |
| `admin_audit_logs` | id, actor_id, action, target, before(JSONB), after(JSONB), ts | FR-ADM-06. |
| `stages` | id, index, waves(JSONB), reward(JSONB), required_stage | FR-PVE-01. |
| `guilds` | id, name, leader_id, capacity | FR-GLD-01. |
| `admin_accounts` | id, email, pw_hash, totp_secret, role | FR-ADM-01. |

### 9.3 Invarian Data (Wajib)

- `currencies.gold/gems/event_tokens >= 0` (CHECK constraint + aplikasi).
- `SUM(ledger_entries.delta WHERE currency=X) = currencies.X` (rekonisiliasi periodik).
- `party_slots` tidak boleh duplikat hero dalam 1 akun.
- `marketplace_listings.status='active'` ⇒ item `locked=true`.
- `banner_pity_counters.counter` reset bila banner berakhir (atau per aturan banner).

---

## 10. Extended Functional Requirements (Detail Modul)

### 10.1 Auth — Detail Alur

| ID | Skenario | Behavior |
|---|---|---|
| FR-AUTH-11 | Email sudah terdaftar saat register | 409, tidak bocorkan keberadaan akun via message ambigu. |
| FR-AUTH-12 | Refresh token kadaluarsa | 401, wajib login ulang. |
| FR-AUTH-13 | Akses endpoint tanpa token | 401. |
| FR-AUTH-14 | Akses endpoint dengan role salah (player vs admin) | 403. |
| FR-AUTH-15 | Device fingerprint ringan untuk deteksi multi-akun (ASP-01) | Simpan hash device; flag anomali. |

### 10.2 Combat — Detail Status Effect

| Status | Efek | Durasi | Stacks? |
|---|---|---|---|
| Burn | Damage per tick = 5% max HP musuh. | 3 turn | Ya (cap 3). |
| Freeze | Skip turn target. | 1 turn | Tidak. |
| Poison | Damage = 3% max HP, ignore def. | 3 turn | Ya. |
| Stun | Skip turn + tidak bisa skill. | 1 turn | Tidak. |
| AtkDown | -30% ATK. | 2 turn | Refresh. |
| DefDown | -30% DEF. | 2 turn | Refresh. |
| Shield | Absorb damage = X. | 2 turn / sampai habis | Tidak. |
| Regen | Heal = 5% max HP per turn. | 3 turn | Ya. |

**Resolusi status:** Diterapkan di awal giliran penerima (server). Immutable order: Shield absorb → Regen/Burn/Poison damage → cleanse check.

### 10.3 PvE — Wave Scaling Formula (Asumsi)

```
enemyHp(stage)   = baseHp   * (1 + 0.08 * stage)
enemyAtk(stage)  = baseAtk  * (1 + 0.06 * stage)
waveMultiplier(w) = 1 + 0.10 * (w - 1)     # w = nomor wave
```

### 10.4 PvP — ELO Update

```
function applyElo(playerElo, oppElo, won, K=32):
    expected = 1 / (1 + 10^((oppElo - playerElo)/400))
    actual = won ? 1 : 0
    return round(playerElo + K * (actual - expected))
```

Tier (asumsi): Bronze <1100, Silver 1100–1299, Gold 1300–1499, Platinum 1500–1699, Diamond 1700+, Legend 1900+.

### 10.5 Monetisasi — Paket IAP (Contoh)

| Package | Gems | Harga (USD asumsi) | Bonus |
|---|---|---|---|
| starter | 100 | 0.99 | +1 R hero |
| small | 500 | 4.99 | — |
| medium | 1200 | 9.99 | +50 bonus |
| large | 3000 | 24.99 | +200 bonus |
| battlepass | — | 9.99 | premium track |

### 10.6 Admin — Matriks Hak Akses

| Aksi | GM | Lead GM | Super Admin |
|---|---|---|---|
| Grant item | Ya | Ya | Ya |
| Ban/Suspend | Ya | Ya | Ya |
| CRUD konten | Tidak | Ya | Ya |
| Ubah config rate | Tidak | Ya | Ya |
| Hapus akun (GDPR) | Tidak | Tidak | Ya |
| Lihat audit log | Ya | Ya | Ya |

---

## 11. Extended API Endpoints

### 11.1 Inventory — Enhance (P2)
#### POST /inventory/enhance
```json
{ "itemId": "itm_w1", "materials": ["mat_1","mat_1"], "currency": { "gold": 1000 } }
```

### 11.2 Marketplace — Search
#### GET /marketplace/search?element=Fire&stat=atk&minRarity=R&sort=price_asc&page=1

### 11.3 Guild
#### POST /guild/create `{ "name": "Dragons", "tag": "DRG" }`
#### POST /guild/join `{ "guildId": "g_1" }`
#### GET /guild/{guildId}/members
#### POST /guild/raid/start `{ "raidId": "raid_1" }`
#### POST /guild/raid/contribute `{ "raidInstanceId": "ri_1", "damage": 5000 }`

### 11.4 Liveops
#### GET /events/active
Response 200:
```json
{ "ok": true, "data": { "events": [ { "eventId": "ev_1", "type": "login_streak", "endTs": 1755000000000, "rewards": [ { "day":1,"itemId":"mat_1" } ] } ] }, "error": null }
```
#### POST /events/claim `{ "eventId": "ev_1" }`

### 11.5 Admin — Content CRUD
#### POST /admin/hero/upsert
```json
{ "defId": "hero_fire_knight", "name": "Fire Knight", "rarity": "R", "element": "Fire", "baseStats": { "hp": 800, "atk": 120, "def": 60, "spd": 105 }, "skills": ["skill_slash"] }
```
#### POST /admin/stage/upsert `{ "index": 11, "waves": [...], "reward": { "gold": 350 } }`
#### POST /admin/config/set `{ "key": "market_fee", "value": 0.05 }`  (hot-reload Redis, FR-ADM-10)
#### GET /admin/metrics  (dashboard, FR-ADM-07)

---

## 12. Extended Business Rules

### 12.1 Level & XP Curve (Asumsi)
```
xpToNext(level) = 100 * level * (1 + 0.1 * level)    # mis. L1→L2 = 110
onGainXp(account, amount):
    account.xp += amount
    while account.xp >= xpToNext(account.level) and level < CAP:
        account.xp -= xpToNext(level)
        account.level += 1
        applyLevelBonus(account)   # +baseStats
```

### 12.2 Equip Validation
```
canEquip(account, heroId, itemId, slot):
    assert item.category == 'gear'
    assert item.slotType == slot           # weapon/armor/accessory
    assert not item.locked                  # tidak di marketplace
    assert item.account_id == account.id
    assert hero.account_id == account.id
    # unequip existing di slot, set baru
```

### 12.3 Gacha Cost (Asumsi)
| Pull | Cost (Gems) |
|---|---|
| 1x | 100 |
| 10x | 1000 (hemat 0) — asumsi tanpa diskon awal |

Banner Event Token: hanya banner event tertentu yang terima Event Tokens (FR-GAC-10).

### 12.4 Marketplace Price Bounds (Asumsi)
```
MIN_PRICE = 10 gems   (atau setara)
MAX_PRICE = 99999 gems
assert MIN_PRICE <= price <= MAX_PRICE
```

### 12.5 Idle Tick Online (Real-time)
```
every 5 seconds (server scheduler per shard):
    for each online account in shard:
        apply passive small reward?  # MVP: akrual hanya di-claim saat offline;
                                      # online tick hanya update last_seen + animasi progres
        update last_seen if active
```
> Catatan: Tick 5 detik menurut CDP adalah interval akrual real-time saat online; reward aktual diklaim via endpoint claim (FR-IDL-03) menggunakan akrual offline-style yang sama. Konsisten: `elapsed` dihitung dari `last_seen`.

---

## 13. Risk Register

| ID | Risiko | Dampak | Mitigasi |
|---|---|---|---|
| RK-01 | Ledger race condition saat traffic spike | Currency negatif / duplikat | Transaksi atomik + optimistic lock version. |
| RK-02 | Webhook payment replay | Double grant | Idempotency key per order. |
| RK-03 | Cheat client prediksi gacha | Kehilangan trust | Seeded RNG server, pity server. |
| RK-04 | Admin rogue | Abuse grant/ban | RBAC + 2FA + audit log + deny destructive tanpa confirm. |
| RK-05 | DB down | Layanan mati | Read-replica, retry, graceful degradation (claim queue). |
| RK-06 | Skala melampaui 50 DAU cepat | Latensi naik | Stateless API → horizontal scale; Redis terdistribusi. |
| RK-07 | Regulatory gacha (bukti fairness) | Compliance | Seeded auditable + history log; upgrade verifiable bila wajib. |
| RK-08 | Multi-akun idle-farm | Ekonomi inflasi | Device fingerprint + rate-limit + cap. |

---

## 14. Acceptance Criteria (MVP Release Gate)

- [ ] Semua FR prioritas P0 lolos test integration/e2e.
- [ ] Ledger tidak pernah negatif di load test.
- [ ] Gacha pity terbukti di unit + integration (pull 90 → L).
- [ ] Offline cap 24h terbukti (simulasi last_seen).
- [ ] Marketplace no double-spend (replay buy ditolak).
- [ ] Admin grant/ban tercatat audit log.
- [ ] OWASP ZAP clean (kecuali false-positive terdokumentasi).
- [ ] PWA install + offline claim queue berfungsi.
- [ ] Availability 99.5% selama 1 minggu canary.
- [ ] Data export/delete (PRIV) tersedia.

---

## 15. Roadmap Post-MVP (Singkat)

| Fase | Item |
|---|---|
| P1 | Verifikasi email wajib, Battle Pass, Guild chat, Event engine, Self-service privacy. |
| P2 | Mythic banner, Hero star-up, Gear enhance, Raid co-op penuh, Subscription. |
| P3 | Localization, Cross-save, Verifiable RNG (audit independen), Turnamen PvP. |

---

## 16. Test Cases Terperinci (Assertions)

### 16.1 Gacha Pity (Unit + Integration)

| TC | Input | Expected |
|---|---|---|
| TC-GAC-01 | 9 pull biasa, pull ke-10 | rarity ∈ {R,E,L} (guaranteed R+) |
| TC-GAC-02 | counter=89, pull ke-90 | rarity == L (hard pity) |
| TC-GAC-03 | counter=74 | prob L ≈ base 1% |
| TC-GAC-04 | counter=82 | prob L > base (soft pity boost) |
| TC-GAC-05 | gems=900, 10-pull (butuh 1000) | 402 INSUFFICIENT_CURRENCY, no hero added |
| TC-GAC-06 | pull normal 1000x (simulasi) | distribusi ≈ 60/30/9/1 (±toleransi) |
| TC-GAC-07 | banner berakhir, counter reset | counter baru mulai 0 |

### 16.2 Offline Cap (Unit)

| TC | last_seen → now | Expected |
|---|---|---|
| TC-IDL-01 | selisih 0s | reward 0, capped=false |
| TC-IDL-02 | selisih 1h | reward = base*1h, capped=false |
| TC-IDL-03 | selisih 24h | reward = base*24h, capped=false |
| TC-IDL-04 | selisih 48h | reward = base*24h, capped=true |
| TC-IDL-05 | claim lalu claim lagi <60s | reward 0 (anti-spam) |

### 16.3 Currency Ledger (Integration)

| TC | Aksi | Expected |
|---|---|---|
| TC-LED-01 | spend lebih dari saldo | rejected, balance tetap |
| TC-LED-02 | grant + spend atomik | SUM(delta)==balance |
| TC-LED-03 | concurrent 100 transfers | no negative, no lost |
| TC-LED-04 | sum ledger != balance | alert rekonisiliasi |

### 16.4 Marketplace (Integration + Security)

| TC | Skenario | Expected |
|---|---|---|
| TC-MKT-01 | buy listing sendiri | 400 SELF_TRADE |
| TC-MKT-02 | buy dg item di-equip | 400 ITEM_LOCKED |
| TC-MKT-03 | buy sukses | item pindah, fee masuk treasury, no double-spend |
| TC-MKT-04 | replay same nonce | 409 / rejected |
| TC-MKT-05 | cancel lalu buy | 404 listing not found |
| TC-MKT-06 | listing expire | status expired, item returned |

### 16.5 Combat (Unit)

| TC | Skenario | Expected |
|---|---|---|
| TC-CMB-01 | speed hero A=120, B=100 | A giliran duluan |
| TC-CMB-02 | Fire vs Water | dmg *0.75 |
| TC-CMB-03 | Fire vs Earth | dmg *1.5 |
| TC-CMB-04 | Stun pada target | target skip turn |
| TC-CMB-05 | mana tidak cukup | intent ditolak, tidak resolve |
| TC-CMB-06 | semua musuh mati | result win, reward granted |

### 16.6 PvP ELO (Unit)

| TC | player/opp/won | Expected |
|---|---|---|
| TC-PVP-01 | 1000/1000/menang | elo > 1000 |
| TC-PVP-02 | 1000/1000/kalah | elo < 1000 |
| TC-PVP-03 | 1000/1500/menang | naik besar (expected rendah) |
| TC-PVP-04 | 1500/1000/kalah | turun kecil |

---

## 17. Combat Resolution Algorithm (Pseudocode Detail)

```
function resolveBattle(seed, playerParty, enemyParty, playerIntents[]):
    rng = seededRng(seed)
    turnOrder = sortBySpeedDesc([...playerParty, ...enemyParty])  # tie-break by id
    state = initState(playerParty, enemyParty)
    log = []

    while not isEnd(state):
        actor = turnOrder.nextAlive()
        if actor.hasStatus(Stun):
            applyStatusTick(actor)        # skip
            continue
        intent = getIntent(actor, playerIntents, rng)   # player: dari input; enemy: AI server
        validateIntent(state, actor, intent)             # mana, target, skill exist
        cost = skillCost(intent.skill)
        actor.mana -= cost
        dmg = computeDamage(actor, intent.target, intent.skill, rng)
        dmg = applyElementMultiplier(dmg, actor.element, intent.target.element)
        dmg = applyDefense(dmg, intent.target)
        dmg = applyShield(dmg, intent.target)
        intent.target.hp -= dmg
        applyStatusEffects(actor, intent.target, intent.skill)
        applyStatusTickAll(state)          # burn/poison/regen per turn
        log.append(snapshot(state, actor, intent, dmg))
        removeDead(turnOrder)

    result = state.playerAlive() ? "win" : "lose"
    rewards = result=="win" ? computeRewards(state) : null
    return { battleId, result, log, rewards, seed }
```

**Catatan keamanan:** `playerIntents` hanya berisi skill id + target id. `rng` di-seed server ⇒ hasil deterministik & replayable (FR-CMB-07). Enemy AI berjalan di server (tidak dikirim klien).

---

## 18. Konfigurasi & Constant (Appendix)

| Key | Nilai (MVP) | Sumber |
|---|---|---|
| IDLE_TICK_SEC | 5 | CDP |
| IDLE_OFFLINE_CAP_HRS | 24 | CDP |
| PARTY_MAX | 5 | CDP |
| GACHA_RATES | C0.6/R0.3/E0.09/L0.01 | CDP |
| PITY_GUARANTEE_EVERY | 10 | CDP |
| HARD_PITY | 90 | CDP |
| SOFT_PITY_START | 75 | CDP |
| ELO_START | 1000 | CDP |
| ELO_K | 32 | asumsi |
| ELEMENT_MULT_STRONG | 1.5 | CDP |
| ELEMENT_MULT_WEAK | 0.75 | CDP |
| MARKET_FEE | 0.05 | asumsi |
| LOGIN_FAIL_LOCK | 5 / 15m | FR-AUTH-04 |
| JWT_ACCESS_TTL_SEC | 900 | SEC-05 |
| HERO_LEVEL_CAP | 60 | asumsi OQ-01 |
| REFRESH_TOKEN_TTL_DAYS | 30 | asumsi |

---

## 20. Deployment & Operational Runbook (Draft)

### 20.1 Topology MVP
- 1 app node (NestJS, bisa replica bila perlu) + 1 PostgreSQL + 1 Redis + 1 BullMQ worker.
- CDN untuk aset PWA (hash-filename, immutable cache).
- Backup PG harian + WAL archive; Redis persistence AOF.

### 20.2 CI/CD
| Stage | Aksi |
|---|---|
| Lint/Type | ESLint + tsc strict. |
| Unit | Jest/Vitest (formula, validasi). |
| Integration | Testcontainers PG+Redis. |
| Security | npm audit + OWASP ZAP baseline. |
| Build | Docker image, tag semantic. |
| Deploy | Rolling update; migrasi DB via tool (Flyway/Prisma migrate) sebelum app. |

### 20.3 On-call & Alert
- Alert latency p99 > 300ms, error rate > 1%, payment failure > 0.5%, idle claim backlog.
- Rollback: image tag sebelumnya + `pg_restore` bila migrasi bermasalah (migrasi dirancang reversible).

### 20.4 Game Economy Guardrails (Operasional)
- Monitor inflasi: Gold per DAU harian; bila > 2x baseline → review idle rate / drop.
- Monitor gacha: distribusi aktual vs rate (drift alert).
- Monitor marketplace: GMV, fee revenue, anomali harga.

---

## 19. Referensi & Sumber

- Canonical Design Parameters (CDP) — parameter kanonik game ini.
- OWASP Top 10 (2021) — baseline keamanan (§3.2.1).
- GDPR / PDPA — panduan privacy (§3.3).
- PostgreSQL docs — ACID ledger, JSONB.
- BullMQ docs — queue/scheduler.
- vite-plugin-pwa — service worker & manifest.

---

**Akhir dokumen SRS v1.0 — Konsisten dengan CDP. Semua angka mengikuti parameter kanonik; asumsi & open questions dicatat eksplisit.**

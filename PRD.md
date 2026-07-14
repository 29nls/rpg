# PRD — Fantasy Turn-Based Idle RPG (Web + PWA)

> **Status:** Draft v1.0 — Untuk review internal (Product / Engineering / Design / Ops)
> **Pemilik Dokumen:** Product Analyst (Analis Produk)
> **Tanggal:** 2026-07-14
> **Lingkup:** MVP hingga LiveOps awal
> **Bahasa:** Indonesia (seluruh dokumen)

> **Catatan Konsistensi (CDP — Canonical Design Parameters):** Semua angka, nama sistem, dan batasan dalam dokumen ini mengikat pada parameter desain kanonik yang disepakati lintas dokumen (lihat Lampiran A). Apabila terdapat konflik antara bagian mana pun dengan CDP, maka **CDP menang**.

---

## Daftar Isi

1. Ringkasan Produk
2. Pilar Gameplay
3. Core Loop & Meta Loop
4. Fitur MVP vs Post-MVP
5. User Stories + Acceptance Criteria
6. Balancing & Economy
7. Monetisasi
8. KPI / Metrik
9. Risiko, Asumsi, Dependensi
10. Rencana Rilis (Milestone)
- Lampiran A — Canonical Design Parameters (Referensi)
- Lampiran B — Glosarium
- Lampiran C — Definisi Status & Elemen
- Lampiran D — Skema Database (Konsep Tingkat Tinggi)

---

# 1. Ringkasan Produk

## 1.1 Visi

Membangun RPG fantasi berbasis *turn-based* yang memadukan **kedalaman strategi** (formasi party, elemen, status effect, sinergi hero) dengan **kenyamanan idle** (perputaran konten otomatis + imbalan offline). Game diakses lewat **web desktop (desktop-first)** dengan progres PWA yang ringan, sehingga pemain dapat bermain di browser tanpa instalasi toko aplikasi, namun tetap mendapatkan nilai "buka tutup" (session pendek maupun panjang).

Tujuan jangka panjang: ekosistem yang **adil, teraudit, dan berkelanjutan** — bukan game *pay-to-win* ekstrem — dengan keamanan server-otoritatif dan alur data pemain yang patuh privasi (GDPR/PDPA-style).

## 1.2 Value Proposition

| Segmen | Masalah Saat Ini | Solusi Kami | Nilai Utama |
|--------|------------------|-------------|-------------|
| **Pemain Kasual** | Game RPG berat butuh waktu lama & skill tinggi | Idle accrual + auto-battle + offline rewards (cap 24 jam) | Progres tanpa *grind* wajib; main 5–15 menit/hari tetap有意义的 |
| **Pemain Hardcore** | Idle game terlalu dangkal | Wave-based combat dalam, elemen/status, gacha pity, PvP ladder, guild/raid | Kedalaman strategi & target jangka panjang |
| **Operator/Studio** | Risiko cheat & ekonomi hancur | Server-authoritative, audit trail, admin panel, anti-inflasi | Ekonomi stabil & biaya operasi terkendali |

*Catatan: "meaningful" di atas sengaja tidak diterjemahkan untuk menjaga makna "tetap terasa bermakna/bermanfaat". Tidak ada klaim bahwa progres kasual setara progres hardcore — keduanya berbeda laju, bukan kesetaraan hasil.*

## 1.3 Target Pengguna

| Tipe Pengguna | Profil | Motivasi Utama | Pola Main |
|---------------|--------|----------------|-----------|
| Kasual | 20–40 th, pekerja, sesi pendek | Koleksi, progres santai, "check-in" | 1–3 sesi/hari, <15 menit |
| Hardcore | 18–35 th, gamer berpengalaman | Optimasi meta, PvP rank, 100% completion | Sesi panjang, tracking spreadsheet |
| Admin / GM | Tim operasional studio | Stabilitas, moderasi, event, audit | Web admin panel, on-demand |

## 1.4 Platform & Batasan Teknis

- **Desktop-first web**: UI dioptimalkan untuk layar lebar (≥1024px). Responsif ke mobile *bonus*, bukan target utama MVP.
- **PWA**: *service worker* + Cache API untuk asset statis (HTML/JS/CSS/sprite); IndexedDB untuk antrean klik idle & state lokal saat offline.
- **Offline**: terbatas. Klik idle & antrean disimpan lokal; sinkronisasi saat *reconnect*. State otoritatif tetap di server.
- **Auth**: email/password (MVP). Sosial login = Post-MVP (lihat §4).
- **Skala awal**: 10–50 DAU pada MVP; arsitektur dirancang scalable ke ribuan DAU tanpa *rewrite* besar.

## 1.5 Asumsi Produk (Eksplisit)

- A1: Server adalah otoritas tunggal untuk currency, drops, gacha, dan hasil battle. Client hanya mengirim *intent* (aksi) dan menerima *state* hasil.
- A2: Tidak ada klaim pendapatan nyata pada MVP; monetisasi mulai dari *soft-launch* terbatas.
- A3: Bahasa produk = Indonesia untuk UI & kebijakan; kode & log tetap Inggris.
- A4: Tidak ada fitur *real-money trading* (RMT) eksternal; marketplace dalam-game menggunakan currency virtual semata.
- A5: Untuk MVP, "Mythic" (M) **tidak tersedia** — hanya C/R/E/L. M masuk via banner kolaborasi/anniversary (Post-MVP).

---

# 2. Pilar Gameplay

Tiga pilar utama membentuk identitas game. Masing-masing memiliki sistem pendukung yang dijelaskan di sub-bagian.

## 2.1 Pilar A — Turn-Based + Wave-Based Combat

### 2.1.1 Konsep Dasar
Pertempuran terjadi dalam *stage* yang terdiri dari beberapa *wave*. Setiap wave berisi musuh (1–N). Player mengontrol **party maksimal 5 hero** (CDP). Giliran (turn) ditentukan oleh **kecepatan (speed)**: hero/musuh dengan speed tinggi mendapat giliran lebih dulu; tie-break deterministik (ID entitas naik) untuk menghindari ambiguitas.

### 2.1.2 Resource & Skill
Setiap hero memiliki:
- **HP** (health points)
- **Mana / Energy** (sumber daya skill; aturan mana vs energy ditentukan per desain skill — CDP sebut "mana/energy")
- Skill: *basic attack* (tanpa/biaya rendah) + 1–3 *active skill* (butuh resource) + opsional *passive*.

### 2.1.3 Elemen & Affinity (CDP)
Enam elemen: **Fire, Water, Earth, Wind, Light, Dark**.
- Advantage → multiplier **1.5x**
- Disadvantage → multiplier **0.75x**

Asumsi peta affinity (eksplisit; belum final, dapat diubah asal konsisten CDP):
| Elemen | Kuat Lawan (1.5x) | Lemah Lawan (0.75x) |
|--------|------------------|---------------------|
| Fire | Wind, Earth | Water, Light |
| Water | Fire, Earth | Wind, Dark |
| Earth | Fire, Lightning→*Wind* | Water, Light |
| Wind | Water, Earth | Fire, Dark |
| Light | Dark | — |
| Dark | Light | — |

> **Asumsi A6 (eksplisit):** Tabel di atas adalah *contoh* peta elemen untuk MVP dan dapat direvisi oleh desain gameplay. Yang mengikat CDP hanya: 6 elemen, multiplier 1.5x / 0.75x. Light↔Dark adalah hubungan mutual-advantage (saling 1.5x) sebagai pengecualian desain — ini *asumsi*, bukan CDP.

### 2.1.4 Status Effect (CDP)
Delapan status effect standar:

| Status | Efek (konseptual) | Durasi | Stacking |
|--------|-------------------|--------|----------|
| Burn | Damage DoT per tick | 2–3 turn | Ya (cap) |
| Freeze | Skip turn / -speed | 1–2 turn | Tidak |
| Poison | Damage DoT (% HP) | 3 turn | Ya |
| Stun | Skip turn | 1 turn | Tidak |
| AtkDown | -ATK % | 2 turn | Ya (cap) |
| DefDown | -DEF % | 2 turn | Ya (cap) |
| Shield | Absorb damage | hingga hancur/2 turn | Ya |
| Regen | Heal per tick | 2–3 turn | Ya |

*Catatan: angka durasi di atas konseptual; nilai final di *balance pass* (§6). CDP hanya mewajibkan keberadaan 8 status tersebut.*

### 2.1.5 Auto-Battle & Strategi
- Auto-battle default ON (idle-friendly). Player dapat ganti target/skill manual (hardcore).
- Preset "AI behavior" per hero (prioritaskan heal/aoe/burst) — fitur MVP ringan, diperluas Post-MVP.

## 2.2 Pilar B — Idle Real-Time + Offline Rewards

### 2.2.1 Real-Time Tick (CDP)
- Tick setiap **5 detik** (server waktu otoritatif untuk akumulasi resmi; client menampilkan estimasi lokal).
- Saat online, stage idle berjalan otomatis; loot masuk ke *inventory* (via server).

### 2.2.2 Offline Accrual (CDP)
- Maksimal akumulasi **24 jam**.
- Rumus: `accrual = f(base_stage_rate, bonus)`, dihitung dari `(server_now - last_seen)` di-clamp ke `[0, 24h]`.
- `last_seen` diperbarui tiap tick online / tiap aksi.
- Saat *claim*, server menghitung ulang berdasarkan `server_now`, bukan waktu client (anti-manipulasi).

### 2.2.3 Trade-off Idle
- **Pro:** retensi kasual tinggi, session fleksibel.
- **Con:** berisiko inflasi economy (sink wajib, lihat §6). Offline cap mencegah "hoarding" tak terbatas.

## 2.3 Pilar C — Progression + Gacha + Economy + Trading

### 2.3.1 Progression
- Hero level/ascension, gear upgrade, material sink.
- Stage progression (PvE) sebagai sumber utama loot & gold.

### 2.3.2 Gacha (CDP)
- Banner standar: C 60% / R 30% / E 9% / L 1%.
- Pity: guaranteed R+ tiap 10-pull; hard pity L di pull ke-90; soft pity mulai pull ke-75 (rate L naik tiap pull hingga 90). Pity counter **per banner**.
- Auditability: setiap pull dicatat (seed/result) untuk transparansi & dispute.

### 2.3.3 Economy & Trading
- Currency: Gold (soft), Gems (hard/premium), Event Tokens.
- Marketplace dasar (MVP): trade item antar pemain dengan **market fee 5%** (CDP asumsi etis; lihat §6).
- Trading terbatas pada kategori tertentu (Gear/Material) — Consumable tidak tradeable di MVP (anti-exploit).

---

# 3. Core Loop & Meta Loop

## 3.1 Core Loop (Harian)

```
[Onboarding] ──▶ [Pilih Party ≤5] ──▶ [Battle Waves (PvE)]
                                               │
                                               ▼
                                        [Loot Drop] ──┐
                                               │        │
                                               ▼        │
                                     [Idle Tick 5s]     │
                                               │        │
                                  ┌────────────┴────────┘
                                  ▼
                         [Offline Accrual ≤24h]
                                  │
                                  ▼
                            [Claim Rewards]
                                  │
                                  ▼
                    [Upgrade Hero / Gear / Summon]
                                  │
                                  ▼
                   [Guild / Raid / PvP Ladder] ◀──(opsional)
                                  │
                                  └──────────▶ [Repeat] ──▶ (Meta Loop)
```

**Penjelasan alur:**
1. Pemain masuk (onboarding singkat untuk akun baru).
2. Susun party (maks 5 hero).
3. Bertarung wave PvE; dapat loot.
4. Saat idle/online, tick 5 detik akumulasi reward.
5. Saat offline, accrual dihitung server (cap 24 jam).
6. Pemain *claim* reward.
7. Gunakan reward untuk upgrade / summon (gacha).
8. Aktivitas sosial (guild/raid/pvp) memberi loop tambahan.
9. Ulangi; progres panjang masuk Meta Loop.

## 3.2 Meta Loop (Jangka Panjang)

```
[Stage Progression] ──▶ [Unlock Hero/Rarity] ──▶ [Gacha Pity Collect]
                                                        │
                                                        ▼
                                          [Gear Optimization] ──▶ [Raid Tier Up]
                                                        │
                                                        ▼
                                          [PvP Ladder Climb] ──▶ [Guild Rank]
                                                        │
                                                        └──▶ [Event/Anniversary (Mythic)]
                                                                  │
                                                                  └──▶ (Loop balik ke progres)
```

Meta loop memberi tujuan di luar hari-ke-hari: koleksi rarity, optimasi gear, naik ladder, event kolaborasi (sumber Mythic terbatas).

## 3.3 Diagram Sink/Source (Ekonomi — konsep)

```
SOURCES                        ───▶  PLAYER INVENTORY  ───▶  SINKS
- Stage clear (gold/loot)              (currency/items)        - Gear upgrade (gold)
- Idle/offline accrual (gold)                                     - Gacha (gems)
- Gacha (hero/gear)                                            - Market fee 5% (gold→burn)
- Marketplace sell (gold)                                      - Consumable use (item burn)
- PvP reward (gems/token)                                      - Ascension (material burn)
- Raid reward (token/material)                                 - Reset/CD (soft sink)
- Event (token)
```

---

# 4. Fitur MVP vs Post-MVP

Matriks fitur dengan justifikasi skala (tim kecil, 10–50 DAU).

## 4.1 Tabel Matriks Fitur

| # | Fitur | MVP | Post-MVP | Catatan / Trade-off |
|---|-------|-----|----------|---------------------|
| F1 | PvE Waves (combat turn-based) | ✅ | — | Inti; wave-based, ≤5 hero, elemen/status |
| F2 | Idle real-time (tick 5s) | ✅ | — | Online accrual |
| F3 | Offline rewards (cap 24h) | ✅ | — | Server-hitungan; clamp [0,24h] |
| F4 | Gacha standar (C/R/E/L) | ✅ | Mythic banner | Pity per banner; audit log |
| F5 | Inventory (Gear/Consumable/Material) | ✅ | — | 3 kategori CDP |
| F6 | Marketplace dasar | ✅ (terbatas) | Expanded (auction, bulk) | Fee 5%; Consumable tidak trade MVP |
| F7 | Guild + Raid ringan | ✅ (ringan) | Raid tier, guild war | MVP = 1 raid mingguan sederhana |
| F8 | PvP Ladder (ELO) | ✅ | Season, reward tier | ELO start 1000, K-factor |
| F9 | Admin Panel dasar | ✅ | Full GM tools | Ban, grant, refund, view logs |
| F10 | Auth email/password | ✅ | Sosial login, 2FA | GDPR/PDPA export/delete |
| F11 | Monetisasi dasar | ✅ (IAP gems) | Battle pass, subscription, ads-opt | Etis; spending limit |
| F12 | PWA offline cache | ✅ (statis+IndexedDB) | Push notif, background sync | Cache API + SW |
| F13 | Mythic rarity (M) | ❌ | ✅ | Hanya kolaborasi/anniversary |
| F14 | Social login | ❌ | ✅ | Google/Apple |
| F15 | Real-time multiplayer raid | ❌ | ✅ | MVP raid async/semi-realtime |
| F16 | Advanced analytics dashboard | ❌ | ✅ | Player behavior ML-lite |
| F17 | Trade chat / negotiation | ❌ | ✅ | MVP fix-price listing |

## 4.2 Kriteria "MVP Realistis"

- Fokus pada **loop bermain utuh** (battle → loot → idle → claim → upgrade → summon).
- Fitur sosial dibatasi (guild/raid/pvp ada tapi ringan).
- Marketplace ada tapi aman (item terbatas, fee, audit).
- Admin panel esensial untuk operasi (moderasi, refund, grant, log).
- Scalable: schema & API dirancang untuk perluasan tanpa rewrite.

---

# 5. User Stories + Acceptance Criteria

Format: **Sebagai [peran], saya ingin [aksi] agar [manfaat]** + Acceptance Criteria (Given/When/Then & checklist).

## 5.1 Pemain — PvE

**US-PVE-01:** Sebagai pemain, saya ingin memilih party hingga 5 hero agar saya dapat menyusun strategi sebelum battle.
- AC:
  - Given akun dengan ≥1 hero, When saya buka layar party, Then saya dapat menambah/mengurang hero (maks 5).
  - Given party berisi 5 hero, When saya coba tambah hero ke-6, Then sistem menolak & tampil pesan.
  - Given party kosong, When saya mulai battle, Then sistem menolak (butuh ≥1 hero).

**US-PVE-02:** Sebagai pemain, saya ingin battle berjalan otomatis (auto-battle) agar saya bisa idle.
- AC:
  - Given battle dimulai, When mode auto aktif, Then hero menyerang musuh tiap giliran by speed.
  - Given saya matikan auto, When giliran hero, Then saya dapat pilih skill/target manual.
  - Given musuh semua mati di wave, Then wave berikutnya mulai otomatis.

**US-PVE-03:** Sebagai pemain, saya ingin elemen & status effect memengaruhi damage agar strategi berasa.
- AC:
  - Given hero elemen advantage, When serang musuh, Then damage ×1.5 (server-hitung).
  - Given status Burn aktif, When tick status, Then musuh terima DoT & durasi berkurang.
  - Given Stun, When giliran musuh, Then musuh skip turn.

**US-PVE-04:** Sebagai pemain, saya ingin loot dari stage masuk inventory agar progres tersimpan.
- AC:
  - Given stage clear, When server proses drop, Then item masuk inventory (server-authoritative).
  - Given duplikasi item, When cek inventory, Then jumlah bertambah, tidak duplikat entry ilegal.

## 5.2 Pemain — Idle / Offline

**US-IDLE-01:** Sebagai pemain, saya ingin akumulasi idle saat online agar reward bertambah tiap 5 detik.
- AC:
  - Given online & stage idle aktif, When 5 detik lewat, Then server update accrual & UI refresh.
  - Given saya tutup tab, When buka lagi, Then accrual lanjut dari last_seen (bukan reset).

**US-IDLE-02:** Sebagai pemain, saya ingin claim offline reward saat kembali agar saya dapat resource.
- AC:
  - Given offline < 24 jam, When saya claim, Then reward = f(base, bonus, durasi).
  - Given offline ≥ 24 jam, When saya claim, Then reward di-cap di nilai 24 jam.
  - Given client kirim waktu palsu, When server hitung, Then pakai server_now (client diabaikan).

**US-IDLE-03:** Sebagai pemain offline (PWA), saya ingin klik idle tersimpan lokal agar tidak hilang.
- AC:
  - Given PWA offline, When saya klik, Then aksi masuk IndexedDB queue.
  - Given koneksi kembali, When sync, Then queue dikirim ke server & divalidasi.

## 5.3 Pemain — Gacha

**US-GAC-01:** Sebagai pemain, saya ingin melakukan pull gacha agar saya dapat hero/gear baru.
- AC:
  - Given cukup gems, When pull 1x, Then server tentukan rarity by rate (C60/R30/E9/L1).
  - Given pull ke-10, When banner belum kasih R+, Then pull ke-10 guarantee R+.
  - Given pull ke-90, When belum L, Then hard pity L di pull ke-90.
  - Given pull ke-75..89, When pull, Then rate L naik bertahap (soft pity).
  - Given banner beda, When cek pity, Then counter independen per banner.

**US-GAC-02:** Sebagai pemain, saya ingin melihat riwayat pull agar transparan.
- AC:
  - Given saya pull, When buka history, Then tampil rarity + timestamp + banner.
  - Given dispute, When admin cek, Then log auditable (seed/result tersimpan).

**US-GAC-03:** Sebagai pemain, saya ingin pity counter terlihat agar saya tahu progres.
- AC:
  - Given banner aktif, When buka banner, Then tampil "pull ke-X dari 90" & guarantee status.

## 5.4 Pemain — Inventory & Items

**US-INV-01:** Sebagai pemain, saya ingin item terkelompok (Gear/Consumable/Material) agar rapi.
- AC:
  - Given saya punya item 3 kategori, When buka inventory, Then filter per kategori tersedia.
  - Given Gear, When lihat detail, Then tampil rarity + stat.

**US-INV-02:** Sebagai pemain, saya ingin upgrade gear agar hero kuat.
- AC:
  - Given material cukup, When upgrade, Then stat naik & material terbakar (sink).
  - Given gold kurang, When upgrade, Then ditolak.

**US-INV-03:** Sebagai pemain, saya ingin pakai Consumable (potion/buff) di battle.
- AC:
  - Given HP rendah, When pakai potion, Then HP naik (server-validasi).
  - Given Consumable, When di marketplace, Then tidak bisa listing (MVP rule).

## 5.5 Pemain — Marketplace

**US-MKT-01:** Sebagai pemain, saya ingin jual item (Gear/Material) agar dapat gold.
- AC:
  - Given item tradeable, When listing, Then harga & fee 5% ditampilkan.
  - Given listing dibuat, When terjual, Then gold = harga − 5% fee masuk penjual.
  - Given item Consumable, When listing, Then ditolak.

**US-MKT-02:** Sebagai pemain, saya ingin beli item dari listing agar dapat item.
- AC:
  - Given listing aktif, When beli, Then gold pembeli berkurang, item masuk inventory.
  - Given gold kurang, When beli, Then ditolak.
  - Given item sudah terjual, When beli, Then status "sold" & ditolak.

**US-MKT-03:** Sebagai pemain, saya ingin aman dari duplikasi item.
- AC:
  - Given item di-lock saat listing, When transaksi, Then item tidak bisa dobel (server atomic).
  - Given race condition 2 pembeli, When server proses, Then hanya 1 sukses (lock).

## 5.6 Pemain — Guild / Raid

**US-GLD-01:** Sebagai pemain, saya ingin gabung guild agar main sosial.
- AC:
  - Given guild ada, When request join, Then masuk (atau pending bila invite-only).
  - Given saya leader, When kick/invite, Then aksi tercatat.

**US-GLD-02:** Sebagai pemain, saya ingin ikut raid mingguan ringan.
- AC:
  - Given raid aktif, When party masuk, Then boss muncul (HP bersama/giliran).
  - Given boss mati, When clear, Then reward token/material dibagi (server).
  - Given raid sudah di-clear minggu ini, When coba lagi, Then dibatasi (1x/minggu MVP).

## 5.7 Pemain — PvP Ladder

**US-PVP-01:** Sebagai pemain, saya ingin tantang lawan di ladder agar naik rank.
- AC:
  - Given ELO 1000 awal, When menang, Then ELO naik by K-factor.
  - Given kalah, When hitung, Then ELO turun.
  - Given tier threshold, When ELO capai, Then naik tier (Bronze→…).

**US-PVP-02:** Sebagai pemain, saya ingin battle PvP adil (server-authoritative).
- AC:
  - Given tantangan, When battle, Then hasil dihitung server (client hanya intent).
  - Given client kirim hasil curang, When server cek, Then ditolak & log.

## 5.8 Admin / GM

**US-ADM-01:** Sebagai admin, saya ingin melihat log transaksi agar audit.
- AC:
  - Given event (gacha/trade/refund), When buka log, Then detail lengkap (user, waktu, amount).
  - Given filter, When cari user, Then histori muncul.

**US-ADM-02:** Sebagai GM, saya ingin ban/mute pemain agar moderasi.
- AC:
  - Given pelanggaran, When ban, Then akses ditolak (server-enforced).
  - Given mute, When chat, Then diblokir (jika chat ada Post-MVP).

**US-ADM-03:** Sebagai admin, saya ingin grant/refund currency agar bantu player legit.
- AC:
  - Given request refund, When approve, Then gold/gems masuk (dengan log & alasan).
  - Given grant, When eksekusi, Then cap harian GM (anti-abuse internal).

**US-ADM-04:** Sebagai admin, saya ingin ekspor/hapus data akun (GDPR/PDPA).
- AC:
  - Given permintaan export, When proses, Then data akun dikembalikan (JSON).
  - Given permintaan delete, When konfirm, Then data anonymize/delete (dengan retensi log minimal wajib hukum).

---

# 6. Balancing & Economy

*Level tinggi — angka adalah konsep awal untuk MVP; final disesuaikan di balance pass berdasar playtest & metrik.*

## 6.1 Mata Uang

| Currency | Tipe | Sumber Utama | Kegunaan | Catatan |
|----------|------|--------------|----------|---------|
| Gold | Soft / in-game | Stage clear, idle, jual market | Upgrade gear, beli consumable, fee market | Inflasi-rawan → butuh sink kuat |
| Gems | Hard / premium | IAP, sebagian PvP/event | Gacha, skip, premium | Server-authoritative; anti-fraud |
| Event Tokens | Event | Raid, event | Tukar item event (batas waktu) | Burn saat event end |

## 6.2 Tabel Sink / Source

| Currency | Source (masuk) | Sink (keluar/burn) | Net Risk |
|----------|----------------|--------------------|----------|
| Gold | Stage clear, idle/offline, jual market, quest harian | Upgrade gear, market fee 5%, beli consumable, ascension material | Inflasi → cap harian idle, fee, burn event |
| Gems | IAP, reward PvP mingguan (kecil), event | Gacha pull, skip CD, premium bundle | Deflasi jika jarang kasih gratis → kasih trickle |
| Event Tokens | Raid, event login | Event shop (batas waktu) | Hangus saat event end (burn) |

## 6.3 Drop Table Konsep (Stage → Loot)

Asumsi (eksplisit, bukan CDP):
| Stage Tier | Gold (avg) | Material | Gear (raritas) | Consumable |
|------------|-----------|----------|---------------|------------|
| 1–10 (Easy) | 100–300 | Common x1–2 | C 80% / R 20% | Potion low |
| 11–30 (Med) | 300–800 | Common/Rare | R 70% / E 30% | Potion mid |
| 31–60 (Hard) | 800–2000 | Rare/Epic | E 85% / L 15% | Buff |
| 61+ (End) | 2000+ | Epic+ | E/L mix | Buff high |

> Trade-off: drop L dari stage jauh lebih rendah dari gacha (gacha L 1% tapi terjamin pity). Stage lebih ke material/gear C-R.

## 6.4 Gacha Pity / Guarantee (CDP)

| Mekanik | Nilai |
|---------|-------|
| Rate standar | C 60% / R 30% / E 9% / L 1% |
| Guarantee R+ | Tiap 10-pull (pull ke-10 pasti R+) |
| Hard pity L | Pull ke-90 (pasti L) |
| Soft pity L | Mulai pull ke-75, rate L naik tiap pull sampai 90 |
| Counter | Per banner (terpisah tiap banner) |
| Mythic (M) | Tidak di pool standar; hanya banner kolaborasi/anniversary |

**Contoh kurva soft pity (asumsi):** pull 75 → L ~3%, naik ~3–5% per pull, hingga 90 = 100% (hard pity). Angka persis di balance pass.

## 6.5 Biaya Trading / Fee

- **Market fee = 5%** dari harga jual (Gold). Fee masuk sink (burn, bukan ke pemain).
- Listing Consumable dilarang MVP (anti-exploit stamina/potion).
- Trade limit: maks N listing aktif per player (asumsi 10) untuk cegah spam.

## 6.6 Strategi Anti-Inflasi

1. **Sink kuat**: upgrade gear, ascension material, market fee, consumable use.
2. **Cap**: idle accrual 24 jam; daily quest cap; listing limit.
3. **Burn events**: event token hangus; event shop batas waktu.
4. **Gold sink premium**: cosmetic (Post-MVP) untuk tarik gold berlebih tanpa power.
5. **Monitoring**: admin panel alert jika GDP gold naik anomali.

---

# 7. Monetisasi

## 7.1 Prinsip Etis

- **Tidak pay-to-win ekstrem**: Gems mempercepat, bukan mengunci power inti. Hero L dapat dari gacha (grind) & pity, bukan beli langsung power absolut.
- Transparansi: rate gacha publik; riwayat pull ada.
- Perlindungan: spending limit, konfirmasi, kebijakan refund, parental (Post-MVP).

## 7.2 Tabel Produk Monetisasi

| Produk | Tipe | MVP? | Harga (konsep) | Posisi Etis |
|--------|------|------|----------------|-------------|
| Gem pack (S/M/L) | IAP | ✅ | $0.99–$49.99 | Ya — currency, bukan power lock |
| Battle Pass | Season | Post-MVP | $4.99 | Ya — cosmetic + bonus grind |
| Subscription (monthly) | Sub | Post-MVP | $9.99 | Ya — QoL (idle cap +? bonus kecil) |
| Ads (rewarded) | Ads | Post-MVP | — | Opsional, non-intrusive |
| Cosmetics | Vanity | Post-MVP | bervariasi | Ya — zero power |

## 7.3 Proteksi Pengguna

| Proteksi | Mekanik | Catatan |
|----------|---------|---------|
| Spending limit | Maks $/hari & $/bulan (default, bisa naik dgn konfirm) | Cegah impuls |
| Konfirmasi | 2-tap untuk pembelian ≥ threshold | Anti-accidental |
| Refund policy | 48 jam untuk IAP tidak terpakai (by platform) | Sesuai toko |
| Parental control | Post-MVP — pin & limit | Perlindungan anak |
| Gacha disclosure | Rate publik & history | Transparansi |

## 7.4 Trade-off Monetisasi

- **Pro:** revenue berkelanjutan, player puas (adil).
- **Con:** ARPU rendah vs P2W; butuh retensi tinggi. Untuk 10–50 DAU MVP, fokus retention dulu, monetisasi pelan.

---

# 8. KPI / Metrik

Target realistis untuk 10–50 DAU MVP. Cara ukur via event analytics (server-side utama).

## 8.1 Tabel Target Awal

| Metrik | Definisi | Target MVP (10–50 DAU) | Cara Ukur |
|--------|----------|------------------------|-----------|
| D1 Retention | % kembali hari-1 | 30–40% | Login event vs install date |
| D7 Retention | % kembali hari-7 | 15–25% | Cohort login |
| Session length | Rata2 menit/sesi | 8–15 mnt | Session start/end timestamp |
| Idle claim freq | Claim/hari per user | 1–3x | Claim event count |
| Conversion rate | % DAU beli IAP | 1–5% | IAP success / DAU |
| ARPDAU | Revenue / DAU | $0.05–$0.50 | Revenue harian / DAU |
| Churn | % berhenti main | <10% mingguan | Cohort drop |
| Trading volume | Gold traded/hari | 10k–100k (skala kecil) | Market tx log |
| PvP participation | % DAU PvP/minggu | 20–40% | PvP match event |
| Guild join rate | % DAU di guild | 30–60% | Guild membership |

## 8.2 Metrik Keamanan (Ops)

| Metrik | Target | Cara Ukur |
|--------|--------|-----------|
| Cheat attempt blocked | 100% terlog | Server validation reject count |
| Duplicate item incident | 0 | Inventory audit scan |
| Gacha dispute | <1% pull | Support ticket / log |
| Refund rate | <2% tx | Refund event |

---

# 9. Risiko, Asumsi, Dependensi

## 9.1 Tabel Risiko

| Risiko | Dampak | Kemungkinan | Mitigasi |
|--------|--------|-------------|----------|
| Exploit marketplace (duplikasi item) | Tinggi (ekonomi hancur) | Sedang | Server-authoritative atomic lock; item lock saat listing; audit scan harian |
| Duplikasi item via race condition | Tinggi | Sedang | DB transaction ACID; unique item instance ID |
| Fraud pembayaran (chargeback) | Menengah (revenue) | Sedang | Platform payment verified; flag anomali; spending limit |
| Abuse gacha (bot/flood) | Menengah | Rendah–Sedang | Rate-limit per akun; pity server-side; captcha Post-MVP |
| Cheat client (modifikasi hasil) | Tinggi | Sedang | Server hitung hasil; client hanya intent; anti-tamper ringan |
| Inflasi economy | Tinggi | Tinggi | Sink kuat, cap, burn event, monitoring admin |
| Churn tinggi (kasual bosen) | Menengah | Sedang | Event rutin, idle reward, meta goal |
| Privasi / compliance gagal | Tinggi (hukum) | Rendah | GDPR/PDPA: policy, export/delete, consent |
| Offline sync conflict | Menengah | Sedang | Server reconciliation; last-write-wins dengan validasi; cap 24h |
| Scale melampaui desain | Menengah | Rendah (MVP) | Arsitektur stateless API, DB index, caching |

## 9.2 Asumsi (Ringkasan)

- A1–A6 (lihat §1.5, §2.1.3, §2.3.3).
- A7: Server time sebagai otoritas waktu (NTP-sync).
- A8: 1 akun = 1 player (no multi-account abuse policy, butuh detect Post-MVP).
- A9: Marketplace hanya currency virtual (tidak RMT).

## 9.3 Dependensi

| Dependensi | Jenis | Dampak jika absen |
|------------|------|-------------------|
| Backend server (API + DB) | Tech | Blocker — semua state otoritatif |
| Payment gateway (IAP) | External | Monetisasi tertunda |
| Service worker / PWA infra | Tech | Offline terbatas gagal |
| Analytics pipeline | Tech | KPI tidak terukur |
| Legal: privacy policy | Compliance | Launch ilegal (GDPR/PDPA) |

---

# 10. Rencana Rilis (Milestone)

## 10.1 Tabel Milestone

| Tahap | Fokus | Estimasi | Exit Criteria |
|-------|-------|----------|---------------|
| Tahap 0 — Prototype | Combat tick, idle accrual, 1 banner gacha | 3–4 minggu | Loop battle→loot→idle jalan lokal |
| MVP | F1–F12 (matriks §4), 10–50 DAU | 8–12 minggu | PvE, idle+offline, gacha+audit, market aman, admin, auth, IAP dasar; KPI dasar tercapai |
| Beta | Invite 50–200, balance pass, anti-cheat | 4–6 minggu | Retensi D7 ≥15%; 0 duplikasi item; cheat blocked |
| Launch | Soft-launch region, monetisasi etis | 2–4 minggu | D1≥30%, conversion 1–5%, refund <2% |
| LiveOps | Event rutin, guild war, raid tier, Mythic banner | Berkelanjutan | Event bulanan; churn <10% |

## 10.2 Catatan Rilis

- MVP fokus **loop utuh & aman**, bukan fitur mewah.
- Setiap milestone punya exit criteria terukur (bukan "selesai coding").
- Post-MVP (Mythic, social login, battle pass, ads) masuk setelah launch stabil.

---

# Lampiran A — Canonical Design Parameters (Referensi Wajib)

| Parameter | Nilai |
|-----------|-------|
| Platform | Desktop-first web + PWA (SW + Cache API statis; IndexedDB offline queue) |
| Party | Maks 5 hero |
| Rarity | C, R, E, L; M (Mythic) terbatas kolaborasi/anniversary; MVP pool C/R/E/L |
| Gacha standar | C 60% / R 30% / E 9% / L 1% |
| Pity | Guarantee R+ tiap 10-pull; hard pity L pull 90; soft pity mulai 75; counter per banner |
| Idle | Tick 5 detik; offline cap 24 jam; accrual = f(base, bonus) dari (server_now − last_seen) clamp [0,24h] |
| Items | Gear (weapon/armor/accessory, rarity+stat), Consumable (potion/buff), Material (upgrade/ascension) |
| PvP | ELO (start 1000, K-factor), ladder tier |
| Currencies | Gold (soft), Gems (hard), Event Tokens |
| Combat | Turn by speed; mana/energy per hero; status: Burn/Freeze/Poison/Stun/AtkDown/DefDown/Shield/Regen; 6 elemen (Fire/Water/Earth/Wind/Light/Dark) affinity 1.5x / 0.75x |
| Skala awal | 10–50 DAU MVP, scalable |
| Auth | Email/password |
| Compliance | GDPR/PDPA-style (privacy policy, export/delete) |
| Security | Server-authoritative (currency, drops, gacha, battle) |

# Lampiran B — Glosarium

| Istilah | Arti |
|---------|------|
| DAU | Daily Active Users |
| MVP | Minimum Viable Product |
| PWA | Progressive Web App |
| Gacha | Sistem summon acak berbayar |
| Pity | Jaminan rarity setelah N pull |
| ELO | Sistem rating PvP |
| Sink | Saluran keluar currency (burn) |
| Source | Saluran masuk currency |
| RMT | Real-Money Trading (dilarang) |
| IndexedDB | Penyimpanan lokal browser (offline) |

# Lampiran C — Definisi Status & Elemen

(Lihat §2.1.3 & §2.1.4 untuk detail. CDP mengikat: 8 status, 6 elemen, multiplier 1.5/0.75.)

# Lampiran D — Skema Database (Konsep Tingkat Tinggi)

| Tabel | Field Kunci (konsep) |
|-------|----------------------|
| users | id, email(hash), password(hash), created_at, deleted_at |
| heroes | id, user_id, rarity, level, ascension, element, stats |
| parties | id, user_id, slot[5] hero_id |
| inventory_items | id (unique instance), user_id, category, rarity, stat, locked(bool) |
| gacha_banners | id, type, rates, pity_config |
| gacha_pity | user_id, banner_id, counter, guaranteed_flag |
| gacha_log | id, user_id, banner_id, rarity, result, timestamp, seed |
| currencies | user_id, gold, gems, event_tokens |
| market_listings | id, seller_id, item_instance_id, price, fee, status |
| pvp_matches | id, player_a, player_b, elo_delta, result |
| guilds | id, name, leader_id |
| raid_progress | guild_id, week, cleared(bool) |
| admin_logs | id, admin_id, action, target_user, reason, timestamp |
| idle_state | user_id, last_seen, accrual_pending |

> Semua tabel di atas konsep; final di desain DB. Prinsip: **item instance punya ID unik** untuk cegah duplikasi; **semua mutasi currency/item lewat transaksi ACID server-side**.

---

# 11. Arsitektur Sistem & Keamanan (Deep Dive)

Bagian ini memperluas §1.4 dan prinsip server-authoritative (CDP). Tujuannya memberi panduan teknis tingkat tinggi tanpa mengunci implementasi spesifik (stack bebas, asal prinsip dipenuhi).

## 11.1 Prinsip Server-Authoritative (Wajib CDP)

Semua state berikut **hanya** boleh dimutasi di server melalui transaksi tervalidasi:
- Currency (Gold / Gems / Event Tokens)
- Drops / loot hasil battle
- Hasil gacha (rarity + instance item)
- Hasil battle (HP akhir, status, menang/kalah)
- Marketplace listing & settlement
- PvP ELO & ladder

Client mengirim **intent** (contoh: `{"action":"use_skill","hero_id":12,"skill_id":3,"target_id":47}`), bukan **state** (tidak boleh kirim `{"hp_enemy":0}`). Server memvalidasi intent terhadap aturan & mengembalikan state baru.

### 11.1.1 Anti-Cheat — Daftar Validasi Server
| Cek | Penjelasan |
|-----|-----------|
| Authorization | Token sesi valid & milik user; aksi hanya pada resource own. |
| Affordability | Cukup currency/resource sebelum eksekusi. |
| Rule compliance | Skill ada, target valid, giliran benar (by speed). |
| Determinism | Seed battle & RNG gacha disimpan server; client tidak generate. |
| Rate-limit | Maks N aksi/detik per user (cegah flood/bot). |
| Replay check | (Post-MVP) bandingkan intent vs simulasi server. |

### 11.1.2 Offline / PWA Reconciliation
- Client offline: klik idle & aksi simpan di IndexedDB (queue).
- Saat reconnect: kirim queue berurutan ke server.
- Server: tolak aksi yang sudah invalid (item terjual, currency kurang) — **reconcile**, bukan blind-apply.
- `last_seen` hanya di-update server saat aksi valid diterima.
- Offline accrual selalu dihitung dari `server_now − last_seen` (client time diabaikan).

## 11.2 Service Worker & Caching (CDP)
| Aset | Strategi Cache |
|------|----------------|
| HTML shell / JS bundle / CSS | Cache-first (SW), update on new version |
| Sprite / audio statis | Cache-first, hashed filename |
| API response dinamis | Network-first, fallback ke cache stale (readonly) |
| Offline queue | IndexedDB (bukan Cache API) |

**Trade-off:** cache-first mempercepat load tapi butuh versioning hash agar update tak stale. Metadata game (rate, banner) selalu dari API (network-first) agar perubahan event langsung terasa.

## 11.3 Model Data Item (Anti-Duplikasi)
Setiap item = **instance unik** (`item_instance_id`, UUID). Inventory menyimpan instance, bukan counter generik.
- Trade: lock instance saat listing; settlement atomic (pembeli dapat instance, penjual gold − fee).
- Race condition: DB row-lock / transaction isolation → hanya 1 pembeli sukses.
- Audit: `gacha_log` & `market_listings` mencatat instance_id → traceability penuh.

## 11.4 Gacha Auditability (CDP)
- Tiap pull: simpan `{user_id, banner_id, pull_index, rng_seed, rarity, instance_id, timestamp}`.
- `rng_seed` di-generate server (CSRNG kriptografis atau HMAC-seeded deterministik).
- Admin dapat rekonstruksi hasil dari seed → bukti transparansi & dispute resolution.
- Pity counter disimpan per `user_id+banner_id`; tidak pernah dihitung client.

# 12. Desain Combat Mendalam

## 12.1 Algoritma Turn Order
1. Kumpulkan semua entitas (party + musuh) hidup.
2. Sort by `speed` desc; tie-break by `entity_id` asc (deterministik).
3. Freeze/Stun → entitas dilewati (skip turn) saat giliran.
4. Setelah semua dapat giliran 1x → putaran (round) baru; status DoT (Burn/Poison) tick di awal giliran entitas atau akhir round (pilih 1, konsisten).
5. Battle berakhir saat satu sisi 0 HP semua.

## 12.2 Rumus Damage (Konsep)
```
base_dmg   = ATK_hero * skill_multiplier - DEF_target * mitigation_factor
element_m  = 1.5  jika advantage
           = 0.75 jika disadvantage
           = 1.0  otherwise
crit_m     = crit ? crit_multiplier : 1.0
final_dmg  = max(1, round(base_dmg * element_m * crit_m * random_variance))
final_dmg  = apply Shield absorb (kurangi shield dulu, sisa ke HP)
```
> Angka `mitigation_factor`, `crit_multiplier`, `random_variance` konseptual; final di balance pass. Server hitung semua; client hanya tampil.

## 12.3 Matriks Interaksi Status (Asumsi)
| Diberikan | Burn | Freeze | Stun | Shield | Regen |
|-----------|------|--------|------|--------|-------|
| Burn | stack (cap) | ya | ya | ya | ya |
| Freeze | batalkan DoT? tidak | refresh durasi | ya | ya | ya |
| Stun | ya | ya | ya | ya | ya |
| Shield | ya | ya | ya | tambah absorb | ya |
| Regen | ya | ya | ya | ya | stack (cap) |

*Asumsi eksplisit: status sama jenis dapat stack dengan cap; jenis beda koeksis. Final di desain status.*

## 12.4 Contoh Simulasi Battle (ilustratif, bukan klaim metrik)
- Party: Hero Fire ATK 100 spd 120; Hero Water ATK 90 spd 110; vs Musuh Wind DEF 30 spd 100.
- Round 1: Fire (spd 120) serang Wind → advantage (Fire>Wind) ×1.5 → dmg tinggi. Water (spd 110) serang → Water<Wind disadvantage ×0.75.
- Musuh (spd 100) giliran terakhir → serang hero terlemah.
- Iterasi hingga satu sisi mati.

# 13. Desain Idle & Offline (Mendalam)

## 13.1 Tick Online (CDP)
- Interval 5 detik (server-driven untuk akumulasi resmi).
- Tiap tick: server tambah `accrual_pending` berdasar stage aktif & bonus.
- `last_seen = server_now` tiap tick valid.
- UI client poll/WS untuk refresh tampilan (estimasi lokal diperbolehkan asal tidak jadi otoritas).

## 13.2 Offline Accrual (CDP)
```
elapsed   = server_now - last_seen
elapsed   = clamp(elapsed, 0, 24h)
rate      = base_stage_rate * (1 + sum_bonus)
accrual   = rate * (elapsed dalam detik)   // atau per-menit, konsisten
gold_gain = floor(accrual)
```
- Cap 24 jam mencegah hoarding tak terbatas.
- Jika `last_seen` hilang/corrupt → anggap elapsed 0 (konservatif, anti-abuse).
- Event berjalan saat offline dihitung sama (rate mungkin berbeda per event).

## 13.3 Edge Cases Idle
| Skenario | Penanganan |
|----------|-----------|
| Client ubah jam sistem | Diabaikan; pakai server_now |
| Last_seen masa depan (clock skew) | elapsed negatif → clamp 0 |
| Stage di-reset (nerf) saat offline | Pakai rate stage saat claim (server) |
| Player banned saat offline | Accrual tidak bisa di-claim (akun locked) |

# 14. Marketplace Mendalam & Anti-Exploit

## 14.1 Aturan Listing (MVP)
- Item tradeable: Gear (semua rarity), Material (upgrade/ascension).
- Item tidak tradeable: Consumable, Event Tokens, bound items (hasil quest).
- Maks 10 listing aktif / player.
- Harga min/max (cegah 0-gold dump & inflasi listing).
- Fee 5% dipotong dari harga jual (sink).

## 14.2 Flow Transaksi (Atomic)
```
BUYER.klik_beli(listing_id)
  → Server: SELECT listing WHERE id=listing_id AND status='active' FOR UPDATE
  → Cek buyer.gold >= price
  → BEGIN TX
      buyer.gold   -= price
      seller.gold  += price - fee
      (fee burned / ke sink_account)
      item.instance_id.owner = buyer
      listing.status = 'sold'
      INSERT market_log
    COMMIT
  → Jika race: 1 sukses, lainnya rollback & "sold"
```

## 14.3 Deteksi Anomali Market (Ops)
| Sinyal | Tindakan |
|--------|----------|
| Harga jauh di bawah market avg (dump) | Flag review; mungkin duplikasi item |
| Volume trade 1 akun >> peer | Review multi-account / RMT |
| Item instance muncul >1 di ledger | ALERT kritis → freeze akun, audit |
| Fee sink tiba-tiba drop | Ekonomi stagnan → event burn |

# 15. Admin Panel — Operasional Detail

## 15.1 Modul Wajib (MVP)
| Modul | Fungsi |
|-------|--------|
| Dashboard | DAU, revenue, GDP gold, alert anomali |
| User Management | Cari, view, ban/mute, export/delete |
| Currency Ops | Grant/refund (dengan alasan + log), cap harian GM |
| Gacha Audit | Lihat pull per user, verifikasi seed |
| Market Ops | Lihat listing, cancel abuse, freeze item |
| Event Mgmt | Buat/stop event, set reward |
| Log Viewer | Semua mutasi currency/item terfilter |

## 15.2 Kontrol Internal GM (Anti-Abuse Staff)
- Setiap aksi GM wajib `reason` & tercatat di `admin_logs`.
- Grant currency dibatasi `cap_harian_gm` (mis. 100k gold / hari / GM).
- Aksi sensitif (delete akun, refund besar) butuh 2-GM approval (Post-MVP).
- Log admin immutable (append-only).

## 15.3 Runbook Insiden (Contoh)
| Insiden | Langkah |
|---------|---------|
| Duplikasi item terdeteksi | Freeze akun, rollback tx terkait, patch, kompensasi legit |
| Gacha rate salah | Stop banner, audit seed, refund/pull kompensasi |
| Chargeback spike | Suspend IAP region, review gateway, flag akun |
| Inflasi gold | Event burn, naik sink, cap idle sementara |

# 16. Compliance — GDPR / PDPA Style

## 16.1 Kewajiban (CDP)
- Privacy policy publik (ID + EN).
- Export data akun (JSON) atas permintaan.
- Delete / anonymize akun & data pribadi.
- Consent untuk analytics/marketing (opt-in).
- Retensi log: transaksi keuangan simpan periode wajib hukum; data pribadi dibatasi.

## 16.2 Data Map
| Data | Simpan? | Retensi | Export/Delete |
|------|---------|---------|---------------|
| Email | Ya (hash) | selama akun aktif | Delete |
| Password | Ya (bcrypt/argon2 hash) | selama akun | Delete |
| Progress game | Ya | selama akun | Export+Delete |
| Analytics event | Ya (anon) | 12–24 bulan | Delete (link ke akun) |
| Payment (pihak ke-3) | Gateway pegang | per kebijakan gateway | via gateway |

## 16.3 Proses Delete
1. User request (in-app / email).
2. Verifikasi identitas (login/email confirmation).
3. Anonymize `users` (hapus email, password, PII), set `deleted_at`.
4. Hapus inventory/currency hero (atau anonymize ke "deleted_user").
5. Pertahankan `admin_logs`/`market_log` minimal (tanpa PII) untuk audit hukum.
6. Konfirmasi ke user.

# 17. Desain Gacha & Pity (Mendalam)

## 17.1 Algoritma Pull (Server)
```
def pull(user, banner):
    pity = get_pity(user, banner)
    if pity.counter % 10 == 9:   # pull ke-10 (0-indexed 9)
        rarity = at_least(R)      # guarantee R+
    elif pity.counter >= 89:      # hard pity pull 90
        rarity = L
    else:
        rarity = roll_with_soft_pity(pity, banner.rates)  # soft mulai 75
    instance = create_item_instance(rarity)
    log_pull(user, banner, pity.counter, rarity, instance, seed)
    pity.counter += 1
    if rarity == L: pity.counter = 0
    return instance
```
> Pseudocode konseptual. `roll_with_soft_pity` naikkan rate L linier/exponential dari pull 75→90. Counter reset saat L didapat (atau tiap banner selesai — pilih 1, konsisten CDP: "counter per banner").

## 17.2 Tabel Rate Efektif (Estimasi Soft Pity)
| Pull ke- | Rate L efektif (asumsi) |
|----------|--------------------------|
| 1–74 | 1% |
| 75 | ~3% |
| 80 | ~15% |
| 85 | ~45% |
| 89 | ~85% |
| 90 | 100% (hard pity) |

*Angka asumsi untuk ilustrasi; final disesuaikan playtest. CDP mengikat: mulai 75, cap 90, hard pity di 90.*

## 17.3 Banner Types
| Banner | Pool | Pity | Catatan |
|--------|------|------|---------|
| Standar | C/R/E/L | per CDP | Selalu tersedia |
| Event Hero | 1 E/L featured + R/E/L | per CDP | Rate L mungkin naik ke featured |
| Kolaborasi/Anniversary | + M (Mythic) | khusus | M hanya di sini; Post-MVP |

# 18. PvP & Ladder Mendalam

## 18.1 ELO (CDP)
```
K = 32 (default; bisa beda per tier)
expected_A = 1 / (1 + 10^((R_B - R_A)/400))
R_A_new = R_A + K * (S_A - expected_A)   # S_A = 1 menang, 0 kalah, 0.5 draw
```
- Start 1000. Tier: Bronze(1000–1199), Silver(1200–1399), Gold(1400–1599), Platinum(1600+).
- Batas matchmaking: ±200 ELO (asumsi) agar adil.
- Hadiah mingguan: Gems kecil + Event Tokens by tier.

## 18.2 Fairness
- Battle PvP disimulasikan server (party lawan dari snapshot). Client hanya pilih formasi & intent.
- Anti-smurf: akun baru main vs sesama baru (MMR hidden awal).
- Replay (Post-MVP) untuk sengketa.

# 19. Progression & Ascension

## 19.1 Jalur Progression
| Tahap | Fokus | Sink Utama |
|-------|-------|-----------|
| 1–30 | Kenali combat, kumpul hero C/R | Gold (gear C/R), Consumable |
| 31–60 | Optimasi elemen, gacha E | Material ascension, Gems gacha |
| 61–90 | Raid, PvP, koleksi L | Material epic, Event Tokens |
| 90+ | Meta completion, Mythic (event) | Cosmetic, sink premium |

## 19.2 Ascension
- Hero naik batas level via material ascension (burn material).
- Setiap ascension +stat % & buka skill/passive.
- Cap ascension per rarity (asumsi: L max 5*, E 4*, R 3*, C 2*) — final di balance.

# 20. QA & Definition of Done (DoD)

## 20.1 DoD per Fitur
- [ ] Server-authoritative (client tidak bisa mutate state)
- [ ] Acceptance criteria §5 lulus (automated + manual)
- [ ] Log audit tersimpan untuk mutasi currency/item
- [ ] Edge case §13.3 / §14.3 ditangani
- [ ] Metrik §8 terinstrumen
- [ ] Privacy/delete path lolos (§16)

## 20.2 Test Matrix
| Jenis | Cakupan |
|-------|---------|
| Unit | Rumus damage, ELO, accrual clamp |
| Integration | Gacha+audit, market atomic, auth |
| Security | Cheat intent reject, duplikasi item, time tamper |
| Load | 50–500 DAU simulasi (scale check) |
| Compliance | Export/delete roundtrip |

## 20.3 RACI (MVP Team Kecil)
| Peran | Tanggung Jawab |
|-------|----------------|
| Product Analyst | PRD, prioritas, KPI |
| Backend Eng | Server-authoritative, DB, API |
| Frontend Eng | UI web, PWA, SW |
| Game Designer | Balance, elemen, status, gacha rate |
| DevOps/Ops | Deploy, monitoring, admin panel |
| (Ops/GM) | Moderasi, event, refund |

---

# 21. Spesifikasi Fitur MVP (Per-Fitur)

Setiap fitur MVP (§4 F1–F12) dijabarkan: tujuan, data, alur, acceptance tambahan.

## 21.1 F1 — PvE Waves
- **Tujuan:** loop combat inti.
- **Data:** `stages(id, tier, wave_count, enemy_ids[], recommended_power)`, `battles(session_id, state)`.
- **Alur:** pilih stage → spawn wave → auto/manual → clear → drop.
- **AC tambahan:** stage lock berurutan (harus clear N sebelum N+1, kecuali "replay" mode).

## 21.2 F2/F3 — Idle & Offline
- Lihat §13 lengkap. AC tambahan: indikator "akumulasi berjalan" di UI; notif (PWA) saat cap 24 jam tercapai.

## 21.3 F4 — Gacha
- Lihat §17. AC tambahan: konfirmasi biaya sebelum pull; "10-pull" bundle dengan diskon kecil (asumsi 9x harga, bukan 10x).

## 21.4 F5 — Inventory
- **Data:** `inventory_items(instance_id, user_id, category, rarity, stat_json, bound, locked)`.
- **AC tambahan:** sort/filter by rarity/kategori; stack Material by type (instance terpisah tapi tampil grup).

## 21.5 F6 — Marketplace
- Lihat §14. AC tambahan: search/filter listing; cancel listing (item kembali, tanpa fee refund ke seller untuk fee sudah dipotong? → asumsi: cancel gratis, fee hanya saat jual).

## 21.6 F7 — Guild & Raid
- **Data:** `guilds`, `guild_members`, `raid_progress`, `raid_boss`.
- **AC tambahan:** guild max 30 anggota (asumsi); raid 1x/minggu per guild; reward bagi rata atau by contribution (asumsi: by participation).

## 21.7 F8 — PvP Ladder
- Lihat §18. AC tambahan: cooldown antar match (asumsi 30 dtk); maks N match/hari untuk reward (asumsi 20).

## 21.8 F9 — Admin Panel
- Lihat §15. AC tambahan: login admin terpisah (role-based); session admin expire pendek.

## 21.9 F10 — Auth
- Email/password; verifikasi email wajib sebelum main penuh (atau grace 24 jam).
- Password min 8 char, hash argon2/bcrypt. Rate-limit login (lock 5 gagal).
- Lupa password via email reset token (expire 30 mnt).

## 21.10 F11 — Monetisasi Dasar
- IAP gem pack via payment gateway (App Store / Play / web checkout).
- Server verifikasi receipt sebelum grant gems (anti-fraud).
- Lihat §7 untuk proteksi.

## 21.11 F12 — PWA
- SW register, offline shell, IndexedDB queue.
- `manifest.json` (icon, name, standalone).
- Update SW otomatis saat bundle baru (skipWaiting + clients.claim terkontrol).

# 22. Onboarding & Tutorial

## 22.1 Alur Onboarding (MVP)
1. Register/login email.
2. Intro singkat (1 layar): "Kumpulkan hero, idle untuk reward, summon untuk rarity tinggi."
3. Tutorial battle: 1 hero vs 1 musuh (guided, auto).
4. Claim first idle reward (mock accrual 5 mnt).
5. Free 10-pull gacha (guarantee R+ per CDP).
6. Masuk stage 1, main bebas.

## 22.2 Prinsip Tutorial
- Tidak memaksa (skip-able setelah 1x).
- Kasual: auto-on; hardcore: hint manual control.
- Tidak ada paywall di tutorial.

# 23. Sistem Event

## 23.1 Tipe Event (MVP/Post)
| Event | MVP? | Reward | Burn? |
|-------|------|--------|-------|
| Login harian (7 hari) | ✅ | Gem kecil, material | — |
| Raid mingguan | ✅ | Token, material | Token hangus |
| Stage double-drop weekend | ✅ | Gold/loot x2 | — |
| Kolaborasi/Anniversary (Mythic) | ❌ | M rarity | Limited |
| Battle Pass season | ❌ | Cosmetic+grind | Season end |

## 23.2 Event Anti-Inflasi
- Event Token selalu batas waktu → burn otomatis.
- Double-drop dibatasi durasi pendek (weekend) agar tidak banjiri ekonomi.
- Admin dapat stop event darurat (§15.3).

# 24. Pipeline Konten & Stage

## 24.1 Stage Tier (Asumsi)
| Tier | Stage # | Power Reco | Fokus |
|------|---------|-----------|-------|
| Easy | 1–10 | 100–500 | Tutorial, C/R loot |
| Med | 11–30 | 500–2000 | R/E, elemen intro |
| Hard | 31–60 | 2000–8000 | E/L, status mastery |
| End | 61+ | 8000+ | Meta, raid prep |

## 24.2 Proses Tambah Konten
1. Game Designer set enemy/stage di config.
2. Balance pass (simulasi).
3. Server deploy config (tanpa client update untuk data).
4. A/B test drop rate kecil (Post-MVP).

# 25. Lokalisasi & Aksesibilitas

## 25.1 Lokalisasi
- UI & policy: Indonesia (utama), English (dual).
- String di resource file (bukan hardcode).
- Angka currency format lokal (Rp / $ tampil sebagai "Gem" abstrak).

## 25.2 Aksesibilitas (Dasar MVP)
- Kontras warna minimal WCAG AA.
- Font dapat diperbesar (PWA/desktop).
- Tidak mengandalkan warna saja untuk status (ikon + teks).
- Keyboard navigable untuk aksi utama.

# 26. Penanganan Error & Kode Error (Konsep)

| Kode | Arti | Respons User |
|------|------|--------------|
| E001 | Unauthorized | Redirect login |
| E002 | Insufficient currency | Pesan + sarankan grind/top-up |
| E003 | Invalid intent (cheat) | Tolak, log; jika berulang → review |
| E004 | Item locked (listing) | "Sedang dijual" |
| E005 | Rate limit | "Tunggu sebentar" |
| E006 | Offline queue full | Sync dulu / purge lama |
| E007 | Banner expired | Tidak bisa pull |
| E008 | Maintenance | Layar maintenance |
| E009 | Duplicate tx | Idempotency: abaikan duplikat |
| E010 | Validation fail | Tampil field error |

**Idempotency:** tiap aksi punya `client_req_id`; server dedupe → cegah double-spend saat retry/offline.

# 27. Kontrak API (Konsep Tingkat Tinggi)

Semua response: `{ok: bool, data?, error?: {code, msg}, req_id}`. Semua mutasi butuh `Authorization: Bearer <token>`.

| Endpoint | Method | Body (intent) | Return |
|----------|--------|---------------|--------|
| /auth/login | POST | {email, pwd} | {token, user} |
| /auth/register | POST | {email, pwd} | {token} |
| /party/set | POST | {hero_ids[≤5]} | {party} |
| /battle/start | POST | {stage_id} | {battle_id, state} |
| /battle/action | POST | {battle_id, action} | {state} |
| /idle/claim | POST | {} | {rewards} |
| /gacha/pull | POST | {banner_id, count:1|10} | {results[]} |
| /inventory/list | GET | {category?} | {items[]} |
| /market/list | POST | {item_instance_id, price} | {listing} |
| /market/buy | POST | {listing_id} | {item, gold} |
| /pvp/match | POST | {} | {battle_id, opponent} |
| /guild/join | POST | {guild_id} | {guild} |
| /raid/start | POST | {guild_id} | {battle_id} |
| /admin/grant | POST | {user_id, currency, amount, reason} | {log} |
| /account/export | GET | {} | {data_url} |
| /account/delete | POST | {confirm} | {status} |

> Path & nama bisa berubah; prinsip: intent-only, server-authoritative, idempotent.

# 28. Kerangka Kebijakan Privasi & Legal (Outline)

- **Pengantar:** siapa pengontrol data, kontak DPO (opsional MVP).
- **Data dikumpul:** email, progress, analytics anonim, payment (pihak ke-3).
- **Dasar hukum:** consent (analytics), kontrak (gameplay), kewajiban hukum (log keuangan).
- **Hak user:** akses, koreksi, hapus, portabilitas (export), batalkan consent.
- **Retensi:** lihat §16.2.
- **Subprocessor:** payment gateway, hosting, analytics.
- **Anak:** usia min (asumsi 13+); parental control Post-MVP (§7.3).

## 28.1 Checklist Launch Legal
- [ ] Privacy policy live (ID+EN)
- [ ] Form export/delete fungsional
- [ ] Consent banner (opt-in analytics)
- [ ] Kontrak payment gateway (refund policy)
- [ ] ToS (terms of service)

---

# 29. Panduan Trade-off Ringkas (Resume Keputusan)

| Keputusan | Pilihan | Trade-off |
|-----------|---------|-----------|
| Idle cap 24 jam | Batasi akumulasi | Retensi kasual vsanti-inflasi ✓ cap dipilih |
| Market fee 5% | Sink moderat | Anti-inflasi vs hambat trade; 5% kompromi |
| Server-authoritative | Aman, latensi | Kompleksitas vs cheat-proof ✓ |
| MVP tanpa Mythic | C/R/E/L saja | Scope kecil vs hype; kolaborasi nanti |
| Email/password only | Sederhana | Onboarding lambat vs aman; sosial login nanti |
| Cache-first PWA | Cepat | Update stale vs load; hash versioning atasi |
| Monetisasi etis | IAP currency | ARPU rendah vs kepercayaan jangka panjang |

# 30. Open Questions (Belum Diputuskan — Butuh Input Desain/Bisnis)

| # | Pertanyaan | Dampak | Pemilik |
|---|-----------|--------|---------|
| Q1 | Peta elemen final (§2.1.3 contoh vs revisi)? | Balance combat | Game Designer |
| Q2 | Nilai pasti soft-pity curve (§17.2)? | Gacha feel | Game Designer |
| Q3 | Harga IAP & region (§7.2)? | Revenue & compliance pajak | Product/Biz |
| Q4 | Batas listing & MM ELO (§14.1, §18.1)? | Market & PvP feel | Game Designer |
| Q5 | Apakah consol login di MVP? | Onboarding | Product |
| Q6 | Threshold refund & spending limit default? | Proteksi user | Product/Legal |
| Q7 | Apakah battle pass di MVP atau pasca-launch? | Scope | Product |
| Q8 | Server time sync toleransi skew? | Idle fairness | Backend |

> Semua jawaban harus konsisten dengan CDP. Jika bertentangan, CDP menang.

---

# 31. User Stories Tambahan (Keamanan, Compliance, Aksesibilitas)

## 31.1 Keamanan
**US-SEC-01:** Sebagai sistem, saya ingin menolak intent tidak valid agar cheat cegah.
- AC: Given client kirim `hp_enemy:0`, When server validasi, Then ditolak (E003) & dicatat.
- AC: Given 3x gagal validasi/menit, When dari IP sama, Then rate-limit naik.

**US-SEC-02:** Sebagai sistem, saya ingin dedupe aksi via `client_req_id` agar double-spend cegah.
- AC: Given aksi sama dikirim 2x, When server proses, Then hanya 1 eksekusi; ke-2 idempotent.

**US-SEC-03:** Sebagai admin, saya ingin alert duplikasi item agar cepat tangani.
- AC: Given instance_id muncul >1 di ledger, When scan harian, Then ALERT kritis & freeze.

## 31.2 Compliance
**US-CMP-01:** Sebagai user, saya ingin export data agar portabilitas.
- AC: Given klik export, When proses, Then email/link JSON berisi profil+progress.
- AC: Given data besar, When export, Then async (status "processing" lalu ready).

**US-CMP-02:** Sebagai user, saya ingin hapus akun agar privasi.
- AC: Given konfirm delete, When eksekusi, Then PII anonymize & inventory purge (§16.3).

**US-CMP-03:** Sebagai user, saya ingin consent analytics agar pilihan dihormati.
- AC: Given opt-out, When event kirim, Then analytics PII tidak dikirim.

## 31.3 Aksesibilitas & UX
**US-ACC-01:** Sebagai user low-vision, saya ingin font besar agar baca.
- AC: Given set zoom, When UI render, Then layout tidak rusak (desktop).

**US-ACC-02:** Sebagai user, saya ingin status pakai ikon+teks agar tidak ada ambiguitas warna.
- AC: Given Burn aktif, When tampil, Then ikon api + label "Burn".

**US-UX-01:** Sebagai user, saya ingin feedback tiap aksi agar paham.
- AC: Given aksi sukses/gagal, When respon, Then toast jelas (bukan silent).

# 32. Contoh Simulasi Battle (Lebih Banyak)

## 32.1 Skenario: Counter-Elemen
- Party: Water (ATK 90, spd 110) vs Fire musuh (DEF 40, spd 100).
- Water>Fire = advantage ×1.5.
- Round1: Water serang → dmg tinggi; musuh Fire serang → normal.
- Musuh mati lebih cepat → efisiensi tinggi saat matchup benar.

## 32.2 Skenario: Status Synergy
- Hero A pasang Burn (DoT), Hero B pasang DefDown.
- Musuh dengan DefDown menerima lebih banyak; Burn tick tiap round.
- Hasil: burst + sustain damage tanpa heal → efisien di wave panjang.

## 32.3 Skenario: Shield + Regen (Tank)
- Tank pasang Shield (absorb) + Regen (heal).
- Musuh burst ke shield dulu; sisa di-heal Regen.
- Trade-off: shield habis di 2 turn → butuh rotasi skill.

# 33. FAQ (Pemain & Internal)

| Q | A |
|---|---|
| Apakah game pay-to-win? | Tidak ekstrem; Gems percepat, tapi L dapat dari grind+gacha pity. |
| Bisakah main offline? | Terbatas: klik idle tersimpan lokal; state resmi saat reconnect. |
| Apakah item bisa diduplikasi? | Tidak; instance unik + lock atomic (server). |
| Apa itu pity? | Jaminan rarity setelah N pull (lihat §17). |
| Bagaimana inflation dicegah? | Sink kuat, cap, burn event (§6.6). |
| Data saya aman? | Server-authoritative; export/delete tersedia (§16). |
| Apa itu Mythic? | Rarity M terbatas kolaborasi/anniversary, bukan pool standar. |
| Berapa party maks? | 5 hero (CDP). |

# 34. Seed Data MVP (Contoh)

## 34.1 Hero Awal (Contoh, bukan CDP)
| Nama | Rarity | Elemen | Role | Skill |
|------|--------|--------|------|-------|
| Kael | C | Fire | DPS | Fire Slash (ATK x1.2) |
| Mira | C | Water | Healer | Aqua Mend (heal) |
| Borin | R | Earth | Tank | Stone Guard (shield) |
| Lira | R | Wind | DPS | Gale (aoe) |
| Sol | E | Light | Support | Smite (burn+atk) |
| Noct | L | Dark | DPS | Void (burst) |

> Nama & stat contoh; final di game design. CDP hanya mewajibkan rarity C/R/E/L & elemen 6.

## 34.2 Stage Awal Contoh
| Stage | Musuh | Elemen | Reco Power |
|-------|-------|--------|-----------|
| 1 | Slime x2 | Earth | 100 |
| 2 | Slime x3 | Earth | 150 |
| 3 | Wolf x2 | Wind | 250 |
| 5 | Boss Golem | Earth | 500 |

# 35. Runbook Operasional Lengkap

## 35.1 Start-of-Day Checklist (Ops)
- [ ] Cek dashboard: DAU, GDP gold, anomaly alert.
- [ ] Cek pending refund/ban request.
- [ ] Cek banner aktif & rate benar.
- [ ] Cek market volume normal.

## 35.2 Incident: Duplikasi Item
1. ALERT terima (§14.3).
2. Freeze akun terkait (US-ADM-02).
3. Query `inventory_items` by instance_id → temukan duplikat.
4. Rollback tx (hapus instance ilegal, kembalikan gold pembeli asli).
5. Patch root cause (lock/transaction).
6. Kompensasi pemain legit (grant seimbang, log).
7. Post-mortem.

## 35.3 Incident: Gacha Rate Melenceng
1. Stop banner (admin).
2. Audit `gacha_log` vs `rng_seed` (§11.4).
3. Hitung selisih rate aktual vs CDP.
4. Refund/pull kompensasi proporsional.
5. Fix seed generator / config.
6. Re-enable banner.

# 36. Anggaran Teknis & Performa

## 36.1 Performance Budget (MVP)
| Metrik | Target |
|--------|--------|
| First load (cached) | < 1.5 dtk |
| First load (cold) | < 3 dtk |
| API p95 latency | < 300 ms |
| Battle tick UI | 5 dtk (CDP) |
| Crash-free session | > 99% |

## 36.2 Tech Budget / Stack (Bebas, Prinsip)
- Backend: stateless, horizontal-scale (CDP: scalable).
- DB: ACID (Postgres/ekivalen) untuk currency/item.
- Cache: Redis untuk session/idle hot-path (opsional).
- Frontend: framework apa pun; PWA compliant.
- Monitoring: error + KPI dashboard (§8, §15).

## 36.3 Skala & Cost (Asumsi)
| DAU | Beban | Catatan |
|-----|-------|---------|
| 10–50 (MVP) | 1 server kecil | Idle tick ringan |
| 500 | 2–3 server | Cache wajib |
| 5000+ | Cluster + DB replica | Auto-scale |

> Asumsi kapasitas kasar; ukur saat beta (§10).

# 37. Definisi Metrik Lanjutan (Instrumentasi)

| Metrik | Query (konsep) | Alat |
|--------|---------------|------|
| D1 Retention | cohort join hari D, login D+1 | Analytics |
| Idle claim rate | count(claim)/DAU | Event log |
| Gacha pull/user | sum(pull)/DAU | gacha_log |
| Market GMV | sum(price) market_log/hari | market_log |
| Cheat block | count(E003)/hari | error log |
| GDP gold | sum(gold) snapshot/hari | currencies |

# 38. Catatan Revisi Dokumen

| Versi | Tanggal | Perubahan |
|-------|---------|-----------|
| v1.0 | 2026-07-14 | Draft awal lengkap (§1–§38), konsisten CDP |

> Setiap revisi harus mempertahankan konsistensi CDP (Lampiran A). Konflik → CDP menang.

---

# 39. User Stories Pemain Lainnya

## 39.1 Progression & Ascension
**US-PRO-01:** Sebagai pemain, saya ingin naik level hero agar kuat.
- AC: Given cukup XP (dari battle), When level up, Then stat naik & cap level sesuai ascension.
- AC: Given cap level tercapai, When tanpa ascension, Then XP idle (tidak bisa naik).

**US-PRO-02:** Sebagai pemain, saya ingin ascension via material agar batas level naik.
- AC: Given material cukup, When ascension, Then material burn & max_level naik.
- AC: Given rarity L, When ascension ke 5*, Then skill/passive terbuka (asumsi).

**US-PRO-03:** Sebagai pemain, saya ingin upgrade gear stat agar optimasi.
- AC: Given gold+material, When upgrade, Then stat +% & sink terbakar (§6.5).
- AC: Given gear L, When upgrade melewati cap, Then ditolak (cap ascension gear).

## 39.2 Social & Notifikasi
**US-SOC-01:** Sebagai pemain guild, saya ingin notif raid buka agar ikut.
- AC: Given raid mingguan aktif, When notif (PWA), Then saya bisa buka & join.
- AC: Given saya offline, When PWA push (Post-MVP), Then notif muncul saat online.

**US-SOC-02:** Sebagai pemain, saya ingin lihat leaderboard PvP/guild agar motivasi.
- AC: Given ladder, When buka, Then rank & top player tampil (tanpa PII sensitif).

**US-SET-01:** Sebagai pemain, saya ingin atur notif & bahasa agar nyaman.
- AC: Given setelan, When simpan, Then preferensi tersimpan (server, bukan hanya lokal).
- AC: Given ganti bahasa, When reload, Then UI pakai bahasa pilihan.

# 40. Simulasi Ekonomi (Contoh Kasar)

Asumsi: 50 DAU, rata2 2 sesi/hari, idle cap 24 jam.

| Sumber Gold/hari (total) | Jumlah (estimasi) | Sink/hari (total) | Jumlah |
|--------------------------|-------------------|-------------------|--------|
| Stage clear (50 user × 500) | 25.000 | Upgrade gear | 15.000 |
| Idle cap (50 × 1.000) | 50.000 | Market fee 5% | 5.000 |
| Jual market | 20.000 | Consumable | 5.000 |
| **Total in** | **~95.000** | Ascension material | 10.000 |
| | | **Total out** | **~35.000** |

> Simulasi ilustratif; menunjukkan net inflow → butuh sink lebih (cosmetic Post-MVP, event burn) agar seimbang. Bukan klaim metrik nyata.

# 41. Register Risiko Diperluas

| Risiko | Dampak | Kemungkinan | Mitigasi Tambahan |
|--------|--------|-------------|-------------------|
| Multi-account (1 player many) | Ekonomi/rank abuse | Sedang | Device/email fingerprint (Post-MVP); guild limit |
| Time-skew abuse idle | Inflasi | Rendah | Clamp negatif→0; server time (§13.3) |
| Poison/DoT overflow | Balance | Rendah | Cap stack status (§12.3) |
| Banner misconfig rate | Trust loss | Rendah | Pre-launch audit + canary (§35.3) |
| Payment gateway down | Lost revenue | Rendah | Retry + manual grant (admin) |
| SW cache stale | Bug UI | Rendah | Hash versioning + skipWaiting |
| DB deadlock market | Latency | Rendah | Row-lock + retry backoff |
| Insider GM abuse | Trust/legal | Rendah | 2-GM approval; log immutable (§15.2) |
| Region block / censorship | Reach | Rendah | Compliance review per region |
| Botting auto-battle | Unfair | Sedang | Behavior heuristic + captcha (Post-MVP) |

# 42. Manajemen Perubahan Config Game

## 42.1 Prinsip
- Data gameplay (rate, stage, enemy) di **config server**, bukan hardcode client.
- Perubahan butuh version + review (Game Designer + Backend).
- Perubahan rate gacha wajib publikasi (transparansi CDP).

## 42.2 Alur Perubahan
1. Draft config (PR).
2. Review (balance pass + security).
3. Staging test (simulasi).
4. Deploy (tanpa downtime; hot-reload).
5. Monitor KPI/anomali (§8, §15).
6. Rollback siap jika anomali.

## 42.3 Versioning
- `config_version` di response API; client cek tiap load.
- Jika client versi tua → prompt reload (PWA update).

# 43. Prinsip Desain Antarmuka (UI)

- **Desktop-first:** layout kolom (party kiri, battle tengah, inventory kanan) ≥1024px.
- **Density:** info padat tapi terbaca (kasual & hardcore butuh beda view? → toggle "simple/advanced" Post-MVP).
- **Feedback:** animasi damage singkat; tidak menghalangi (non-blocking).
- **Idle prominent:** panel accrual selalu terlihat (inti value).
- **Consistency:** warna rarity konsisten (C abu, R biru, E ungu, L emas, M merah muda — asumsi).

# 44. Kesimpulan

PRD ini mendefinisikan RPG fantasi turn-based idle web+PWA dengan fokus **aman & beroperasi**. Seluruh angka mengikat CDP (Lampiran A). MVP realistis untuk tim kecil & 10–50 DAU, scalable ke ribuan. Keamanan (server-authoritative, gacha audit, marketplace anti-duplikasi) dan compliance (GDPR/PDPA) diperlakukan sebagai fitur inti, bukan tambahan. Monetisasi etis (currency, bukan power-lock) menjaga kepercayaan jangka panjang.

Langkah berikut: jawab Open Questions (§30), lalu masuk Tahap 0 (§10).

---

# 45. Contoh Hari Main Pemain (End-to-End)

**Profil:** Kasual, 1 sesi pagi (10 mnt), 1 sesi malam (10 mnt).

| Waktu | Aksi | Sistem |
|-------|------|--------|
| 08:00 | Buka game (PWA) | SW load cache; claim offline (cap 24j) |
| 08:01 | Claim gold idle | Server hitung (server_now − last_seen) |
| 08:02 | 10-pull gacha (gratis) | Guarantee R+ (CDP) |
| 08:05 | Stage 1–5 auto | Loot C/R, material |
| 08:10 | Tutup; idle jalan (server tick) | Accrual akumulasi |
| 20:00 | Buka lagi | Claim idle + jual gear R di market |
| 20:05 | Raid guild (mingguan) | Reward token/material |
| 20:10 | PvP 3 match | ELO naik; tier Silver |
| 20:12 | Upgrade gear, ascension | Sink gold/material |
| 20:15 | Tutup; idle lanjut | Loop berulang besok |

# 46. Indeks Tabel (Referensi Cepat)

| Tabel | Bagian | Isi |
|-------|--------|-----|
| Value Proposition | §1.2 | Nilai per segmen |
| Target User | §1.3 | Profil pemain/admin |
| Elemen Affinity | §2.1.3 | 1.5x/0.75x map (asumsi) |
| Status Effect | §2.1.4 | 8 status |
| Matriks Fitur | §4.1 | MVP vs Post-MVP |
| Sink/Source | §6.2 | Economy currency |
| Drop Table | §6.3 | Stage→loot |
| Gacha Pity | §6.4 / §17 | Rate & pity CDP |
| Monetisasi | §7.2 | Produk & etis |
| KPI | §8.1 | Target 10–50 DAU |
| Risiko | §9.1 / §41 | Risk register |
| Milestone | §10.1 | Rilis |
| CDP | Lampiran A | Parameter wajib |
| Glosarium | Lampiran B | Istilah |
| API Contract | §27 | Endpoint konsep |
| Error Code | §26 | E001–E010 |
| Open Questions | §30 | Belum diputus |
| Perf Budget | §36.1 | Target teknis |

# 47. Glosarium Ekstended

| Istilah | Arti |
|---------|------|
| Wave | Satu kelompok musuh dalam stage |
| Stage | Level/Peta dalam PvE |
| Banner | Event gacha tertentu (pool + pity) |
| Soft pity | Kenaikan rate L mulai pull 75 |
| Hard pity | Jaminan L di pull 90 |
| Instance item | Salinan item unik (anti-dup) |
| Sink | Saluran keluar currency (burn) |
| Source | Saluran masuk currency |
| ELO | Rating PvP (start 1000) |
| Idempotency | Cegah double-exec aksi sama |
| Reconcile | Serikatkan state offline→server |
| SW | Service Worker (PWA) |
| IndexedDB | Storage lokal browser (offline queue) |
| Cache API | Cache aset statis PWA |
| GDPR/PDPA | Regulasi privasi EU/SEA-style |
| RNG | Random number generator (server) |
| DoT | Damage over time (Burn/Poison) |
| AoE | Area of effect (serang banyak) |
| DPS | Damage per second (peran) |
| Tank | Hero tahan (shield/high DEF) |

# 48. Log Keputusan (Decision Log)

| Tgl | Keputusan | Alasan | Ref |
|-----|-----------|--------|-----|
| 2026-07-14 | MVP tanpa Mythic | Scope & anti-inflasi | §2.3.3 |
| 2026-07-14 | Market fee 5% | Sink moderat | §6.5 |
| 2026-07-14 | Server-authoritative wajib | Anti-cheat | CDP |
| 2026-07-14 | Idle cap 24j | Anti-hoard | §2.2.2 |
| 2026-07-14 | Monetisasi etis (currency) | Kepercayaan | §7.1 |

> Log ini hidup; update tiap keputusan desain penting. Selalu rujuk CDP bila konflik.

---

*— Akhir PRD v1.0 (dokumen lengkap §1–§48, konsisten CDP) —*

# SAVE DATA SCHEMA LENGKAP — Fantasy Turn-Based Idle RPG (Web + PWA)

> **Dokumen:** Save Data Schema (Penyimpanan Sisi Klien)
> **Engine:** Phaser 3 + TypeScript + Vite
> **Platform:** Desktop-first Web + PWA (installable, offline-capable)
> **Arsitektur:** Server-Authoritative (klien = cache & prediksi saja)
> **Versi Dokumen:** 1.0.0
> **Status:** Canonical / Mengikat
> **Bahasa:** Indonesia (seluruh isi)

---

## Daftar Isi

1. [Prinsip Penyimpanan & Kebenaran](#1-prinsip-penyimpanan--kebenaran)
2. [localStorage Schema](#2-localstorage-schema)
3. [IndexedDB Schema](#3-indexeddb-schema)
4. [Field Type Tables (Rinci per Store)](#4-field-type-tables-rinci-per-store)
5. [Versioning & Migration](#5-versioning--migration)
6. [Security](#6-security)
7. [Cleanup & Retensi](#7-cleanup--retensi)
8. [Sync Strategy](#8-sync-strategy)
9. [Anti-Cheat Notes](#9-anti-cheat-notes)
10. [Hubungan dengan Dokumen Lain](#10-hubungan-dengan-dokumen-lain)
11. [Appendix A — Asumsi Eksplisit](#11-appendix-a--asumsi-eksplisit)
12. [Appendix B — Contoh JSON Snapshot](#12-appendix-b--contoh-json-snapshot)
13. [Appendix C — Glosarium](#13-appendix-c--glosarium)

---

## 1. Prinsip Penyimpanan & Kebenaran

Dokumen ini mendefinisikan **hanya** skema penyimpanan sisi klien. Penyimpanan sisi klien bukan sumber kebenaran. Sumber kebenaran tunggal (Single Source of Truth / SSOT) berada di server.

### 1.1 Hierarki Kebenaran

```
[ SERVER ]  ===== SSOT (currency, drops, gacha, battle outcome, progression) =====
     ▲   │
     │   │  (1) Klien kirim aksi ber-idempotency-key
     │   ▼
[ CLIENT ] ===== CACHE + PREDICTSI + OFFLINE QUEUE =====
```

- **Server** memegang otoritas mutlak atas: jumlah currency (Gold/Gems/EventToken), hasil drop, hasil gacha, hasil pertarungan, progresi karakter, owned-items, dan segala nilai ekonomi.
- **Klien** menyimpan:
  - *Settings* & *flags* kecil (localStorage) — murni preferensi UI, tidak berdampak ekonomi.
  - *Cache snapshot* UI/offline-queue (IndexedDB) — agar game bisa diputar saat offline dan agar UI cepat.

### 1.2 Aturan Emas (Hard Rules)

| # | Aturan | Alasan |
|---|--------|--------|
| R1 | Klien **tidak pernah** otoritas atas currency/progression. | Anti-cheat; mencegah manipulasi nilai ekonomi. |
| R2 | Semua nilai ekonomi di klien = *display cache* dari server, bukan kebenaran. | Konsistensi lintas sesi/perangkat. |
| R3 | Setiap aksi ekonomi dikirim ke server dengan **idempotency key**. | Mencegah duplikasi saat retry/offline-queue flush. |
| R4 | Saat konflik, **server menang** (server-wins conflict resolution). | Deterministic; tidak ada last-write-wins di klien. |
| R5 | Access token **tidak** disentuh oleh JS (httpOnly cookie). | Mencegah pencurian token via XSS. |
| R6 | Data sensitif (currency, inventory) **tidak** disimpan di localStorage. | localStorage mudah dibaca/dimanipulasi. |
| R7 | Autosave snapshot di IndexedDB **boleh** berisi prediksi lokal, tapi harus dibedakan flag `isPredicted`. | Agar UI tidak menampilkan "kebenaran palsu" saat offline. |
| R8 | Tick idle = **5 detik**, idle cap = **24 jam**. | Batas akumulasi idle agar seimbang gameplay. |
| R9 | Tidak ada cryptography sisi klien yang dianggap rahasia. | Klien dapat dibongkar; enkripsi klien bukan keamanan. |
| R10 | Semua write ke storage di-throttle (lihat §8.4). | Mencegah thrashing IndexedDB/disk. |

### 1.3 Definisi "Sensitive" vs "Non-Sensitive"

- **Sensitive (JANGAN di localStorage, hanya server):** currency balance, inventory, owned heroes, gacha pity counter, battle results, purchase receipts, user PII (email), session identity.
- **Non-Sensitive (BOLEH di localStorage/IndexedDB):** volume audio, bahasa UI, keybind, tutorial-done flag, seen-intro flag, audio-unlocked flag, fcm token opt-in, draft UI, uiState terakhir.

### 1.4 Asumsi Dasar (lihat Appendix A untuk lengkap)

- Akses internet tersedia sebagian besar waktu; offline adalah kondisi transien, bukan mode utama.
- Salah satu akun = satu `playerId` (UUID v4 dari server).
- Klien hanya menyimpan snapshot milik akun yang sedang login.
- Multi-akun di satu browser: hanya 1 akun aktif dalam satu waktu (tidak ada account switcher persisten di v1).

---

## 2. localStorage Schema

`localStorage` digunakan **hanya** untuk preferensi UI kecil dan flag non-ekonomi. Semua key di-namespaced dengan prefix `rpg:` agar tidak tabrakan dengan aplikasi lain di origin yang sama.

### 2.1 Namespace

```
Prefix wajib:  rpg:
Contoh:        rpg:settings, rpg:flags, rpg:fcmOptIn
```

### 2.2 Tabel localStorage

| key | tipe | default | encrypted? | keterangan |
|-----|------|---------|------------|------------|
| `rpg:settings` | JSON object | lihat 2.3 | Tidak | Preferensi audio & UI. Tidak sensitif. |
| `rpg:flags` | JSON object | lihat 2.4 | Tidak | Flag progresi tutorial/intro/audio. Non-ekonomi. |
| `rpg:fcmOptIn` | boolean | `false` | Tidak | Persetujuan push notification (GDPR/PDPA opt-in). |
| `rpg:clientVersion` | string | `""` | Tidak | Versi build klien terakhir; dipakai deteksi stale cache. |
| `rpg:lastSyncAt` | number (epoch ms) | `0` | Tidak | Waktu sinkronisasi terakhir ke server (display saja). |
| `rpg:offlineMode` | boolean | `false` | Tidak | Flag apakah klien terakhir berada di mode offline. |
| `rpg:schemaVersion` | number | `1` | Tidak | Versi schema localStorage; lihat §5. |

> **CATATAN KEAMANAN:** Access token **TIDAK** disimpan di localStorage. Token ada di httpOnly cookie (server-set). Refresh token juga httpOnly. JS tidak pernah membaca token. (Lihat §6.)

### 2.3 `rpg:settings` (JSON object)

| field | tipe | default | encrypted? | keterangan |
|-------|------|---------|------------|------------|
| `masterVolume` | number (0.0–1.0) | `0.8` | Tidak | Volume master global. |
| `musicVolume` | number (0.0–1.0) | `0.6` | Tidak | Volume musik latar. |
| `sfxVolume` | number (0.0–1.0) | `0.7` | Tidak | Volume efek suara. |
| `uiVolume` | number (0.0–1.0) | `0.5` | Tidak | Volume SFX antarmuka (tombol, dll). |
| `language` | string enum | `"id"` | Tidak | `"id"` \| `"en"`. Lainnya fallback ke `"en"`. |
| `inputBindings` | JSON object | lihat 2.3.1 | Tidak | Pemetaan tombol/shortcut kustom. |

#### 2.3.1 `inputBindings` (JSON object)

| field | tipe | default | encrypted? | keterangan |
|-------|------|---------|------------|------------|
| `confirm` | string | `"Space"` | Tidak | Tombol konfirmasi default. |
| `cancel` | string | `"Esc"` | Tidak | Tombol batal default. |
| `autoBattle` | string | `"A"` | Tidak | Toggle auto-battle. |
| `openMenu` | string | `"M"` | Tidak | Buka menu utama. |
| `speedToggle` | string | `"F"` | Tidak | Toggle kecepatan idle. |
| `custom` | JSON object | `{}` | Tidak | Override kustom per-aksi; validasi di §6.4. |

> Batas: `inputBindings` maksimal 64 pasangan. Key hanya karakter `A-Z`, `0-9`, `Space`, `Esc`, `F1`–`F12`. (Validasi §6.4.)

### 2.4 `rpg:flags` (JSON object)

| field | tipe | default | encrypted? | keterangan |
|-------|------|---------|------------|------------|
| `tutorialDone` | boolean | `false` | Tidak | True bila tutorial utama selesai. |
| `seenIntro` | boolean | `false` | Tidak | True bila cutscene intro pernah diputar. |
| `audioUnlocked` | boolean | `false` | Tidak | True bila autoplay policy sudah di-unlock (klik pertama). |
| `seenGachaIntro` | boolean | `false` | Tidak | True bila intro gacha pernah dilihat. |
| `seenBattleTutorial` | boolean | `false` | Tidak | True bila tutorial battle pernah dilihat. |
| `dataOptInAnalytics` | boolean | `false` | Tidak | Persetujuan analytics (terpisah dari fcmOptIn). |

### 2.5 Contoh Isi localStorage (mentah)

```jsonc
// rpg:settings
{
  "masterVolume": 0.8,
  "musicVolume": 0.6,
  "sfxVolume": 0.7,
  "uiVolume": 0.5,
  "language": "id",
  "inputBindings": {
    "confirm": "Space",
    "cancel": "Esc",
    "autoBattle": "A",
    "openMenu": "M",
    "speedToggle": "F",
    "custom": {}
  }
}

// rpg:flags
{
  "tutorialDone": true,
  "seenIntro": true,
  "audioUnlocked": true,
  "seenGachaIntro": false,
  "seenBattleTutorial": true,
  "dataOptInAnalytics": false
}

// rpg:fcmOptIn
false

// rpg:clientVersion
"1.0.0-rc3"

// rpg:lastSyncAt
1718000000000

// rpg:offlineMode
false

// rpg:schemaVersion
1
```

### 2.6 Batas & Constraint localStorage

| Constraint | Nilai | Catatan |
|------------|-------|---------|
| Maks total ukuran | ~5 MB per origin | Praktik browser modern; tidak diandalkan. |
| Maks panjang 1 key | ≤ 256 char | Nama key pendek. |
| Maks panjang 1 value | ≤ 5 MB total | `rpg:settings` & `rpg:flags` sangat kecil (< 2 KB). |
| Sinkronisasi lintas tab | Ya (event `storage`) | Klien listen event untuk sync settings antar-tab. |
| Akses | Synchronous | Gunakan hanya data kecil; jangan simpan array besar. |

---

## 3. IndexedDB Schema

Database utama klien: **`rpg_client`**. Berisi object store untuk offline queue, autosave snapshot, drafts, dan asset cache metadata.

### 3.1 Spesifikasi Database

| properti | nilai |
|----------|-------|
| Nama DB | `rpg_client` |
| Versi DB | `1` (lihat §5 untuk migrasi) |
| Engine | IndexedDB API (via wrapper minimal; tidak perlu library besar) |
| Isolasi per akun | Key `playerId` pada `autosave_snapshot`; queue/drafts di-namespaced dengan `playerId` di `payload.meta`. |
| Wipe saat logout | `autosave_snapshot` untuk `playerId` dihapus (lihat §7.3). |

### 3.2 Object Stores Overview

| store | keyPath | tujuan | persistensi |
|-------|---------|--------|-------------|
| `offline_queue` | `id` (UUID) | Antrean aksi saat offline / saat request gagal. | Transien; di-flush ke server. |
| `autosave_snapshot` | `playerId` | Cache snapshot UI & prediksi terakhir. | Persist; dihapus saat logout. |
| `drafts` | `id` (UUID) | Draft UI (market listing, pesan, build). | Persist sampai di-submit/batal. |
| `asset_cache_meta` | `assetId` | Metadata cache aset (hash, expiry, ukuran). Opsional. | Cache; purge saat quota penuh. |

---

### 3.3 Store (a): `offline_queue`

Menyimpan aksi-aksi yang belum terkonfirmasi server. Setiap record adalah satu "intent" yang nanti di-flush secara idempoten ke endpoint `/sync`.

#### 3.3.1 Field Table — `offline_queue`

| field | tipe | nullable | constraint | index | keterangan |
|-------|------|----------|------------|-------|------------|
| `id` | string (UUID v4) | Tidak | Unique; PK | primary key (keyPath) | Idempotency key unik. |
| `type` | string enum | Tidak | ∈ aksi valid (§9) | `idx_type` | Jenis aksi (claimIdle, buyItem, summonGacha, dll). |
| `payload` | JSON object | Tidak | ≤ 8 KB | — | Body request ke server. Tidak mengandung currency. |
| `createdAt` | number (epoch ms) | Tidak | > 0 | `idx_createdAt` | Waktu enqueue. |
| `status` | string enum | Tidak | `pending`\|`inflight`\|`acked`\|`failed` | `idx_status` | Status pengiriman. |
| `retry` | number (int ≥ 0) | Tidak | default 0, ≤ `MAX_RETRY` (5) | — | Jumlah percobaan ulang. |
| `lastError` | string \| null | Ya | ≤ 512 char | — | Pesan error terakhir (tanpa secret). |
| `playerId` | string (UUID) | Tidak | ∈ akun aktif | `idx_player` | Pemilik aksi (namespacing). |
| `idempotencyKey` | string | Tidak | == `id` (cermin) | `idx_idem` | Key idempotensi server; cegah duplikasi. |
| `clientVersion` | string | Tidak | build version | — | Versi klien saat enqueue (audit). |
| `ttl` | number (epoch ms) | Ya | null = no expiry | — | Batas waktu aksi valid (mis. gacha 24j). |

#### 3.3.2 Index Definition — `offline_queue`

```sql
-- Pseudo-DDL (IndexedDB tidak pakai SQL; ini dokumentasi intent)
CREATE OBJECT STORE offline_queue (keyPath: "id", autoIncrement: false);
CREATE INDEX idx_type      ON offline_queue (type);
CREATE INDEX idx_createdAt ON offline_queue (createdAt);
CREATE INDEX idx_status    ON offline_queue (status);
CREATE INDEX idx_player    ON offline_queue (playerId);
CREATE INDEX idx_idem      ON offline_queue (idempotencyKey);
```

#### 3.3.3 `payload` Structure (by type)

Semua payload **hanya** berisi parameter aksi, BUKAN hasil/server-state.

| type | payload.minimal | keterangan |
|------|-----------------|------------|
| `claimIdle` | `{ stageId, seconds, ticket }` | `ticket` = server-issued idle claim token (anti-replay). |
| `buyItem` | `{ shopId, itemId, qty }` | Server validasi harga & saldo. |
| `sellItem` | `{ itemId, qty }` | Server validasi kepemilikan. |
| `summonGacha` | `{ bannerId, pulls }` | Server tentukan hasil (pity, rng). |
| `equipItem` | `{ heroId, slot, itemId }` | Server validasi ownership. |
| `startBattle` | `{ stageId, party }` | Party = array heroId; server tentukan hasil. |
| `upgradeHero` | `{ heroId, stat }` | Server validasi resource. |
| `useConsumable` | `{ itemId, targetId }` | Battle-scoped. |
| `marketList` | `{ itemId, qty, price }` | Draft listing market. |
| `collectMail` | `{ mailId }` | Klaim attachment mail. |
| `dailyCheckin` | `{ day }` | Server validasi streak. |

> Semua payload di atas **tidak** menyertakan jumlah currency. Server menghitung ulang dari SSOT.

---

### 3.4 Store (b): `autosave_snapshot`

Cache snapshot terakhir dari state UI & prediksi lokal, di-key oleh `playerId`. Ini yang membuat "game tetap tampil" saat offline dan saat reload cepat.

#### 3.4.1 Field Table — `autosave_snapshot`

| field | tipe | nullable | constraint | index | keterangan |
|-------|------|----------|------------|-------|------------|
| `playerId` | string (UUID) | Tidak | PK | primary key (keyPath) | Pemilik snapshot. |
| `uiState` | JSON object | Tidak | ≤ 64 KB | — | State UI terakhir (screen, scroll, filter). |
| `lastSeen` | JSON object | Tidak | lihat §4.4 | — | State machine terakhir (screen + timestamp). |
| `partyDraft` | JSON object \| null | Ya | ≤ 4 hero | — | Draft susunan party sebelum battle. |
| `version` | number | Tidak | == DB schema ver | — | Versi snapshot (migrasi). |
| `currencyCache` | JSON object | Tidak | mirror server (display only) | — | `{ gold, gems, eventToken }` — CACHE, bukan kebenaran. |
| `predictedDelta` | JSON object | Ya | flag `isPredicted` | — | Prediksi perubahan idle saat offline. |
| `savedAt` | number (epoch ms) | Tidak | > 0 | `idx_savedAt` | Waktu snapshot dibuat. |
| `isStale` | boolean | Tidak | default false | — | True bila server state lebih baru. |
| `clientVersion` | string | Tidak | build version | — | Versi klien pembuat snapshot. |

#### 3.4.2 Index Definition — `autosave_snapshot`

```sql
CREATE OBJECT STORE autosave_snapshot (keyPath: "playerId", autoIncrement: false);
CREATE INDEX idx_savedAt ON autosave_snapshot (savedAt);
```

#### 3.4.3 Contoh Record

```jsonc
{
  "playerId": "a1b2c3d4-0000-4000-8000-000000000001",
  "uiState": {
    "currentScreen": "town",
    "openPanels": ["inventory"],
    "selectedHeroId": "hero_042",
    "scrollPositions": { "inventory": 128 }
  },
  "lastSeen": {
    "state": "TOWN",
    "subState": "IDLE",
    "timestamp": 1718000000000
  },
  "partyDraft": {
    "slots": ["hero_042", "hero_017", "hero_008", null],
    "formation": "front-back"
  },
  "version": 1,
  "currencyCache": { "gold": 125000, "gems": 320, "eventToken": 12 },
  "predictedDelta": {
    "isPredicted": true,
    "goldPerHour": 5400,
    "sinceMs": 1717998200000
  },
  "savedAt": 1718000000000,
  "isStale": false,
  "clientVersion": "1.0.0-rc3"
}
```

> **PERINGATAN:** `currencyCache` dan `predictedDelta` adalah *display-only*. UI boleh menampilkannya, tapi logika ekonomi apa pun yang menggunakannya untuk keputusan gameplay **dilarang**. Keputusan harus menunggu konfirmasi server.

---

### 3.5 Store (c): `drafts`

Menyimpan draft UI yang belum di-submit (market listing, pesan guild, build/loadout). Bukan aksi ekonomi — hanya teks/parameter input pengguna.

#### 3.5.1 Field Table — `drafts`

| field | tipe | nullable | constraint | index | keterangan |
|-------|------|----------|------------|-------|------------|
| `id` | string (UUID) | Tidak | PK | primary key | ID draft. |
| `kind` | string enum | Tidak | `market`\|`message`\|`loadout`\|`note` | `idx_kind` | Jenis draft. |
| `playerId` | string (UUID) | Tidak | ∈ akun aktif | `idx_player` | Pemilik. |
| `data` | JSON object | Tidak | ≤ 16 KB | — | Isi draft (tergantung kind). |
| `createdAt` | number | Tidak | > 0 | `idx_createdAt` | Waktu dibuat. |
| `updatedAt` | number | Tidak | ≥ createdAt | — | Waktu terakhir diubah. |
| `ttl` | number \| null | Ya | null = persist | — | Kadaluarsa draft (mis. market 7 hari). |

#### 3.5.2 Index Definition — `drafts`

```sql
CREATE OBJECT STORE drafts (keyPath: "id", autoIncrement: false);
CREATE INDEX idx_kind      ON drafts (kind);
CREATE INDEX idx_player    ON drafts (playerId);
CREATE INDEX idx_createdAt ON drafts (createdAt);
```

---

### 3.6 Store (d): `asset_cache_meta` (Opsional)

Metadata aset yang di-cache (gambar sprite, audio, JSON stage) agar PWA cepat saat offline. Isi biner aset disimpan di Cache API (bukan IndexedDB); store ini hanya metadata.

#### 3.6.1 Field Table — `asset_cache_meta`

| field | tipe | nullable | constraint | index | keterangan |
|-------|------|----------|------------|-------|------------|
| `assetId` | string | Tidak | PK | primary key | ID aset (hash-aware path). |
| `url` | string (URL) | Tidak | valid URL | `idx_url` | URL asal aset. |
| `hash` | string (sha256 hex) | Tidak | 64 char | — | Content hash untuk validasi integritas. |
| `size` | number (bytes) | Tidak | ≥ 0 | `idx_size` | Ukuran aset. |
| `cachedAt` | number | Tidak | > 0 | — | Waktu cache. |
| `expiry` | number \| null | Ya | null = tidak expire | `idx_expiry` | Kadaluarsa cache. |
| `priority` | number (1–5) | Tidak | default 3 | — | Prioritas saat purge quota. |

#### 3.6.2 Index Definition — `asset_cache_meta`

```sql
CREATE OBJECT STORE asset_cache_meta (keyPath: "assetId", autoIncrement: false);
CREATE INDEX idx_url     ON asset_cache_meta (url);
CREATE INDEX idx_size    ON asset_cache_meta (size);
CREATE INDEX idx_expiry ON asset_cache_meta (expiry);
```

---

## 4. Field Type Tables (Rinci per Store)

Bagian ini merangkum tipe data formal untuk setiap store. Digunakan sebagai kontrak antara dev klien dan dev server.

### 4.1 Tipe Primitif Standar

| tipe | deskripsi | contoh |
|------|-----------|--------|
| `string` | UTF-8 text. | `"town"` |
| `number` | Float64 (IndexedDB). Gunakan int untuk counter. | `125000` |
| `boolean` | true/false. | `true` |
| `enum` | String terbatas (lihat nilai valid). | `"pending"` |
| `uuid` | UUID v4 RFC 4122. | `"a1b2...-0001"` |
| `epoch` | number ms sejak 1970 (Date.now()). | `1718000000000` |
| `json` | Object/array bebas (dengan constraint ukuran). | `{...}` |
| `url` | String URL absolut (https). | `"https://cdn.../a.png"` |
| `hash` | Hex string (sha256). | `"9f86d0..."` |

### 4.2 `offline_queue` — Type Table Lengkap

| nama | tipe | nullable | constraint | index |
|------|------|----------|------------|-------|
| `id` | uuid | T | unique, PK | primary |
| `type` | enum | F | ∈ {claimIdle,buyItem,sellItem,summonGacha,equipItem,startBattle,upgradeHero,useConsumable,marketList,collectMail,dailyCheckin} | idx_type |
| `payload` | json | F | ≤ 8 KB, no currency | — |
| `createdAt` | epoch | F | > 0 | idx_createdAt |
| `status` | enum | F | ∈ {pending,inflight,acked,failed} | idx_status |
| `retry` | number | F | 0 ≤ retry ≤ 5 | — |
| `lastError` | string\|null | Y | ≤ 512 char | — |
| `playerId` | uuid | F | ∈ active account | idx_player |
| `idempotencyKey` | string | F | == id | idx_idem |
| `clientVersion` | string | F | semver-ish | — |
| `ttl` | epoch\|null | Y | null or > createdAt | — |

### 4.3 `autosave_snapshot` — Type Table Lengkap

| nama | tipe | nullable | constraint | index |
|------|------|----------|------------|-------|
| `playerId` | uuid | F | PK | primary |
| `uiState` | json | F | ≤ 64 KB | — |
| `lastSeen` | json | F | {state,subState,timestamp} | — |
| `partyDraft` | json\|null | Y | ≤ 4 heroId | — |
| `version` | number | F | == schema ver | — |
| `currencyCache` | json | F | {gold,gems,eventToken} | — |
| `predictedDelta` | json\|null | Y | {isPredicted,goldPerHour,sinceMs} | — |
| `savedAt` | epoch | F | > 0 | idx_savedAt |
| `isStale` | boolean | F | default false | — |
| `clientVersion` | string | F | build version | — |

### 4.4 `lastSeen` Sub-Structure (State Machine)

| field | tipe | nullable | constraint | keterangan |
|-------|------|----------|------------|------------|
| `state` | enum | F | ∈ {BOOT,MENU,TOWN,STAGE_MAP,BATTLE,INVENTORY,GACHA,MARKET,SETTINGS,OFFLINE} | State utama (lihat §10.1). |
| `subState` | string | F | bebas (≤ 64 char) | Detail state (mis. `IDLE`, `AUTO`). |
| `timestamp` | epoch | F | > 0 | Kapan state diambil. |
| `stageId` | string\|null | Y | bila state == STAGE_MAP/BATTLE | Stage aktif. |
| `battleId` | string\|null | Y | bila state == BATTLE | ID pertarungan berjalan. |

> `lastSeen` dipersist agar saat reload, klien langsung kembalikan pemain ke state terakhir (resume). Namun kebenaran state ada di server; klien hanya "preview".

### 4.5 `currencyCache` Sub-Structure

| field | tipe | nullable | constraint | keterangan |
|-------|------|----------|------------|------------|
| `gold` | number | F | ≥ 0, display only | Mirror Gold server. |
| `gems` | number | F | ≥ 0, display only | Mirror Gems server. |
| `eventToken` | number | F | ≥ 0, display only | Mirror EventToken server. |
| `syncedAt` | epoch | F | waktu mirror diambil | Untuk UI "data mungkin usang". |

### 4.6 `drafts` — Type Table Lengkap

| nama | tipe | nullable | constraint | index |
|------|------|----------|------------|-------|
| `id` | uuid | F | PK | primary |
| `kind` | enum | F | ∈ {market,message,loadout,note} | idx_kind |
| `playerId` | uuid | F | ∈ active account | idx_player |
| `data` | json | F | ≤ 16 KB | — |
| `createdAt` | epoch | F | > 0 | idx_createdAt |
| `updatedAt` | epoch | F | ≥ createdAt | — |
| `ttl` | epoch\|null | Y | null = persist | — |

### 4.7 `asset_cache_meta` — Type Table Lengkap

| nama | tipe | nullable | constraint | index |
|------|------|----------|------------|-------|
| `assetId` | string | F | PK | primary |
| `url` | url | F | https | idx_url |
| `hash` | hash | F | 64 hex | — |
| `size` | number | F | ≥ 0 | idx_size |
| `cachedAt` | epoch | F | > 0 | — |
| `expiry` | epoch\|null | Y | null or > cachedAt | idx_expiry |
| `priority` | number | F | 1–5 | — |

### 4.8 Enum Values — Referensi Cepat

```
type (offline_queue):      claimIdle | buyItem | sellItem | summonGacha |
                           equipItem | startBattle | upgradeHero | useConsumable |
                           marketList | collectMail | dailyCheckin

status (offline_queue):    pending | inflight | acked | failed

kind (drafts):             market | message | loadout | note

state (lastSeen):          BOOT | MENU | TOWN | STAGE_MAP | BATTLE |
                           INVENTORY | GACHA | MARKET | SETTINGS | OFFLINE

language:                  id | en
```

---

## 5. Versioning & Migration

Setiap store memiliki `version`. DB memiliki versi sendiri. Migrasi dilakukan di callback `onupgradeneeded`.

### 5.1 Versi

| entitas | field/versi | nilai awal |
|---------|-------------|-----------|
| Database `rpg_client` | `dbVersion` | `1` |
| `offline_queue` | record `version` (implisit via dbVersion) | — |
| `autosave_snapshot` | field `version` | `1` |
| `drafts` | (implisit dbVersion) | — |
| `asset_cache_meta` | (implisit dbVersion) | — |
| localStorage | `rpg:schemaVersion` | `1` |

### 5.2 Strategi Migrasi (onupgrade)

```typescript
// PSEUDOCODE — migrasi IndexedDB
function openDB(): Promise<IDBDatabase> {
  return new Promise((resolve, reject) => {
    const req = indexedDB.open('rpg_client', DB_VERSION); // DB_VERSION = 1

    req.onupgradeneeded = (event) => {
      const db = req.result;
      const oldVer = event.oldVersion; // 0 = baru dibuat

      // Store (a) offline_queue
      if (!db.objectStoreNames.contains('offline_queue')) {
        const s = db.createObjectStore('offline_queue', { keyPath: 'id' });
        s.createIndex('idx_type', 'type', { unique: false });
        s.createIndex('idx_createdAt', 'createdAt', { unique: false });
        s.createIndex('idx_status', 'status', { unique: false });
        s.createIndex('idx_player', 'playerId', { unique: false });
        s.createIndex('idx_idem', 'idempotencyKey', { unique: false });
      }

      // Store (b) autosave_snapshot
      if (!db.objectStoreNames.contains('autosave_snapshot')) {
        const s = db.createObjectStore('autosave_snapshot', { keyPath: 'playerId' });
        s.createIndex('idx_savedAt', 'savedAt', { unique: false });
      }

      // Store (c) drafts
      if (!db.objectStoreNames.contains('drafts')) {
        const s = db.createObjectStore('drafts', { keyPath: 'id' });
        s.createIndex('idx_kind', 'kind', { unique: false });
        s.createIndex('idx_player', 'playerId', { unique: false });
        s.createIndex('idx_createdAt', 'createdAt', { unique: false });
      }

      // Store (d) asset_cache_meta
      if (!db.objectStoreNames.contains('asset_cache_meta')) {
        const s = db.createObjectStore('asset_cache_meta', { keyPath: 'assetId' });
        s.createIndex('idx_url', 'url', { unique: false });
        s.createIndex('idx_size', 'size', { unique: false });
        s.createIndex('idx_expiry', 'expiry', { unique: false });
      }

      // Future migrations: if (oldVer < 2) { ... add field ... }
      if (oldVer < 2) { migrateToV2(db); }
    };

    req.onsuccess = () => resolve(req.result);
    req.onerror = () => reject(req.error);
  });
}
```

### 5.3 Backward Compatibility Rules

| aturan | penjelasan |
|--------|-----------|
| Field baru → optional + default. | Penambahan field tidak merusak record lama. |
| Penghapusan field → jangan baca lagi, purge perlahan. | Jangan hapus kolom secara destruktif di v1. |
| Perubahan tipe → migrasi eksplisit di `onupgradeneeded`. | Cast aman (number↔string) dengan validasi. |
| `autosave_snapshot.version` mismatch → rebuild. | Jika version > expected, anggap stale, pull server. |
| localStorage `rpg:schemaVersion` naik → migrate keys. | Tambah key baru dengan default; jangan overwrite preferensi user. |

### 5.4 Migration Playbook (contoh naik ke v2)

Jika di masa depan `offline_queue` butuh field `priority`:

```typescript
if (oldVer < 2) {
  const store = event.target.result.transaction.objectStore('offline_queue');
  // Tambah field default via cursor (non-destructive)
  store.openCursor().onsuccess = (e) => {
    const cursor = e.target.result;
    if (cursor) {
      const rec = cursor.value;
      if (rec.priority === undefined) rec.priority = 3;
      cursor.update(rec);
      cursor.continue();
    }
  };
}
```

### 5.5 localStorage Migration

```typescript
const LS_VERSION_KEY = 'rpg:schemaVersion';
const CURRENT_LS_VERSION = 1;

function migrateLocalStorage() {
  const v = Number(localStorage.getItem(LS_VERSION_KEY) || '0');
  if (v < 1) {
    // v0 -> v1: inisialisasi default settings/flags bila belum ada
    if (!localStorage.getItem('rpg:settings')) {
      localStorage.setItem('rpg:settings', JSON.stringify(DEFAULT_SETTINGS));
    }
    if (!localStorage.getItem('rpg:flags')) {
      localStorage.setItem('rpg:flags', JSON.stringify(DEFAULT_FLAGS));
    }
  }
  localStorage.setItem(LS_VERSION_KEY, String(CURRENT_LS_VERSION));
}
```

---

## 6. Security

Penyimpanan klien adalah permukaan serangan. Berikut kebijakan keamanan wajib.

### 6.1 Token Handling (Kritis)

| item | lokasi | dibaca JS? | keterangan |
|------|--------|-----------|-----------|
| Access token (JWT Bearer) | httpOnly cookie (server-set) | **TIDAK** | Tidak tersentuh `localStorage`. |
| Refresh token | httpOnly cookie (server-set) | **TIDAK** | Rotasi di server. |
| `playerId` (public) | IndexedDB / state | Ya (aman, bukan rahasia) | Identifier publik, bukan secret. |
| CSRF token | header khusus / double-submit | Ya (terbatas) | Hanya untuk proteksi CSRF, bukan auth. |

> **JANGAN** pernah menaruh JWT/refresh di `localStorage` atau variabel global JS. XSS akan mencurinya. Cookie `HttpOnly; Secure; SameSite=Strict` adalah wajib.

### 6.2 localStorage Hanya Non-Sensitive

- Yang **BOLEH**: settings audio, language, keybind, flags tutorial, opt-in FCM/analytics, client version, lastSyncAt, offlineMode.
- Yang **DILARANG**: currency, inventory, jwt, refresh, email user (PII), battle results, purchase receipts.

### 6.3 Jangan Log Secret

- Jangan `console.log` payload yang mengandung token, PII, atau signature.
- Log hanya `type`, `id` (idempotency), `status`, `retry` — bukan isi sensitif.
- Di produksi, matikan log verbosity tinggi (via `import.meta.env.PROD`).

### 6.4 Sanitasi Input (Trust Boundary)

Semua data yang dibaca dari storage DIPERLAKUKAN sebagai tidak-terpercaya:

| field | validasi sebelum pakai |
|-------|------------------------|
| `language` | ∈ {`id`,`en`}; else fallback `en`. |
| `*.Volume` | number 0.0–1.0; clamp; else default. |
| `inputBindings` | key ∈ whitelist; value ∈ whitelist tombol; max 64 entri. |
| `flags.*` | boolean; else default false. |
| `offline_queue.payload` | validasi schema per `type` (zod-like) sebelum flush. |
| `autosave_snapshot.uiState` | batasi kedalaman JSON (≤ 6 level); tolak bila > 64 KB. |
| `drafts.data` | batasi ukuran; escape HTML bila dirender. |

```typescript
// PSEUDOCODE — sanitasi settings saat load
function loadSettings(): Settings {
  const raw = localStorage.getItem('rpg:settings');
  const parsed = safeParse(raw) ?? {};
  return {
    masterVolume: clamp(num(parsed.masterVolume, 0.8), 0, 1),
    musicVolume:  clamp(num(parsed.musicVolume,  0.6), 0, 1),
    sfxVolume:    clamp(num(parsed.sfxVolume,    0.7), 0, 1),
    uiVolume:     clamp(num(parsed.uiVolume,     0.5), 0, 1),
    language:     parsed.language === 'id' ? 'id' : 'en',
    inputBindings: sanitizeBindings(parsed.inputBindings),
  };
}
```

### 6.5 Content Security Policy (CSP)

PWA wajib mengirim header CSP agar XSS & injection ditekan. Rekomendasi:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';          // Phaser butuh inline style terbatas
  img-src 'self' https://cdn.rpg.example;
  connect-src 'self' https://api.rpg.example; // hanya ke API resmi
  font-src 'self';
  media-src 'self' https://cdn.rpg.example;
  object-src 'none';
  base-uri 'self';
  frame-ancestors 'none';
```

> `connect-src` dibatasi ke `api.rpg.example` → mencegah exfiltrasi token ke domain lain. `localStorage`/`IndexedDB` tidak terlindung CSP, tapi CSP mencegah JS jahat yang membacanya.

### 6.6 Encryption di Klien (Catatan Penting)

- **Tidak ada** enkripsi klien yang dianggap rahasia. Kode klien & key dapat dibongkar.
- `IndexedDB` disimpan plaintext di disk. Itu **boleh** karena tidak ada secret di dalamnya (hanya cache/queue/draft).
- Bila suatu hari perlu menyimpan data PII di klien (tidak di v1), gunakan Web Crypto API dengan key dari **server** (bukan hardcode di bundle).

### 6.7 Storage Event & Multi-Tab

- `window.addEventListener('storage', ...)` dipakai untuk sync settings antar-tab.
- Jangan broadcast data sensitif lewat `BroadcastChannel` tanpa filter.
- Saat logout di satu tab, tab lain harus mendengar event dan menghapus snapshot lokal.

---

## 7. Cleanup & Retensi

Klien harus menjaga ukuran storage agar tidak melanggar quota dan menghormati retensi data (GDPR/PDPA).

### 7.1 Quota & Monitoring

| mekanisme | detail |
|-----------|--------|
| Deteksi quota | Tangkap `QuotaExceededError` pada write IndexedDB. |
| Estimasi pemakaian | `navigator.storage.estimate()` (jika didukung). |
| Ambang peringatan | Jika `usage / quota > 0.8`, picu purge `asset_cache_meta`. |
| Batas keras | `asset_cache_meta` total ≤ 200 MB; `offline_queue` ≤ 500 record. |

### 7.2 Purge Policy — `offline_queue`

| kondisi | aksi |
|--------|------|
| `status == acked` dan umur > 7 hari | Hapus (hard purge). |
| `status == failed` dan `retry >= 5` | Pindah ke "dead-letter" log lokal (tanpa secret), lalu hapus dari queue. |
| `ttl` terlewati | Hapus (aksi kadaluarsa). |
| `playerId` ≠ akun aktif | Hapus (orphan). |

### 7.3 Purge Saat Logout / Delete Account

```typescript
async function wipeForLogout(playerId: string): Promise<void> {
  const db = await openDB();
  // Hapus snapshot milik player
  db.transaction('autosave_snapshot', 'readwrite')
    .objectStore('autosave_snapshot').delete(playerId);
  // Hapus queue & draft milik player
  deleteByIndex('offline_queue', 'idx_player', playerId);
  deleteByIndex('drafts', 'idx_player', playerId);
  // Hapus opt-in FCM/analytics (PII-adjacent) dari localStorage
  localStorage.removeItem('rpg:fcmOptIn');
  localStorage.removeItem('rpg:flags'); // flags berisi opt-in analytics
  // Catatan: settings audio BOLEH dipertahankan (preferensi UI, bukan akun)
}
```

> **GDPR / PDPA:** Saat "delete account" diminta, klien harus memanggil endpoint server `/account/delete`, lalu menjalankan `wipeForLogout` + menghapus `rpg:settings` juga (karena settings dianggap milik akun, bukan perangkat, di kasus delete-account). Untuk logout biasa, `rpg:settings` dipertahankan.

### 7.4 Retensi `asset_cache_meta`

| prioritas | kebijakan purge saat quota penuh |
|-----------|----------------------------------|
| 1 (terendah) | Dihapus pertama saat perlu ruang. |
| 5 (tertinggi) | Dipertahankan (mis. sprite karakter inti, font). |
| `expiry` < now | Dihapus dulu (kadaluarsa). |
| `cachedAt` terlama | Dihapus berikutnya (LRU). |

### 7.5 Retensi `autosave_snapshot`

| kondisi | aksi |
|--------|------|
| `savedAt` < now - 30 hari (akun tidak login) | Tandai `isStale`; biarkan server jadi otoritas saat login. |
| Konflik versi (`version` > expected) | Rebuild dari server (jangan pakai snapshot usang). |
| Logout | Hapus (§7.3). |

### 7.6 Jadwal Cleanup

- Cleanup ringan: setiap kali autosave (throttle 5 dtk, lihat §8.4).
- Cleanup berat (purge asset): saat `estimate().usage/quota > 0.8`.
- Cleanup periodik: saat app resume dari background (visibilitychange).

---

## 8. Sync Strategy

Ini jantung arsitektur server-authoritative. Klien mengantre aksi, menyiramnya saat online, dan menerima state incremental dari server.

### 8.1 Status Koneksi

| status | definisi |
|--------|----------|
| `ONLINE` | `navigator.onLine === true` DAN heartbeat API sukses. |
| `OFFLINE` | `navigator.onLine === false` ATAU heartbeat gagal 3x. |
| `RECONNECTING` | Transisi OFFLINE→ONLINE; sedang flush queue. |

> `navigator.onLine` tidak cukup akurat; gunakan heartbeat ke `/ping` tiap 30 dtk.

### 8.2 Saat Online — Flush Queue

```typescript
// PSEUDOCODE — flush offline_queue ke /sync (idempoten)
async function flushQueue(): Promise<void> {
  const pending = await getByIndex('offline_queue', 'idx_status', 'pending');
  for (const item of pending) {
    item.status = 'inflight';
    await put('offline_queue', item);
    try {
      const res = await api.post('/sync', {
        idempotencyKey: item.idempotencyKey,
        type: item.type,
        payload: item.payload,
      }, { headers: { 'Idempotency-Key': item.idempotencyKey } });
      // Server mengembalikan state baru (currency, inventory, dll)
      applyServerState(res.data);
      item.status = 'acked';
    } catch (err) {
      item.retry += 1;
      item.lastError = sanitizeError(err);
      item.status = item.retry >= MAX_RETRY ? 'failed' : 'pending';
    }
    await put('offline_queue', item);
  }
}
```

### 8.3 Idempotency (Kritis)

- Setiap aksi di `offline_queue` membawa `idempotencyKey == id`.
- Server menyimpan map `idempotencyKey → result` (TTL 24j).
- Bila klien retry dengan key sama, server mengembalikan hasil lama tanpa menjalankan ulang efek ekonomi.
- Ini mencegah duplikasi currency saat retry / flush ganda.

### 8.4 Save Throttle (Game Loop Integration)

| event | aksi | throttle |
|-------|------|----------|
| Tick idle (5 dtk) | Update `predictedDelta` di memory; **tidak** langsung write. | Write snapshot tiap 15 dtk (3 tick). |
| State change (UI) | Update `uiState`/`lastSeen` di memory. | Debounce 1 dtk sebelum write. |
| Party draft change | Update `partyDraft`. | Debounce 500 ms. |
| Visibility hidden | **Langsung** write snapshot (panic save). | Tanpa throttle. |
| Before unload | `navigator.sendBeacon` ping logout-soft + write. | Sekali. |
| Quota error | Hentikan write, picu purge. | — |

```typescript
// PSEUDOCODE — throttle autosave
let lastSave = 0;
const SAVE_INTERVAL = 15_000; // 15 detik

function maybeAutosave(snapshot: Snapshot) {
  const now = Date.now();
  if (now - lastSave >= SAVE_INTERVAL) {
    writeSnapshot(snapshot); // IndexedDB
    lastSave = now;
  }
}

document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') {
    writeSnapshot(currentSnapshot()); // panic save, ignore throttle
  }
});
```

### 8.5 Pull Incremental State

- Endpoint: `GET /state?since={lastSyncAt}` → mengembalikan delta (currency, inventory, mail, dll).
- Klien update `currencyCache` dari delta; set `isStale=false`.
- Bila delta tidak bisa diterapkan (version mismatch), klien lakukan full pull `GET /state/full`.

### 8.6 Saat Offline — Queue & Predict

```typescript
// PSEUDOCODE — enqueue saat offline
function performAction(type: ActionType, payload: object) {
  const item = {
    id: uuidv4(),
    type,
    payload,
    createdAt: Date.now(),
    status: 'pending',
    retry: 0,
    lastError: null,
    playerId: currentPlayerId,
    idempotencyKey: '', // diisi = id setelah dibuat
    clientVersion: BUILD_VERSION,
    ttl: computeTtl(type),
  };
  item.idempotencyKey = item.id;
  if (isOnline()) {
    flushItem(item); // coba langsung
  } else {
    put('offline_queue', item); // antre
    applyLocalPrediction(type, payload); // prediksi UI (flag isPredicted)
  }
}
```

### 8.7 Conflict Resolution (Server-Wins)

| skenario | resolusi |
|----------|----------|
| Klien prediksi +1000 gold, server bilang +800 | **Server menang**: `currencyCache.gold = serverValue`. |
| Queue belum flush, server sudah beri reward lain | Gabungkan via delta server; abaikan prediksi lokal. |
| Dua device aksi bersamaan | Server proses berurutan; idempotency cegah duplikat. |
| Snapshot `version` < server | Buang snapshot, pull full. |

> **Prinsip:** Prediksi lokal hanya untuk *responsivitas UI*. Begitu server merespons, prediksi dibuang dan diganti nilai server.

### 8.8 Idle Cap & Tick

- Idle cap = **24 jam**. Server menghitung akumulasi dari `lastSeen.timestamp` hingga `now`, dibatasi 24j.
- Tick klien = **5 detik** untuk update UI prediksi (`predictedDelta`).
- Saat claim idle (`claimIdle`), klien kirim `seconds` + `ticket` (server-issued); server validasi & batasi 24j.
- Klien **tidak** menghitung reward idle riil; hanya menampilkan proyeksi.

---

## 9. Anti-Cheat Notes

Karena server-authoritative, beban anti-cheat ada di server. Klien hanya memastikan integritas antrian & tidak membocorkan vektor.

### 9.1 Validasi Server per Aksi

| aksi (queue type) | validasi server wajib |
|-------------------|------------------------|
| `claimIdle` | Batas 24j; `ticket` valid & belum dipakai; `seconds` ∈ [0, 86400]. |
| `buyItem` | Saldo cukup; harga sesuai catalog versi server; item ada. |
| `sellItem` | Pemain punya `qty` item; item sellable. |
| `summonGacha` | Bismillah: server rng + pity; deduksi currency; banner aktif. |
| `equipItem` | Hero & item milik pemain; slot valid. |
| `startBattle` | Stage terbuka; party valid; tidak sedang battle lain. |
| `upgradeHero` | Resource cukup; level ≤ cap; stat valid. |
| `useConsumable` | Item ada di battle; target valid. |
| `marketList` | Item milik pemain; harga ∈ range; belum listed. |
| `collectMail` | Mail ada & belum diklaim. |
| `dailyCheckin` | Streak valid; belum klaim hari ini. |

### 9.2 Penolakan Manipulasi

- Jika `payload` berisi field tidak dikenal (mis. `gold: 999999`), server **membuang** field & memproses sisanya, atau menolak seluruhnya (fail-closed).
- Jika `playerId` di queue ≠ token JWT, tolak (401).
- Jika `idempotencyKey` dipakai ulang dengan `payload` beda, tolak (tamper).

### 9.3 Signature & Integrity

- Opsional v1: server tandatangani `idempotencyKey` dengan HMAC saat issue `ticket`.
- Klien tidak perlu verifikasi signature (tidak punya secret); verifikasi di server.
- Setiap response `/sync` menyertakan `serverStateHash`; klien simpan untuk deteksi tamper lokal (display warning, bukan validasi).

### 9.4 Client-Side Hardening

| teknik | tujuan |
|--------|--------|
| Minify + obfuscate bundle | Naikkan hambatan bongkar kode. |
| Deteksi debugger (best-effort) | Usir cheat tool kasar. |
| Jangan ekspos fungsi `grantCurrency` di global | Tutup vektor konsol. |
| Integrity check aset (hash di `asset_cache_meta`) | Cegah modifikasi sprite/audio lokal. |
| Rate limit flush (≤ N req/detik) | Cegah spam endpoint. |

> **Peringatan:** Semua hardening klien bersifat *best-effort*. Klien terpercaya tapi diverifikasi — kebenaran tetap di server.

### 9.5 Logging untuk Forensik

- Server log: `playerId`, `type`, `idempotencyKey`, `result`, `timestamp`, `clientVersion`.
- Jika pola anomali (mis. claimIdle 24j tiap 5 menit), server flag akun untuk review.
- Klien log minimal: `type`, `id`, `status`, `retry` saja.

---

## 10. Hubungan dengan Dokumen Lain

Skema ini terhubung erat dengan dokumen arsitektur lain. Penamaan field konsisten (CDP) lintas dokumen.

### 10.1 State Machine (Persist `last_seen` / `state`)

Dokumen *Game State Machine* mendefinisikan enum `state` yang dipakai di `autosave_snapshot.lastSeen.state`:

```
BOOT → MENU → TOWN ⇄ STAGE_MAP → BATTLE → (result) → TOWN
              ⇄ INVENTORY ⇄ GACHA ⇄ MARKET ⇄ SETTINGS
OFFLINE (overlay saat koneksi putus)
```

- Klien persist `lastSeen` tiap transisi state (debounce 1 dtk).
- Saat boot, klien baca `lastSeen` untuk resume layar, tapi server validasi akses (mis. tidak bisa resume mid-battle bila server bilang battle sudah selesai).

### 10.2 Game Loop (Save Throttle)

Dokumen *Game Loop* mendefinisikan tick 5 dtk. Hubungan:

| game loop event | save action |
|-----------------|-------------|
| Idle tick (5s) | Update prediksi; snapshot tiap 15s. |
| Battle tick | Tidak simpan state battle (server otoritas). |
| Pause/visibility hidden | Panic save snapshot. |
| Settings change | Debounce save ke localStorage. |

### 10.3 API Contract (Sync Endpoint)

Dokumen *API Contract* mendefinisikan:

| endpoint | method | hubungan dengan save schema |
|----------|--------|------------------------------|
| `/sync` | POST | Menerima `offline_queue` items (idempotencyKey). |
| `/state?since=` | GET | Mengembalikan delta untuk update `currencyCache`. |
| `/state/full` | GET | Full pull saat version mismatch. |
| `/ping` | GET | Heartbeat untuk status ONLINE/OFFLINE. |
| `/account/delete` | POST | Pemicu wipe lokal (§7.3). |
| `/auth/refresh` | POST | Rotasi refresh cookie (httpOnly). |

> Field `idempotencyKey` di schema == header `Idempotency-Key` di API contract. Konsisten CDP.

### 10.4 Asset Loading

Dokumen *Asset Pipeline* mendefinisikan hash aset. Hubungan:

- `asset_cache_meta.hash` harus cocok dengan hash di manifest aset server.
- Saat load aset, klien cek `asset_cache_meta`; bila `hash` tidak cocok atau `expiry` lewat, re-fetch.
- PWA `Cache API` menyimpan biner; `asset_cache_meta` menyimpan metadata saja.

### 10.5 Auth & Session

Dokumen *Auth* mendefinisikan:

- Login → server set httpOnly cookie (access + refresh).
- Klien **tidak** simpan token; hanya baca `playerId` dari respons login (public id).
- Logout → server hapus cookie; klien jalankan `wipeForLogout` (§7.3).

---

## 11. Appendix A — Asumsi Eksplisit

| # | asumsi | dampak schema |
|---|--------|---------------|
| A1 | Satu akun aktif per browser pada satu waktu (v1). | Tidak ada multi-account store terpisah. |
| A2 | `playerId` adalah UUID v4 dari server, dianggap publik. | Disimpan di IndexedDB, bukan rahasia. |
| A3 | Offline adalah kondisi transien, bukan mode utama. | Queue dibatasi; purge agresif. |
| A4 | Akses internet tersedia mayoritas waktu. | Sync inkremental sering; cache sekunder. |
| A5 | Idle cap 24j, tick 5 dtk (given). | `claimIdle` validasi 24j; prediksi 5s. |
| A6 | 3 currency: Gold, Gems, EventToken (given). | `currencyCache` = 3 field fixed. |
| A7 | Auth email/password + JWT Bearer + refresh httpOnly (given). | Token tidak di localStorage. |
| A8 | PWA installable, desktop-first (given). | CSP, asset cache, offline queue relevan. |
| A9 | `localStorage` aman untuk preferensi non-ekonomi. | Settings/flags di sana. |
| A10 | Quota IndexedDB bervariasi per browser (~50MB–unlimited). | Purge berbasis `estimate()`. |
| A11 | Bahasa UI v1: `id` & `en`. | Enum `language` = 2 nilai. |
| A12 | Tidak ada PII di klien di v1 (email hanya di server). | `rpg:flags` tidak simpan email. |
| A13 | Battle outcome ditentukan server sepenuhnya. | Klien tidak cache hasil battle. |
| A14 | Gacha pity & rng di server. | Klien hanya kirim `summonGacha` intent. |

---

## 12. Appendix B — Contoh JSON Snapshot

### 12.1 `offline_queue` record (claimIdle)

```json
{
  "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "type": "claimIdle",
  "payload": {
    "stageId": "stage_0142",
    "seconds": 43200,
    "ticket": "idle-tok-9f8e7d6c5b4a"
  },
  "createdAt": 1718000000000,
  "status": "pending",
  "retry": 0,
  "lastError": null,
  "playerId": "a1b2c3d4-0000-4000-8000-000000000001",
  "idempotencyKey": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "clientVersion": "1.0.0-rc3",
  "ttl": 1718086400000
}
```

### 12.2 `offline_queue` record (summonGacha)

```json
{
  "id": "7c6b8e2a-1f3d-4c5b-9a0e-2d4f6a8b1c3e",
  "type": "summonGacha",
  "payload": { "bannerId": "banner_anniv", "pulls": 10 },
  "createdAt": 1718000100000,
  "status": "inflight",
  "retry": 1,
  "lastError": null,
  "playerId": "a1b2c3d4-0000-4000-8000-000000000001",
  "idempotencyKey": "7c6b8e2a-1f3d-4c5b-9a0e-2d4f6a8b1c3e",
  "clientVersion": "1.0.0-rc3",
  "ttl": null
}
```

### 12.3 `drafts` record (market)

```json
{
  "id": "d3e4f5a6-2b1c-4d7e-8f90-1a2b3c4d5e6f",
  "kind": "market",
  "playerId": "a1b2c3d4-0000-4000-8000-000000000001",
  "data": {
    "itemId": "sword_flame_03",
    "qty": 2,
    "price": 15000,
    "currency": "gold"
  },
  "createdAt": 1717990000000,
  "updatedAt": 1717995000000,
  "ttl": 1718594800000
}
```

---

## 13. Appendix C — Glosarium

| istilah | arti |
|---------|------|
| SSOT | Single Source of Truth — server sebagai kebenaran tunggal. |
| Idempotency Key | Identifier unik tiap aksi agar pengiriman ulang tidak menggandakan efek. |
| Predicted Delta | Proyeksi perubahan (mis. gold idle) di klien saat offline; bukan kebenaran. |
| Autosave Snapshot | Cache state UI & prediksi terakhir di IndexedDB. |
| Offline Queue | Antrean aksi yang belum dikonfirmasi server. |
| httpOnly Cookie | Cookie tidak dapat dibaca JS; aman dari XSS token-theft. |
| CSP | Content Security Policy — header keamanan batas sumber eksekusi. |
| CDP | Consistent Design Parameter — parameter desain yang dipakai seragam lintas dokumen. |
| Heartbeat | Ping periodik ke server untuk deteksi status koneksi. |
| Panic Save | Penyimpanan segera saat aplikasi akan tertutup/tersembunyi. |
| TTL | Time-To-Live — batas waktu validitas record. |
| PII | Personally Identifiable Information — data yang mengidentifikasi individu. |
| GDPR / PDPA | Regulasi perlindungan data (EU / Indonesia). |
| QuotaExceededError | Error saat storage melewati batas browser. |

---

## Ringkasan Kontrak (Executive)

- **Kebenaran:** Server otoritatif untuk currency (Gold/Gems/EventToken), drops, gacha, battle, progression. Klien = cache + prediksi + offline-queue.
- **localStorage:** Hanya settings audio/UI, flags non-ekonomi, FCM opt-in. **Tidak ada token** (httpOnly cookie).
- **IndexedDB (`rpg_client`):** 4 store — `offline_queue`, `autosave_snapshot`, `drafts`, `asset_cache_meta`. Masing-masing punya keyPath, index, dan `version`.
- **Sync:** Flush queue idempoten saat online; pull delta; server-wins konflik; prediksi lokal dibuang saat server merespons.
- **Security:** Token httpOnly; localStorage non-sensitive; sanitasi trust-boundary; CSP `connect-src` ke API resmi; tidak ada enkripsi klien dianggap rahasia.
- **Retensi:** Purge queue sukses >7 hari, purge asset saat quota >80%, wipe snapshot saat logout, full wipe + hapus settings saat delete-account (GDPR/PDPA).
- **Anti-cheat:** Validasi server per aksi; tolak payload manipulatif; idempotency cegah duplikat; signature opsional di v1.

> Dokumen ini mengikat. Setiap perubahan schema harus menaikkan `version` dan didokumentasikan di §5 (Migration). Konsistensi penamaan field dengan API Contract, State Machine, Game Loop, dan Auth wajib dijaga (CDP).

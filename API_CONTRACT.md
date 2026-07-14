# API CONTRACT — Fantasy Turn-Based Idle RPG (Web + PWA)

> **Dokumen:** API Contract (Spesifikasi Endpoint REST & WebSocket)
> **Versi:** 1.0.0
> **Status:** Kanonikal (Final)
> **Bahasa:** Indonesia
> **Owner:** Backend / API Architecture Team
> **Terakhir diperbarui:** 2026-07-14 (UTC)

---

## Daftar Isi

1. [Konvensi Umum](#1-konvensi-umum)
2. [Strategi Autentikasi (Auth Strategy)](#2-strategi-autentikasi-auth-strategy)
3. [REST Endpoints](#3-rest-endpoints)
   - 3.1. [Profile & Progression](#31-profile--progression)
   - 3.2. [Hero & Party](#32-hero--party)
   - 3.3. [Gacha](#33-gacha)
   - 3.4. [PvE (Stage & Battle)](#34-pve-stage--battle)
   - 3.5. [Inventory & Equipment](#35-inventory--equipment)
   - 3.6. [Marketplace](#36-marketplace)
   - 3.7. [Guild & Raid](#37-guild--raid)
   - 3.8. [PvP & Leaderboard](#38-pvp--leaderboard)
   - 3.9. [Events](#39-events)
   - 3.10. [Shop & Monetization](#310-shop--monetization)
   - 3.11. [Admin / GM](#311-admin--gm)
   - 3.12. [GDPR / PDPA (Data Subject Rights)](#312-gdpr--pdpa-data-subject-rights)
4. [WebSocket](#4-websocket)
5. [Mekanisme Idle Claim](#5-mekanisme-idle-claim)
6. [Kode Error](#6-kode-error)
7. [Rate Limits & Pagination](#7-rate-limits--pagination)
8. [Keamanan (Security)](#8-keamanan-security)
9. [Hubungan dengan Dokumen Lain](#9-hubungan-dengan-dokumen-lain)
10. [Lampiran: Model Data Inti & Enum](#10-lampiran-model-data-inti--enum)

---

## 0. Asumsi Eksplisit (Canonical Design Parameters)

Dokumen ini **wajib** konsisten dengan parameter desain kanonikal berikut. Semua angka, nama, dan perilaku di bawah digunakan secara seragam di seluruh endpoint.

| Parameter | Nilai | Keterangan |
|---|---|---|
| Backend | Node.js / NestJS | Framework aplikasi (modular, DI). |
| Database | PostgreSQL | Sumber truth (ACID) untuk akun, ledger, progresi. |
| Cache / PubSub | Redis | Sesi, lock distribusi, antrian realtime, cache ladder. |
| Job Queue | BullMQ | Proses async (email, settle marketplace, settlement guild). |
| Realtime | WebSocket (socket.io atau ws) | Event push (leaderboard, PvP, notif). |
| CDN | CDN statis | Aset (sprite, audio, manifest PWA). |
| Auth | Email/Password | Access JWT (Bearer, pendek) + Refresh (httpOnly cookie, rotasi). |
| Otoritas | Server-authoritative | Klien TIDAK menghitung reward/hasil; server yang resolve. |
| Ukuran Party | 5 | `party` maksimal 5 slot hero aktif. |
| Rarity | C / R / E / L (+M) | Common, Rare, Epic, Legendary, Mythic (limited/event). |
| Gacha rate | 60 / 30 / 9 / 1 | C=60%, R=30%, E=9%, L=1%. |
| Gacha pity | 10 / 75 / 90 | Floor Rare tiap 10; soft-pity mulai 75; hard-pity L di 90. |
| Idle tick | 5 detik | Interval akumulasi idle di server. |
| Idle cap | 24 jam | Akrual offline di-clamp maksimal 24h. |
| Marketplace fee | 5% | Pajak platform dari hasil jual. |
| PvP ELO awal | 1000 | Rating awal pemain di ladder PvP. |
| Currency | 3 jenis | `gold`, `gem`, `rune` (lihat §10). |
| Compliance | GDPR / PDPA | Export & delete data subjek; audit log. |
| Admin | Panel GM | Role `admin`/`gm` untuk operasi GM. |

**Asumsi tambahan (eksplisit, boleh disesuaikan):**

- `gold` = soft currency (drop PvE, idle, quest). Tidak dapat dibeli langsung.
- `gem` = premium currency (top-up riil, event). Satu-satunya mata uang monetisasi.
- `rune` = PvP/gacha currency (didapat dari PvP & event; bukan top-up langsung).
- Semua timestamp dikembalikan dalam **UTC, format ISO8601** (`2026-07-14T03:38:56Z`).
- ID menggunakan UUID v4 (string), kecuali dinyatakan lain (mis. `runId` internal bisa ULID).
- Semua respons body berupa JSON dengan `Content-Type: application/json; charset=utf-8`.
- Klien **hanya menampilkan**; tidak ada perhitungan reward/ekonomi di sisi klien.
- Waktu server referensi diambil dari `Date.now()` server (NTP-sync); klien tidak boleh mengirim `now`.

---

## 1. Konvensi Umum

### 1.1 Base URL & Versioning

```
REST:    https://api.example.com/api/v1
WS:      wss://api.example.com/ws
CDN:     https://cdn.example.com
```

- Versioning via **path prefix** `/api/v1`. Versi mayor di path; versi minor di header `API-Version` (opsional, default terbaru minor).
- Breaking change → naik versi path (`/api/v2`). Non-breaking → rilis minor tanpa ubah path.
- Endpoint yang sudah tidak dipakai diberi flag `deprecated: true` di `meta` selama minimal 90 hari sebelum dihapus.

### 1.2 Content-Type & Encoding

- Request: `Content-Type: application/json; charset=utf-8`.
- Response: `Content-Type: application/json; charset=utf-8`.
- Encoding UTF-8 seluruhnya. Field kosong dihilangkan (bukan `null`) kecuali untuk field wajib schema.

### 1.3 Envelope Respons Standar

Setiap respons REST membungkus payload dalam **envelope** tunggal:

```json
{
  "data": { },
  "error": null,
  "meta": {
    "requestId": "req_8f2c1a9b3e",
    "timestamp": "2026-07-14T03:38:56Z",
    "version": "1.0.0",
    "pagination": null
  }
}
```

- `data` — payload sukses (object/array). `null` saat error.
- `error` — object error saat gagal (lihat §6), `null` saat sukses.
- `meta` — metadata lintas endpoint:
  - `requestId` — ID korelasi unik (untuk tracing & support).
  - `timestamp` — waktu server (UTC ISO8601).
  - `version` — versi API yang melayani.
  - `pagination` — ada bila respons ter-paginasikan (lihat §7).

### 1.4 Zona Waktu & Format Tanggal

- Semua waktu **UTC**.
- Format: **ISO8601** (`YYYY-MM-DDTHH:mm:ssZ`). Durasi pakai ISO8601 duration bila relevan (`PT24H`).
- Klien wajib mengonversi ke lokal sendiri untuk tampilan.

### 1.5 Header Standar (Request)

| Header | Wajib? | Keterangan |
|---|---|---|
| `Authorization` | Ya* | `Bearer <access_jwt>` untuk endpoint terproteksi. |
| `Content-Type` | Ya (POST/PUT/PATCH) | `application/json`. |
| `Accept` | Tidak | `application/json` (default). |
| `Idempotency-Key` | Ya** | UUID untuk operasi mutasi berbayar (marketplace, purchase). |
| `X-Request-Timestamp` | Ya (WS/anti-replay) | epoch ms; lihat §8.4. |
| `X-Nonce` | Ya (WS/anti-replay) | string acak unik per request. |
| `X-Client-Version` | Ya | Versi build klien (PWA). |
| `X-Platform` | Ya | `web` / `pwa` / `ios` / `android`. |
| `Accept-Language` | Tidak | `id-ID` default. |

\* Kecuali endpoint publik (`/auth/login`, `/auth/register`, `/auth/refresh` via cookie, healthcheck).
\** Lihat §8.3.

### 1.6 Konvensi Penamaan

- Path: kebab-case, plural untuk koleksi (`/heroes`, `/market/listings`).
- Field JSON: camelCase (`bannerId`, `lastSeenAt`).
- Enum: UPPER_SNAKE (`COMMON`, `RARE`, `EPIC`, `LEGENDARY`, `MYTHIC`).
- Method semantik: `GET` baca, `POST` buat/aksin, `PUT` replace, `PATCH` partial, `DELETE` hapus.

### 1.7 Healthcheck

```
GET /healthz
GET /api/v1/health
```

Respons: `{"data":{"status":"ok","uptimeSec":12345,"db":"up","redis":"up"},"error":null,"meta":{...}}`

---

## 2. Strategi Autentikasi (Auth Strategy)

### 2.1 Ringkasan

Sistem menggunakan **dual-token**:

- **Access Token** — JWT pendek (TTL **15 menit**), dikirim via header `Authorization: Bearer <access>`. Tidak disimpan di cookie.
- **Refresh Token** — opaque token (bukan JWT) dengan TTL **30 hari**, disimpan di `httpOnly`, `Secure`, `SameSite=Strict` cookie. Dipakai HANYA untuk `/auth/refresh`.

**Rotasi wajib:** setiap `/auth/refresh` membatalkan refresh lama dan mengeluarkan pasangan access+refresh baru (refresh rotation). Deteksi reuse refresh token → batalkan seluruh sesi user (logout paksa di semua perangkat).

### 2.2 Diagram Alur

```
[Register/Login]
   └─> 200 OK: { data: { accessToken, expiresIn } }  +  Set-Cookie: refresh=<opaque>; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth/refresh; Max-Age=2592000

[Request API]
   └─> Header: Authorization: Bearer <access>

[Access expired]
   └─> POST /auth/refresh  (browser kirim cookie refresh otomatis)
        └─> 200 OK: { data: { accessToken, expiresIn } } + Set-Cookie refresh BARU (rotasi)

[Logout]
   └─> POST /auth/logout -> hapus refresh di DB + Clear-Cookie
```

### 2.3 Endpoint Auth

#### POST /auth/register

Daftar akun baru (email/password).

**Rate limit:** 10 req/ip/menit.

**Request body:**
```json
{
  "email": "player@example.com",
  "password": "Str0ngP@ssw0rd!",
  "displayName": "ShadowKnight",
  "acceptTos": true,
  "lang": "id-ID"
}
```

**Validasi:**
- `email` — RFC5322, lowercase, trim, unik.
- `password` — min 10 char, wajib ada huruf besar, kecil, angka, simbol (PDPA/keamanan).
- `displayName` — 3–20 char, alfanumerik + underscore, unik (case-insensitive).
- `acceptTos` — harus `true`.

**Response 201:**
```json
{
  "data": {
    "userId": "usr_9a1b2c3d",
    "email": "player@example.com",
    "displayName": "ShadowKnight",
    "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 900,
    "emailVerified": false,
    "createdAt": "2026-07-14T03:38:56Z"
  },
  "error": null,
  "meta": { "requestId": "req_a1", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Set-Cookie (response header):**
```
Set-Cookie: refresh=<opaque_token>; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth/refresh; Max-Age=2592000
```

**Error:** `409` (email/displayName sudah dipakai), `422` (validasi gagal).

---

#### POST /auth/login

Login email/password → access JWT + set refresh cookie.

**Rate limit:** 20 req/ip/menit (brute-force guard). Setelah 5 gagal berturut-turut → lock 15 menit (429 + `retryAfterSec`).

**Request body:**
```json
{
  "email": "player@example.com",
  "password": "Str0ngP@ssw0rd!",
  "deviceId": "dev_abc123"
}
```

**Response 200:**
```json
{
  "data": {
    "userId": "usr_9a1b2c3d",
    "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 900,
    "emailVerified": true,
    "lastSeenAt": "2026-07-14T03:38:56Z"
  },
  "error": null,
  "meta": { "requestId": "req_b2", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```
+ `Set-Cookie` refresh (sama seperti register).

**Error:** `401` (kredensial salah), `423` (akun terkunci sementara), `403` (akun diban).

---

#### POST /api/v1/auth/refresh

Rotasi access+refresh. **Tidak butuh** `Authorization` header; membaca cookie `refresh`.

**Auth:** Cookie `refresh` (httpOnly).
**Rate limit:** 30 req/ip/menit.

**Request body:** kosong (`{}`) atau `{}`.

**Response 200:**
```json
{
  "data": {
    "accessToken": "eyJ...baru",
    "expiresIn": 900,
    "refreshRotated": true
  },
  "error": null,
  "meta": { "requestId": "req_c3", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```
+ `Set-Cookie` refresh baru.

**Error:** `401` (refresh tidak valid/kadaluarsa), `403` (refresh reuse terdeteksi → semua sesi dibatalkan).

> **Anti-reuse:** server simpan `refreshTokenId` + `replacedBy`. Bila token lama dipakai setelah rotasi → `403 REUSE_DETECTED`, seluruh token user di-revoke.

---

#### POST /api/v1/auth/logout

Hapus sesi saat ini (revoke refresh token).

**Auth:** Bearer access + cookie refresh.
**Rate limit:** 10 req/user/menit.

**Request body:**
```json
{ "allDevices": false }
```
- `allDevices: true` → revoke semua refresh token user (logout semua perangkat).

**Response 200:**
```json
{ "data": { "loggedOut": true }, "error": null, "meta": { "requestId": "req_d4", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" } }
```
+ `Set-Cookie: refresh=; Max-Age=0` (clear).

---

#### POST /api/v1/auth/reset-password

Minta & konfirmasi reset password (flow 2 langkah).

**Langkah 1 — request token:**
```json
{ "email": "player@example.com" }
```
Respons 202: `{ "data": { "resetRequested": true }, "error": null, "meta": {...} }`
> Untuk keamanan, respons SELALU 202 meski email tak terdaftar (hindari user enumeration). Email reset dikirim via BullMQ.

**Langkah 2 — set password baru (token dari email):**
```json
{ "token": "rst_8f2c...", "password": "NeWp@ssw0rd!" }
```
Respons 200: `{ "data": { "reset": true }, "error": null, "meta": {...} }` → semua refresh token lama di-revoke.

**Rate limit:** 5 req/ip/menit.

---

#### POST /api/v1/auth/verify-email

Verifikasi email via token (dikirim saat register).

**Request body:**
```json
{ "token": "vrf_1a2b3c" }
```

**Response 200:**
```json
{ "data": { "emailVerified": true, "verifiedAt": "2026-07-14T03:38:56Z" }, "error": null, "meta": {...} }
```

**Error:** `409` (sudah terverifikasi), `422` (token invalid/expired).

---

### 2.4 Struktur Access JWT

Header `alg: RS256`, payload claim:

```json
{
  "sub": "usr_9a1b2c3d",
  "email": "player@example.com",
  "role": "player",
  "scope": ["game:read", "game:write"],
  "iss": "https://api.example.com",
  "aud": "rpg-web",
  "iat": 1752459536,
  "exp": 1752459536,
  "jti": "tok_abc123"
}
```

- `role`: `player` | `gm` | `admin`.
- `scope`: hak akses granular.
- Verifikasi via public key (JWKS endpoint `/.well-known/jwks.json`).

---

## 3. REST Endpoints

> **Legenda kolom Auth:** `Bearer` = butuh `Authorization: Bearer <access>`; `Public` = tanpa auth; `Admin` = butuh role `admin`/`gm`.
> **Rate limit** diformat `N req / window`. Header respons: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (lihat §7).

---

### 3.1 Profile & Progression

#### GET /api/v1/me

Profil lengkap pemain (identitas, currency, stat ringkas).

**Auth:** Bearer. **Rate limit:** 60/user/menit.
**Query:** `-` (opsional `?include=party,stats` untuk eager-load).

**Response 200:**
```json
{
  "data": {
    "userId": "usr_9a1b2c3d",
    "displayName": "ShadowKnight",
    "email": "player@example.com",
    "emailVerified": true,
    "level": 42,
    "exp": 183200,
    "currency": { "gold": 1250000, "gem": 320, "rune": 87 },
    "rarityOwned": { "C": 30, "R": 18, "E": 7, "L": 2, "M": 0 },
    "lastSeenAt": "2026-07-14T03:38:56Z",
    "createdAt": "2026-05-01T10:00:00Z",
    "pvpElo": 1240,
    "guildId": "gld_55",
    "settings": { "lang": "id-ID", "notifPush": true }
  },
  "error": null,
  "meta": { "requestId": "req_e5", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

---

#### GET /api/v1/progression

Detail progresi: stage tertinggi, idle state, achievement ringkas.

**Auth:** Bearer. **Rate limit:** 60/user/menit.

**Response 200:**
```json
{
  "data": {
    "level": 42,
    "exp": 183200,
    "expToNext": 17000,
    "highestStageCleared": 87,
    "idle": {
      "lastClaimAt": "2026-07-13T22:00:00Z",
      "lastSeenAt": "2026-07-14T03:38:56Z",
      "accruedSec": 19836,
      "accruedCapped": true,
      "pendingGold": 198360,
      "pendingExp": 9918
    },
    "achievements": { "total": 54, "completed": 31 },
    "dailyStreak": 12
  },
  "error": null,
  "meta": { "requestId": "req_f6", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

> Field `idle` bersifat **READ-ONLY informasi**; reward sebenarnya di-claim via `POST /api/v1/claim-idle` (§5).

---

#### POST /api/v1/claim-idle

Klaim idle reward. **Server menghitung** `(now - lastSeenAt)`, clamp `[0, 86400]` detik, hasilkan reward. Klien hanya tampilkan.

**Auth:** Bearer. **Rate limit:** 6/user/menit (serangan 5 dtk secara teori, tapi dibatasi untuk cegah spam).
**Idempotency:** tidak wajib (server pakai `lastClaimAt` guard).

**Request body:** `{}` (kosong).

**Response 200:**
```json
{
  "data": {
    "claimedSec": 86400,
    "capped": true,
    "reward": {
      "gold": 864000,
      "exp": 43200,
      "items": [
        { "itemId": "itm_idle_chest_c", "qty": 2 }
      ]
    },
    "newBalance": { "gold": 2114000, "gem": 320, "rune": 87 },
    "nextIdleAvailableAt": "2026-07-14T03:38:56Z"
  },
  "error": null,
  "meta": { "requestId": "req_g7", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Aturan server:**
- `elapsed = min(now - lastSeenAt, 86400)` detik.
- Bila `elapsed < 0` (clock skew) → `422 INVALID_IDLE_STATE`.
- `now` diambil dari server clock, bukan dari klien.
- Setelah klaim → `lastSeenAt = now`, `lastClaimAt = now`.
- Bila diklaim lagi dalam < 5 dtk → `409 ALREADY_CLAIMED`.

**Error:** `409` (sudah klaim <5dtk), `422` (state idle invalid).

> Penjelasan lengkap di §5.

---

### 3.2 Hero & Party

#### GET /api/v1/heroes

Daftar semua hero milik pemain.

**Auth:** Bearer. **Rate limit:** 60/user/menit.
**Query:** `?rarity=L&limit=20&cursor=...` (filter & pagination).

**Response 200:**
```json
{
  "data": [
    {
      "heroId": "her_001",
      "defId": "def_hero_archer",
      "name": "Elowen",
      "rarity": "LEGENDARY",
      "level": 60,
      "ascension": 4,
      "equip": { "weapon": "eq_w_12", "armor": null, "trinket": "eq_t_03" },
      "power": 48210,
      "acquiredAt": "2026-06-10T12:00:00Z"
    },
    {
      "heroId": "her_002",
      "defId": "def_hero_tank",
      "name": "Borin",
      "rarity": "EPIC",
      "level": 55,
      "ascension": 2,
      "equip": { "weapon": null, "armor": "eq_a_07", "trinket": null },
      "power": 39800,
      "acquiredAt": "2026-05-20T08:30:00Z"
    }
  ],
  "error": null,
  "meta": { "requestId": "req_h8", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0",
    "pagination": { "limit": 20, "nextCursor": "cur_xyz", "hasMore": true } }
}
```

---

#### POST /api/v1/heroes/{heroId}/ascend

Naik ascension hero (butuh material + gold). Server validasi kepemilikan & material.

**Auth:** Bearer. **Rate limit:** 10/user/menit. **Idempotency:** disarankan.

**Path param:** `heroId` (wajib).

**Request body:**
```json
{ "confirm": true }
```

**Response 200:**
```json
{
  "data": {
    "heroId": "her_001",
    "ascension": 5,
    "level": 60,
    "power": 51200,
    "consumed": {
      "gold": 500000,
      "materials": [ { "itemId": "mat_asc_core_l", "qty": 5 } ]
    },
    "newBalance": { "gold": 1614000, "gem": 320, "rune": 87 }
  },
  "error": null,
  "meta": { "requestId": "req_i9", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `404` (hero tak dimiliki), `409` (material/gold kurang), `422` (sudah max ascension).

---

#### PUT /api/v1/party

Set party aktif (maksimal 5 slot). Server-authoritative: validasi hero milik user & tidak duplikat.

**Auth:** Bearer. **Rate limit:** 30/user/menit.

**Request body:**
```json
{
  "slots": [
    { "slot": 0, "heroId": "her_001" },
    { "slot": 1, "heroId": "her_002" },
    { "slot": 2, "heroId": "her_007" },
    { "slot": 3, "heroId": "her_012" },
    { "slot": 4, "heroId": "her_015" }
  ]
}
```

Validasi:
- `slots` array panjang 1–5.
- `slot` 0–4 unik.
- semua `heroId` dimiliki user & tidak duplikat.
- hero dalam status tidak "lock" (mis. sedang di-marketplace).

**Response 200:**
```json
{
  "data": {
    "partyId": "pty_77",
    "slots": [
      { "slot": 0, "heroId": "her_001" },
      { "slot": 1, "heroId": "her_002" },
      { "slot": 2, "heroId": "her_007" },
      { "slot": 3, "heroId": "her_012" },
      { "slot": 4, "heroId": "her_015" }
    ],
    "totalPower": 223810
  },
  "error": null,
  "meta": { "requestId": "req_j10", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `409` (hero tidak dimiliki/duplikat/terkunci), `422` (slot invalid).

---

### 3.3 Gacha

Rates kanonikal: **C=60%, R=30%, E=9%, L=1%**. Pity: floor **R tiap 10 pull**, soft-pity mulai **75** (rate L naik bertahap), hard-pity **L di 90**.

#### GET /api/v1/gacha/banners

Daftar banner aktif (event/standard).

**Auth:** Bearer. **Rate limit:** 60/user/menit.

**Response 200:**
```json
{
  "data": [
    {
      "bannerId": "bn_std",
      "name": "Standard Banner",
      "type": "STANDARD",
      "rates": { "C": 0.60, "R": 0.30, "E": 0.09, "L": 0.01 },
      "featured": [],
      "startsAt": "2026-01-01T00:00:00Z",
      "endsAt": null,
      "costPerPull": { "currency": "gem", "amount": 160 }
    },
    {
      "bannerId": "bn_evt_01",
      "name": "Festival Elowen",
      "type": "EVENT",
      "rates": { "C": 0.60, "R": 0.30, "E": 0.09, "L": 0.01 },
      "featured": [ { "defId": "def_hero_archer", "rarity": "LEGENDARY", "rateUp": true } ],
      "startsAt": "2026-07-01T00:00:00Z",
      "endsAt": "2026-07-31T23:59:59Z",
      "costPerPull": { "currency": "gem", "amount": 160 }
    }
  ],
  "error": null,
  "meta": { "requestId": "req_k11", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

---

#### POST /api/v1/gacha/pull

Tarik gacha. `count` 1 atau 10. **Server resolve** rarity via RNG + pity state (server-side, tidak bisa dimanipulasi klien).

**Auth:** Bearer. **Rate limit:** 20/user/menit. **Idempotency:** disarankan (cegah double-charge).

**Request body:**
```json
{ "bannerId": "bn_evt_01", "count": 10 }
```

**Response 200:**
```json
{
  "data": {
    "bannerId": "bn_evt_01",
    "pulls": [
      { "index": 1, "heroDefId": "def_hero_c1", "rarity": "COMMON", "isNew": false },
      { "index": 2, "heroDefId": "def_hero_r3", "rarity": "RARE", "isNew": true },
      { "index": 3, "heroDefId": "def_hero_e2", "rarity": "EPIC", "isNew": true },
      { "index": 4, "heroDefId": "def_hero_c5", "rarity": "COMMON", "isNew": false },
      { "index": 5, "heroDefId": "def_hero_r1", "rarity": "RARE", "isNew": false },
      { "index": 6, "heroDefId": "def_hero_c9", "rarity": "COMMON", "isNew": false },
      { "index": 7, "heroDefId": "def_hero_r4", "rarity": "RARE", "isNew": true },
      { "index": 8, "heroDefId": "def_hero_c2", "rarity": "COMMON", "isNew": false },
      { "index": 9, "heroDefId": "def_hero_e1", "rarity": "EPIC", "isNew": false },
      { "index": 10, "heroDefId": "def_hero_archer", "rarity": "LEGENDARY", "isNew": true }
    ],
    "pity": {
      "bannerId": "bn_evt_01",
      "sinceRare": 0,
      "sinceLegendary": 0,
      "softPityActive": false
    },
    "consumed": { "gem": 1600 },
    "newBalance": { "gold": 2114000, "gem": -1280, "rune": 87 }
  },
  "error": null,
  "meta": { "requestId": "req_l12", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Aturan server (pity):**
- `sinceRare`: reset ke 0 bila dapat R/E/L. Bila mencapai 10 → pull berikutnya dijamin minimal R (floor).
- `sinceLegendary`: reset ke 0 bila dapat L. Bila ≥ 75 → `softPityActive`, rate L di-interpolasi naik hingga 100% di 90. Bila = 90 → guaranteed L (hard pity).
- RNG seed diambil dari sumber kriptografis server; tidak deterministik dari klien.
- `isNew` menandakan hero baru vs duplikat (duplikat → shards/convert ke currency).

**Error:** `404` (banner tidak ada/tidak aktif), `409` (currency kurang), `422` (count bukan 1/10).

---

#### GET /api/v1/gacha/history

Riwayat pull (filter by banner).

**Auth:** Bearer. **Rate limit:** 60/user/menit. **Pagination:** cursor.

**Query:** `?bannerId=bn_evt_01&limit=20&cursor=...`

**Response 200:**
```json
{
  "data": [
    { "pullId": "pl_001", "bannerId": "bn_evt_01", "heroDefId": "def_hero_archer", "rarity": "LEGENDARY", "at": "2026-07-14T03:00:00Z" },
    { "pullId": "pl_002", "bannerId": "bn_evt_01", "heroDefId": "def_hero_e1", "rarity": "EPIC", "at": "2026-07-13T20:00:00Z" }
  ],
  "error": null,
  "meta": { "requestId": "req_m13", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0",
    "pagination": { "limit": 20, "nextCursor": "cur_002", "hasMore": true } }
}
```

---

#### GET /api/v1/gacha/pity

State pity per banner.

**Auth:** Bearer. **Rate limit:** 60/user/menit.

**Response 200:**
```json
{
  "data": {
    "bn_std": { "sinceRare": 3, "sinceLegendary": 41, "softPityActive": false },
    "bn_evt_01": { "sinceRare": 0, "sinceLegendary": 0, "softPityActive": false }
  },
  "error": null,
  "meta": { "requestId": "req_n14", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

---

### 3.4 PvE (Stage & Battle)

Server-authoritative: klien kirim intent mulai battle; server resolve hasil wave (replay/state dikembalikan). Klien render.

#### GET /api/v1/stages

Daftar stage (chapter/zone), status clear.

**Auth:** Bearer. **Rate limit:** 60/user/menit. **Pagination:** cursor/offset.

**Query:** `?chapter=3&limit=50`

**Response 200:**
```json
{
  "data": [
    { "stageId": "stg_301", "chapter": 3, "index": 1, "name": "Gua Berkabut", "recommendedPower": 30000, "cleared": true, "stars": 3, "costStamina": 6 },
    { "stageId": "stg_302", "chapter": 3, "index": 2, "name": "Jembatan Runtuh", "recommendedPower": 32000, "cleared": false, "stars": 0, "costStamina": 6 }
  ],
  "error": null,
  "meta": { "requestId": "req_o15", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

---

#### POST /api/v1/stages/{stageId}/battle

Mulai run PvE. Server validasi stamina, partai, lalu **resolve** wave (simulasi server-side deterministik). Kembalikan `runId` + summary.

**Auth:** Bearer. **Rate limit:** 30/user/menit. **Idempotency:** wajib (cegah double-stamina).

**Path param:** `stageId`.
**Request body:**
```json
{ "partyId": "pty_77", "useStaminaPotion": false }
```

**Response 202 (Accepted — resolved async/queued):**
```json
{
  "data": {
    "runId": "run_9f2c1a",
    "stageId": "stg_302",
    "status": "RESOLVED",
    "outcome": "WIN",
    "staminaSpent": 6,
    "newStamina": 114,
    "rewards": {
      "gold": 4200,
      "exp": 2100,
      "items": [ { "itemId": "itm_chest_e", "qty": 1 } ],
      "stars": 3
    },
    "battleLogRef": "log_9f2c1a",
    "resolvedAt": "2026-07-14T03:38:56Z"
  },
  "error": null,
  "meta": { "requestId": "req_p16", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

> Mode async (BullMQ): bila stage berat, respons `202` dengan `status: QUEUED`, klien polling `GET /api/v1/battle/{runId}/result` atau dapat push WS `battle_resolved`. Untuk sebagian besar stage, resolve sinkron (202 + RESOLVED langsung).

**Error:** `404` (stage tak ada), `409` (stamina kurang / party invalid), `423` (stage belum terbuka).

---

#### GET /api/v1/battle/{runId}/result

Ambil hasil/state battle (replay data) untuk render klien.

**Auth:** Bearer. **Rate limit:** 60/user/menit.

**Path param:** `runId`.

**Response 200:**
```json
{
  "data": {
    "runId": "run_9f2c1a",
    "status": "RESOLVED",
    "outcome": "WIN",
    "turns": [
      { "turn": 1, "actor": "her_001", "action": "SKILL", "target": "mob_02", "damage": 12400, "crit": true },
      { "turn": 2, "actor": "mob_01", "action": "ATTACK", "target": "her_002", "damage": 3200, "crit": false }
    ],
    "rewards": { "gold": 4200, "exp": 2100, "items": [ { "itemId": "itm_chest_e", "qty": 1 } ], "stars": 3 },
    "seed": 88213745
  },
  "error": null,
  "meta": { "requestId": "req_q17", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `404` (run tidak ada), `403` (bukan run milik user).

---

### 3.5 Inventory & Equipment

#### GET /api/v1/inventory

Daftar item milik pemain (equipment, material, consumable).

**Auth:** Bearer. **Rate limit:** 60/user/menit. **Pagination:** cursor.

**Query:** `?type=EQUIPMENT&rarity=R&limit=50`

**Response 200:**
```json
{
  "data": [
    { "invId": "inv_001", "itemId": "eq_w_12", "defId": "def_eq_blade", "name": "Pedang Kabut", "type": "EQUIPMENT", "rarity": "RARE", "level": 12, "qty": 1, "equippedBy": "her_001" },
    { "invId": "inv_002", "itemId": "mat_asc_core_l", "defId": "def_mat_core", "name": "Inti Ascensi", "type": "MATERIAL", "rarity": "LEGENDARY", "qty": 5, "equippedBy": null }
  ],
  "error": null,
  "meta": { "requestId": "req_r18", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0",
    "pagination": { "limit": 50, "nextCursor": null, "hasMore": false } }
}
```

---

#### POST /api/v1/equip

Pasang/lepas equipment ke hero.

**Auth:** Bearer. **Rate limit:** 30/user/menit.

**Request body:**
```json
{
  "heroId": "her_001",
  "slot": "weapon",
  "invId": "inv_001"
}
```
- `invId: null` → lepas.

**Response 200:**
```json
{
  "data": {
    "heroId": "her_001",
    "equip": { "weapon": "inv_001", "armor": null, "trinket": "inv_009" },
    "powerDelta": 2310,
    "newPower": 50520
  },
  "error": null,
  "meta": { "requestId": "req_s19", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `404` (hero/inv tak dimiliki), `409` (equip dipakai hero lain / tier tidak cocok), `422` (slot invalid).

---

### 3.6 Marketplace

Fee platform **5%**. Listing memasukkan item ke **escrow** (server tahan kepemilikan sampai terjual/batal). Semua mutasi butuh `Idempotency-Key`.

#### GET /api/v1/market/listings

Cari listing (filter rarity/price/type).

**Auth:** Bearer. **Rate limit:** 120/user/menit. **Pagination:** cursor.

**Query:** `?rarity=LEGENDARY&minPrice=1000&maxPrice=50000&currency=gem&sort=price_asc&limit=20&cursor=...`

**Response 200:**
```json
{
  "data": [
    {
      "listingId": "lst_001",
      "sellerId": "usr_22",
      "itemId": "eq_w_40",
      "name": "Excalibur Tirta",
      "rarity": "LEGENDARY",
      "price": { "currency": "gem", "amount": 25000 },
      "listedAt": "2026-07-14T01:00:00Z"
    },
    {
      "listingId": "lst_002",
      "sellerId": "usr_31",
      "itemId": "her_099",
      "name": "Hero: Seraphina",
      "rarity": "LEGENDARY",
      "price": { "currency": "gem", "amount": 40000 },
      "listedAt": "2026-07-14T02:30:00Z"
    }
  ],
  "error": null,
  "meta": { "requestId": "req_t20", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0",
    "pagination": { "limit": 20, "nextCursor": "cur_003", "hasMore": true } }
}
```

---

#### POST /api/v1/market/listings

Buat listing → item masuk escrow.

**Auth:** Bearer. **Rate limit:** 20/user/menit. **Idempotency:** **wajib**.

**Request body:**
```json
{
  "invId": "inv_050",
  "price": { "currency": "gem", "amount": 25000 }
}
```

**Response 201:**
```json
{
  "data": {
    "listingId": "lst_003",
    "invId": "inv_050",
    "price": { "currency": "gem", "amount": 25000 },
    "escrow": true,
    "status": "LISTED",
    "listedAt": "2026-07-14T03:38:56Z",
    "expiresAt": "2026-07-21T03:38:56Z"
  },
  "error": null,
  "meta": { "requestId": "req_u21", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `404` (item tak dimiliki), `409` (item sedang dipakai/di-escrow), `422` (harga di luar batas / currency tidak boleh gold), `429` (batas listing aktif).

---

#### POST /api/v1/market/buy/{listingId}

Beli listing. Server tahan currency pembeli, transfer item, potong **fee 5%** ke seller.

**Auth:** Bearer. **Rate limit:** 20/user/menit. **Idempotency:** **wajib**.

**Path param:** `listingId`.
**Request body:** `{}`.

**Response 200:**
```json
{
  "data": {
    "listingId": "lst_003",
    "buyerId": "usr_9a1b2c3d",
    "sellerId": "usr_22",
    "price": { "currency": "gem", "amount": 25000 },
    "fee": { "currency": "gem", "amount": 1250 },
    "sellerReceived": { "currency": "gem", "amount": 23750 },
    "itemTransferred": { "invId": "inv_050", "toHeroInventory": true },
    "buyerBalance": { "gold": 2114000, "gem": -25250, "rune": 87 },
    "txnId": "txn_mkt_001",
    "at": "2026-07-14T03:38:56Z"
  },
  "error": null,
  "meta": { "requestId": "req_v22", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

> Perhitungan fee: `fee = floor(price * 0.05)`, `sellerReceived = price - fee`. Ledger ganda (double-entry) di PostgreSQL (lihat ERD §9).

**Error:** `404` (listing tidak ada), `409` (currency pembeli kurang / beli item sendiri), `410` (listing sudah laku/batal), `429` (rate limit).

---

#### POST /api/v1/market/cancel/{listingId}

Batalkan listing → item kembali dari escrow ke inventory.

**Auth:** Bearer. **Rate limit:** 20/user/menit. **Idempotency:** disarankan.

**Path param:** `listingId`.
**Request body:** `{}`.

**Response 200:**
```json
{
  "data": {
    "listingId": "lst_003",
    "status": "CANCELLED",
    "itemReturned": { "invId": "inv_050", "toInventory": true },
    "at": "2026-07-14T03:38:56Z"
  },
  "error": null,
  "meta": { "requestId": "req_w23", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `404` (listing tak ada), `403` (bukan milik user), `409` (sudah terjual).

---

### 3.7 Guild & Raid

#### POST /api/v1/guilds

Buat guild.

**Auth:** Bearer. **Rate limit:** 5/user/menit. **Idempotency:** disarankan.

**Request body:**
```json
{ "name": "Naga Biru", "tag": "NGB", "description": "Guild PvP serius" }
```

**Response 201:**
```json
{
  "data": {
    "guildId": "gld_55",
    "name": "Naga Biru",
    "tag": "NGB",
    "ownerId": "usr_9a1b2c3d",
    "memberCount": 1,
    "maxMembers": 30,
    "createdAt": "2026-07-14T03:38:56Z"
  },
  "error": null,
  "meta": { "requestId": "req_x24", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `409` (tag/nama dup), `422` (format invalid), `409` (sudah di guild lain).

---

#### POST /api/v1/guilds/{guildId}/join

Gabung guild (langsung bila open, via request bila private).

**Auth:** Bearer. **Rate limit:** 10/user/menit.

**Path param:** `guildId`.
**Request body:** `{}` (atau `{"message":"terima saya"}` bila private).

**Response 200:**
```json
{
  "data": {
    "guildId": "gld_55",
    "status": "JOINED",
    "memberCount": 13,
    "joinedAt": "2026-07-14T03:38:56Z"
  },
  "error": null,
  "meta": { "requestId": "req_y25", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `404` (guild tak ada), `409` (sudah di guild / penuh), `403` (private butuh approve).

---

#### POST /api/v1/guilds/{guildId}/raids/{raidId}/start

Mulai guild raid. Server resolve, bagi reward ke peserta (via BullMQ settlement).

**Auth:** Bearer (harus anggota, role LEADER/OFFICER untuk start). **Rate limit:** 5/user/menit. **Idempotency:** wajib.

**Path param:** `guildId`, `raidId`.
**Request body:**
```json
{ "participants": ["usr_9a1b2c3d", "usr_22", "usr_31"], "partyId": "pty_77" }
```

**Response 202:**
```json
{
  "data": {
    "raidRunId": "raidrun_77",
    "guildId": "gld_55",
    "raidId": "raid_01",
    "status": "QUEUED",
    "participants": 3,
    "estimatedSettleSec": 30
  },
  "error": null,
  "meta": { "requestId": "req_z26", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `404` (guild/raid tak ada), `403` (bukan officer), `409` (raid cooldown aktif / stamina kurang).

---

### 3.8 PvP & Leaderboard

ELO awal **1000**. Matchmaking berbasis ELO (±band). Hasil di-resolve server (replay/state) untuk fairness.

#### GET /api/v1/pvp/ladder

Ambil snapshot ladder PvP (top + posisi user).

**Auth:** Bearer. **Rate limit:** 60/user/menit. **Pagination:** offset.

**Query:** `?scope=global&limit=100&offset=0`

**Response 200:**
```json
{
  "data": {
    "userRank": 412,
    "userElo": 1240,
    "entries": [
      { "rank": 1, "userId": "usr_top1", "displayName": "GodSlayer", "elo": 2140, "winStreak": 18 },
      { "rank": 2, "userId": "usr_top2", "displayName": "Moonlit", "elo": 2095, "winStreak": 4 }
    ],
    "totalPlayers": 58321
  },
  "error": null,
  "meta": { "requestId": "req_aa27", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

---

#### POST /api/v1/pvp/match

Cari match PvP. Server cari lawan se-ELO (band ±100 awal, melebar tiap 5 dtk tunggu). Kembalikan `matchId` + kedua party (hati-hati: party lawan di-mask ke stat ringkas, bukan full inventory).

**Auth:** Bearer. **Rate limit:** 20/user/menit. **Idempotency:** disarankan.

**Request body:**
```json
{ "partyId": "pty_77", "mode": "RANKED" }
```

**Response 200 (match ditemukan):**
```json
{
  "data": {
    "matchId": "pvp_m_5521",
    "mode": "RANKED",
    "status": "MATCHED",
    "opponent": {
      "userId": "usr_22",
      "displayName": "BladeMaster",
      "elo": 1235,
      "partyPower": 221000,
      "heroes": [ { "defId": "def_hero_tank", "rarity": "LEGENDARY", "level": 60 } ]
    },
    "yourElo": 1240,
    "foundInSec": 3,
    "resolveBy": "2026-07-14T03:39:26Z"
  },
  "error": null,
  "meta": { "requestId": "req_bb28", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

> Bila 30 dtk tak ketemu → `202` dengan `status: SEARCHING` + `ticketId`; klien dapat push WS `pvp_match_found` (§4).

**Error:** `409` (sedang dalam match/penalti), `423` (party invalid/kosong).

---

#### POST /api/v1/pvp/match/{matchId}/submit

Kirim hasil/aksi battle PvP (klien kirim input; server yang resolve final state untuk cegah cheating). *Opsional bila mode full-server-resolve.*

**Auth:** Bearer. **Rate limit:** 10/user/menit.

**Request body:**
```json
{
  "matchId": "pvp_m_5521",
  "actions": [ { "turn": 1, "actor": "her_001", "action": "SKILL", "targetSlot": 2 } ],
  "clientSeed": 12345
}
```

**Response 200:**
```json
{
  "data": {
    "matchId": "pvp_m_5521",
    "status": "RESOLVED",
    "outcome": "WIN",
    "eloDelta": 18,
    "newElo": 1258,
    "reward": { "rune": 15, "gold": 2000 },
    "battleLogRef": "log_pvp_5521"
  },
  "error": null,
  "meta": { "requestId": "req_cc29", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `404` (match tak ada), `409` (sudah submit / bukan peserta), `422` (format aksi invalid), `410` (match expired).

---

#### GET /api/v1/leaderboard

Leaderboard global (power/exp/idle). Tidak butuh filter berat.

**Auth:** Bearer. **Rate limit:** 60/user/menit. **Pagination:** offset.

**Query:** `?category=POWER&scope=global&limit=100&offset=0`

**Response 200:**
```json
{
  "data": {
    "category": "POWER",
    "entries": [
      { "rank": 1, "userId": "usr_top1", "displayName": "GodSlayer", "value": 512000 },
      { "rank": 2, "userId": "usr_top2", "displayName": "Moonlit", "value": 498200 }
    ],
    "userRank": 412,
    "userValue": 223810,
    "total": 58321
  },
  "error": null,
  "meta": { "requestId": "req_dd30", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

> Kategori: `POWER` | `EXP` | `IDLE_GOLD` | `PVE_STAGE` | `GUILD`.

---

### 3.9 Events

#### GET /api/v1/events

Daftar event aktif + progres user.

**Auth:** Bearer. **Rate limit:** 60/user/menit.

**Response 200:**
```json
{
  "data": [
    {
      "eventId": "evt_summer",
      "name": "Festival Musim Panas",
      "type": "LOGIN_STREAK",
      "startsAt": "2026-07-01T00:00:00Z",
      "endsAt": "2026-07-31T23:59:59Z",
      "progress": { "current": 12, "goal": 14, "claimed": [1,2,3,4,5,6,7,8,9,10,11,12] }
    }
  ],
  "error": null,
  "meta": { "requestId": "req_ee31", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

---

#### POST /api/v1/events/{eventId}/claim

Klaim reward event (mis. daily login). Server validasi progres.

**Auth:** Bearer. **Rate limit:** 20/user/menit. **Idempotency:** disarankan.

**Path param:** `eventId`.
**Request body:**
```json
{ "step": 13 }
```

**Response 200:**
```json
{
  "data": {
    "eventId": "evt_summer",
    "claimedStep": 13,
    "reward": { "gem": 100, "items": [ { "itemId": "itm_chest_l", "qty": 1 } ] },
    "newBalance": { "gold": 2114000, "gem": 420, "rune": 87 }
  },
  "error": null,
  "meta": { "requestId": "req_ff32", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `404` (event tak ada), `409` (step belum capai / sudah diklaim), `410` (event berakhir).

---

### 3.10 Shop & Monetization

#### GET /api/v1/shop/products

Daftar produk (sku) tersedia. Harga riil (IDR/USD) dikelola Payment Provider; klien dapat `productId` lalu redirect ke provider.

**Auth:** Bearer. **Rate limit:** 60/user/menit.

**Response 200:**
```json
{
  "data": [
    { "productId": "sku_gem_100", "name": "100 Gem", "currencyGranted": "gem", "amount": 100, "price": { "currency": "USD", "amount": 0.99 }, "region": "GLOBAL" },
    { "productId": "sku_gem_1000", "name": "1000 Gem", "currencyGranted": "gem", "amount": 1000, "price": { "currency": "USD", "amount": 9.99 }, "region": "GLOBAL" },
    { "productId": "sku_pass_month", "name": "Monthly Pass", "currencyGranted": "gem", "amount": 300, "price": { "currency": "USD", "amount": 4.99 }, "region": "GLOBAL" }
  ],
  "error": null,
  "meta": { "requestId": "req_gg33", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

---

#### POST /api/v1/shop/purchase

Inisiasi pembelian → buat order, kembalikan URL/ token provider. **Idempotency wajib** (mencegah double-charge).

**Auth:** Bearer. **Rate limit:** 10/user/menit. **Idempotency:** **wajib**.

**Request body:**
```json
{ "productId": "sku_gem_1000", "provider": "STRIPE", "returnUrl": "https://play.example.com/store/return" }
```

**Response 200:**
```json
{
  "data": {
    "orderId": "ord_5521",
    "productId": "sku_gem_1000",
    "amount": { "currency": "USD", "amount": 9.99 },
    "checkoutUrl": "https://pay.stripe.com/checkout/xxx",
    "status": "PENDING",
    "expiresAt": "2026-07-14T03:53:56Z"
  },
  "error": null,
  "meta": { "requestId": "req_hh34", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `404` (produk tak ada), `409` (region tidak cocok), `422` (provider tidak didukung).

---

#### POST /api/v1/shop/webhook

Webhook dari Payment Provider (STRIPE/XENDIT/dst). **TIDAK butuh Bearer**; verifikasi via signature header (`X-Provider-Signature`).

**Auth:** Signature provider. **Rate limit:** 100/ip/menit.

**Request header:** `X-Provider-Signature: sha256=...`, `X-Provider: STRIPE`.
**Request body (raw, sesuai provider):**
```json
{
  "orderId": "ord_5521",
  "providerTxnId": "txn_stripe_abc",
  "status": "PAID",
  "amount": { "currency": "USD", "amount": 9.99 }
}
```

**Response 200:**
```json
{ "data": { "received": true }, "error": null, "meta": { "requestId": "req_ii35", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" } }
```

> Setelah verifikasi signature & idempotensi `orderId`, server grant `gem` via ledger dan kirim notif WS `notification`. Webhook harus idempoten (provider bisa retry).

**Error:** `401` (signature invalid), `409` (order sudah diproses), `422` (payload invalid).

---

### 3.11 Admin / GM

Semua endpoint ini **wajib** role `admin` atau `gm`. Setiap aksi menulis **audit log** (lihat §8.5).

#### POST /api/v1/admin/heroes

Buat definisi hero baru (master data).

**Auth:** Admin. **Rate limit:** 10/admin/menit.

**Request body:**
```json
{
  "defId": "def_hero_new",
  "name": "Aurora",
  "rarity": "MYTHIC",
  "basePower": 5000,
  "element": "LIGHT",
  "role": "SUPPORT"
}
```

**Response 201:**
```json
{ "data": { "defId": "def_hero_new", "created": true }, "error": null, "meta": { "requestId": "req_jj36", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" } }
```

---

#### PUT /api/v1/admin/gacha/banners/{bannerId}/rate

Ubah rate banner (GM tuning). Validasi: total rate = 1.0.

**Auth:** Admin. **Rate limit:** 10/admin/menit.

**Path param:** `bannerId`.
**Request body:**
```json
{ "rates": { "C": 0.55, "R": 0.33, "E": 0.10, "L": 0.02 } }
```

**Response 200:**
```json
{ "data": { "bannerId": "bn_evt_01", "rates": { "C": 0.55, "R": 0.33, "E": 0.10, "L": 0.02 }, "updatedAt": "2026-07-14T03:38:56Z" }, "error": null, "meta": { "requestId": "req_kk37", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" } }
```

**Error:** `422` (total rate ≠ 1.0), `404` (banner tak ada).

---

#### POST /api/v1/admin/grant

GM grant currency/item ke user (event/compen).

**Auth:** Admin. **Rate limit:** 20/admin/menit. **Idempotency:** wajib (audit).

**Request body:**
```json
{
  "userId": "usr_9a1b2c3d",
  "grant": { "gold": 1000000, "gem": 500, "items": [ { "itemId": "itm_chest_l", "qty": 2 } ] },
  "reason": "Compensation server downtime 2026-07-13"
}
```

**Response 200:**
```json
{
  "data": {
    "userId": "usr_9a1b2c3d",
    "granted": { "gold": 1000000, "gem": 500, "items": [ { "itemId": "itm_chest_l", "qty": 2 } ] },
    "newBalance": { "gold": 3114000, "gem": 920, "rune": 87 },
    "auditId": "aud_77"
  },
  "error": null,
  "meta": { "requestId": "req_ll38", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

**Error:** `404` (user tak ada), `422` (jumlah negatif).

---

#### POST /api/v1/admin/ban

Ban/unban user.

**Auth:** Admin. **Rate limit:** 10/admin/menit.

**Request body:**
```json
{ "userId": "usr_99", "action": "BAN", "durationDays": 7, "reason": "Cheating (speed hack)" }
```

**Response 200:**
```json
{ "data": { "userId": "usr_99", "status": "BANNED", "until": "2026-07-21T03:38:56Z", "auditId": "aud_78" }, "error": null, "meta": { "requestId": "req_mm39", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" } }
```

**Error:** `404` (user tak ada), `409` (sudah dalam status tersebut).

---

### 3.12 GDPR / PDPA (Data Subject Rights)

Sesuai compliance, user dapat export & delete data diri.

#### GET /api/v1/account/export

Export seluruh data user (GDPR Art. 20 / PDPA). Generate file via BullMQ; notif saat siap.

**Auth:** Bearer. **Rate limit:** 2/user/hari.

**Response 202:**
```json
{ "data": { "exportRequested": true, "estimatedSec": 120, "status": "QUEUED" }, "error": null, "meta": { "requestId": "req_nn40", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" } }
```

> Saat siap → notif WS `notification` + URL download signed (CDN, TTL 1 jam).

---

#### DELETE /api/v1/account

Hapus akun & anonymisasi data (GDPR right to erasure / PDPA). Soft-delete + anonymize ledger (jangan hapus row ledger demi audit finansial — anonymize `userId`).

**Auth:** Bearer. **Rate limit:** 1/user/hari. **Idempotency:** wajib.

**Request body:**
```json
{ "confirm": true, "reason": "User request" }
```

**Response 200:**
```json
{ "data": { "deleted": true, "anonymizedAt": "2026-07-14T03:38:56Z", "retentionNote": "Ledger finansial di-anonimize, tidak dihapus (kewajiban audit)." }, "error": null, "meta": { "requestId": "req_oo41", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" } }
```

> Setelah delete → revoke semua token, clear cookie, logout paksa.

---

## 4. WebSocket

### 4.1 Koneksi

```
wss://api.example.com/ws?token=<access_jwt>
```

- Upgrade auth: server verifikasi `token` (access JWT) saat handshake. Gagal → tutup koneksi (code 4401).
- Satu koneksi per `userId` (device lain → koneksi sebelumnya di-close code 4409, "new connection").
- Heartbeat: client ping tiap 25 dtk, server pong. Timeout 60 dtk → disconnect.
- Reconnect: exponential backoff (1s, 2s, 4s … max 30s).

### 4.2 Envelope Pesan

Semua pesan (client→server & server→client) pakai envelope:

```json
{
  "type": "EVENT_NAME",
  "payload": { },
  "ts": 1752459536,
  "nonce": "abc123"
}
```

- `type` — nama event (lihat di bawah).
- `payload` — data event.
- `ts` — epoch ms (untuk anti-replay di sisi server, lihat §8.4).
- `nonce` — string unik (anti-replay).

### 4.3 Events Server → Client

#### `leaderboard_push`

Push update ladder/leaderboard (top berubah / user naik rank).

```json
{
  "type": "leaderboard_push",
  "payload": {
    "category": "POWER",
    "scope": "global",
    "userRank": 405,
    "userValue": 226000,
    "topChanged": true,
    "top": [ { "rank": 1, "userId": "usr_top1", "displayName": "GodSlayer", "value": 513000 } ]
  },
  "ts": 1752459536,
  "nonce": "lb_001"
}
```

#### `pvp_match_found`

Lawan ditemukan (respon atas `POST /pvp/match` search async).

```json
{
  "type": "pvp_match_found",
  "payload": {
    "ticketId": "tk_99",
    "matchId": "pvp_m_5521",
    "opponent": { "userId": "usr_22", "displayName": "BladeMaster", "elo": 1235, "partyPower": 221000 },
    "yourElo": 1240,
    "foundInSec": 7,
    "resolveBy": "2026-07-14T03:39:33Z"
  },
  "ts": 1752459536,
  "nonce": "mf_001"
}
```

#### `pvp_battle_sync`

Sinkronisasi state/replay battle PvP (tiap turn atau snapshot periodik). Server-authoritative.

```json
{
  "type": "pvp_battle_sync",
  "payload": {
    "matchId": "pvp_m_5521",
    "turn": 4,
    "state": {
      "yourParty": [ { "slot": 0, "hp": 8200, "maxHp": 12000, "energy": 3 } ],
      "oppParty": [ { "slot": 0, "hp": 5400, "maxHp": 11000, "energy": 2 } ]
    },
    "lastAction": { "actor": "her_001", "action": "SKILL", "targetSlot": 0, "damage": 9800, "crit": true },
    "seed": 88213745
  },
  "ts": 1752459536,
  "nonce": "bs_001"
}
```

#### `notification`

Notifikasi umum (reward event siap, export data siap, pesan guild, dll).

```json
{
  "type": "notification",
  "payload": {
    "id": "ntf_001",
    "kind": "REWARD_READY",
    "title": "Data Export Siap",
    "body": "File export data Anda dapat diunduh.",
    "data": { "downloadUrl": "https://cdn.example.com/export/usr_9a1b2c3d.zip?sig=...", "ttlSec": 3600 },
    "createdAt": "2026-07-14T03:40:00Z"
  },
  "ts": 1752459536,
  "nonce": "nt_001"
}
```

#### `idle_claim_result`

Hasil klaim idle (bila diklaim via WS, mis. auto-claim saat buka app).

```json
{
  "type": "idle_claim_result",
  "payload": {
    "claimedSec": 86400,
    "capped": true,
    "reward": { "gold": 864000, "exp": 43200, "items": [ { "itemId": "itm_idle_chest_c", "qty": 2 } ] },
    "newBalance": { "gold": 2978000, "gem": 320, "rune": 87 }
  },
  "ts": 1752459536,
  "nonce": "ic_001"
}
```

#### `guild_announcement`

Pengumuman dari officer/leader guild.

```json
{
  "type": "guild_announcement",
  "payload": {
    "guildId": "gld_55",
    "authorId": "usr_22",
    "message": "Raid malam ini jam 20:00, siapkan party!",
    "createdAt": "2026-07-14T03:30:00Z"
  },
  "ts": 1752459536,
  "nonce": "ga_001"
}
```

### 4.4 Events Client → Server (contoh)

Klien dapat mengirim intent terbatas (mis. ping, atau submit aksi PvP). Semua butuh `ts`+`nonce` (anti-replay §8.4).

```json
{ "type": "pvp_action", "payload": { "matchId": "pvp_m_5521", "action": { "turn": 5, "actor": "her_001", "action": "ATTACK", "targetSlot": 1 } }, "ts": 1752459536, "nonce": "ca_001" }
```

```json
{ "type": "ping", "payload": {}, "ts": 1752459536, "nonce": "ca_002" }
```

### 4.5 Kode Tutup WS (Close Code)

| Code | Makna |
|---|---|
| 4401 | Unauthorized (token invalid/expired). |
| 4403 | Forbidden (role tidak cukup). |
| 4409 | Connection superseded (koneksi baru dari device sama). |
| 4410 | Server shutdown (maintenance). |
| 1000 | Normal close. |
| 1001 | Going away (klien tutup/tab pindah). |
| 1013 | Try again later (server overload). |

---

## 5. Mekanisme Idle Claim

Server-authoritative. Klien **tidak** menghitung reward; hanya mengirim trigger dan menampilkan hasil.

### 5.1 Algoritma Server

```
FUNGSI claimIdle(userId):
  player = SELECT * FROM players WHERE id = userId
  now = serverNow()                      # clock server, BUKAN klien
  elapsedSec = now - player.last_seen_at # detik
  IF elapsedSec < 0: RAISE 422 INVALID_IDLE_STATE
  IF now - player.last_claim_at < 5: RAISE 409 ALREADY_CLAIMED

  accruedSec = CLAMP(elapsedSec, 0, 86400)   # cap 24 jam
  capped = elapsedSec > 86400

  goldRate   = IDLE_GOLD_PER_SEC(player.level)   # mis. level*20/dt
  expRate    = IDLE_EXP_PER_SEC(player.level)
  gold = floor(accruedSec * goldRate)
  exp  = floor(accruedSec * expRate)
  items = rollIdleChest(accruedSec)             # chance chest tiap N jam

  BEGIN TXN:
    UPDATE players SET gold += gold, exp += exp,
                       last_seen_at = now, last_claim_at = now
    INSERT ledger (idle claim)
    (optional) INSERT inventory items
  COMMIT

  RETURN { claimedSec: accruedSec, capped, reward:{gold,exp,items}, newBalance }
```

### 5.2 Tick & Cap

- **Tick idle:** server mengakuakumulasi tiap **5 detik** (interval komputasi internal; tidak perlu request klien tiap 5 dtk).
- **Cap:** akrual di-clamp maksimal **86400 detik (24 jam)**. Melebihi itu → `capped: true`, reward dihitung untuk 24j saja.
- `last_seen_at` diperbarui setiap kali user melakukan aktivitas API apa pun (bukan hanya idle claim), sehingga idle dihitung sejak terakhir aktif.

### 5.3 Contoh Skenario

| Skenario | elapsedSec | accruedSec | capped | reward |
|---|---|---|---|---|
| Baru 10 mnt | 600 | 600 | false | 600 × rate |
| Offline 8 jam | 28800 | 28800 | false | 28800 × rate |
| Offline 30 jam | 108000 | 86400 | true | 86400 × rate (sisa hangus) |

### 5.4 Endpoint

`POST /api/v1/claim-idle` (detail di §3.1). Juga tersedia via WS `idle_claim_result` bila klien memilih jalur WS.

---

## 6. Kode Error

### 6.1 Envelope Error

Saat gagal, `data: null`, `error` terisi:

```json
{
  "data": null,
  "error": {
    "code": "INVALID_CREDENTIALS",
    "httpStatus": 401,
    "message": "Email atau password salah.",
    "details": { "field": "password" },
    "requestId": "req_err_01"
  },
  "meta": { "requestId": "req_err_01", "timestamp": "2026-07-14T03:38:56Z", "version": "1.0.0" }
}
```

### 6.2 Tabel Kode

| HTTP | code | Pesan (id) | Penjelasan |
|---|---|---|---|
| 400 | BAD_REQUEST | Permintaan tidak valid. | Malformed JSON / query. |
| 401 | UNAUTHORIZED | Tidak terautentikasi. | Access token hilang/expired. |
| 401 | INVALID_CREDENTIALS | Email atau password salah. | Login gagal. |
| 401 | TOKEN_EXPIRED | Token kadaluarsa. | Access JWT lewat exp. |
| 401 | REFRESH_INVALID | Refresh token tidak valid. | Cookie refresh rusak. |
| 403 | FORBIDDEN | Akses ditolak. | Role tidak cukup. |
| 403 | REUSE_DETECTED | Refresh reuse terdeteksi. | Semua sesi di-revoke. |
| 403 | ACCOUNT_BANNED | Akun diban. | User diban GM. |
| 404 | NOT_FOUND | Tidak ditemukan. | Resource tak ada. |
| 409 | CONFLICT | Konflik data. | Duplikat / state bentrok. |
| 409 | INSUFFICIENT_CURRENCY | Currency tidak cukup. | Gold/gem/rune kurang. |
| 409 | INSUFFICIENT_RESOURCE | Resource tidak cukup. | Material/stamina kurang. |
| 409 | ALREADY_CLAIMED | Sudah diklaim. | Idle/event sudah diambil. |
| 409 | EMAIL_TAKEN | Email sudah dipakai. | Register. |
| 410 | GONE | Resource sudah tidak tersedia. | Listing laku / event berakhir. |
| 422 | VALIDATION_ERROR | Validasi gagal. | Field tidak lolos aturan. |
| 422 | INVALID_IDLE_STATE | State idle tidak valid. | clock skew / negatif. |
| 423 | LOCKED | Terkunci sementara. | Akun/resource lock (brute-force). |
| 429 | RATE_LIMITED | Terlalu banyak permintaan. | Melebihi rate limit. |
| 500 | INTERNAL_ERROR | Kesalahan server. | Bug tak terduga. |
| 502 | BAD_GATEWAY | Gateway error. | Upstream (DB/queue) gagal. |
| 503 | SERVICE_UNAVAILABLE | Layanan tidak tersedia. | Maintenance. |
| 504 | TIMEOUT | Permintaan timeout. | Job terlalu lama. |

### 6.3 Detail Field `details`

Untuk `422`, sertakan array masalah:

```json
"details": {
  "fields": [
    { "field": "password", "rule": "min_length", "message": "Minimal 10 karakter." },
    { "field": "email", "rule": "format", "message": "Format email tidak valid." }
  ]
}
```

---

## 7. Rate Limits & Pagination

### 7.1 Rate Limit Headers

Setiap respons menyertakan:

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 58
X-RateLimit-Reset: 1752459596
```

- `Limit` — kuota per window.
- `Remaining` — sisa.
- `Reset` — epoch detik saat window reset.

Bila lewat → `429` + header `Retry-After: <detik>`.

### 7.2 Strategi Per-Endpoint

| Endpoint grup | Limit | Window |
|---|---|---|
| /auth/login | 20 | 1 mnt / ip |
| /auth/register | 10 | 1 mnt / ip |
| /auth/refresh | 30 | 1 mnt / ip |
| /me, /progression | 60 | 1 mnt / user |
| /gacha/pull | 20 | 1 mnt / user |
| /market/* | 20 | 1 mnt / user |
| /pvp/match | 20 | 1 mnt / user |
| /shop/purchase | 10 | 1 mnt / user |
| /admin/* | 10–20 | 1 mnt / admin |
| /account/export | 2 | 1 hari / user |
| Static/CDN | 1000 | 1 mnt / ip |

> Limit dapat berbeda per tier (VIP). Nilai di atas baseline.

### 7.3 Pagination

Dua mode didukung.

**A. Cursor (direkomendasikan untuk list besar):**
```
GET /api/v1/market/listings?limit=20&cursor=cur_003
```
Respons `meta.pagination`:
```json
{ "limit": 20, "nextCursor": "cur_004", "hasMore": true }
```
- `cursor` opaque (jangan di-parse klien).
- Urutan stabil (server pakai `(created_at, id)`).

**B. Offset (untuk leaderboard/ladder):**
```
GET /api/v1/leaderboard?category=POWER&limit=100&offset=0
```
Respons `meta.pagination`:
```json
{ "limit": 100, "offset": 0, "total": 58321, "hasMore": true }
```

**Batas:** `limit` maksimal 100 (default 20). `offset` maksimal 10000.

---

## 8. Keamanan (Security)

### 8.1 HTTPS & Transport

- **HTTPS wajib** seluruhnya (REST + WS `wss://`). HTTP redirect 301 ke HTTPS.
- HSTS header: `Strict-Transport-Security: max-age=31536000; includeSubDomains`.
- TLS minimal 1.2; prefer 1.3.
- Sertifikat rotasi otomatis (ACME).

### 8.2 CORS

- `Access-Control-Allow-Origin` di-whitelist ke domain web resmi (`https://play.example.com`, `https://www.example.com`). Tidak pakai `*`.
- `Access-Control-Allow-Credentials: true` (diperlukan karena cookie refresh).
- Method diizinkan: `GET, POST, PUT, PATCH, DELETE, OPTIONS`.
- Header diizinkan: `Authorization, Content-Type, Idempotency-Key, X-Request-Timestamp, X-Nonce, X-Client-Version, X-Platform`.
- Preflight `OPTIONS` di-cache: `Access-Control-Max-Age: 86400`.
- Cookie refresh `SameSite=Strict` → tidak dikirim lintas-site; CORS credential aman.

### 8.3 Idempotency Key

Operasi mutasi berbayar (marketplace listing/buy/cancel, shop purchase, admin grant) **wajib** header:

```
Idempotency-Key: 4b1c2d3e-5f6a-7b8c-9d0e-1f2a3b4c5d6e
```

- UUID v4 unik per intent bisnis.
- Server simpan `idempotencyKey → response` (TTL 24j di Redis).
- Request ulang dengan key sama → kembalikan respons tersimpan (bukan eksekusi ganda).
- Key sama dengan body beda → `422 IDEMPOTENCY_MISMATCH`.

### 8.4 Anti-Replay (WS & mutasi kritis)

Untuk pesan WS dan mutasi kritis, klien kirim:

```
X-Request-Timestamp: 1752459536   # epoch ms
X-Nonce: a1b2c3d4                  # unik per request
```

**Validasi server:**
- `ts` dalam window ±30 detik dari server now (tolak terlalu tua/ masa depan jauh).
- `nonce` belum pernah dipakai `userId+ts` (simpan di Redis TTL 60 dtk).
- Bila `ts` kadaluarsa atau `nonce` reuse → `401 REPLAY_DETECTED`.

> Mencegah attacker capture & replay pesan (mis. `pvp_action` berulang).

### 8.5 Audit Log Admin

Setiap aksi `/admin/*` (grant, ban, rate change, hero create) ditulis ke tabel `admin_audit_log`:

| Kolom | Tipe | Keterangan |
|---|---|---|
| `audit_id` | UUID | PK. |
| `admin_id` | UUID | GM pelaku. |
| `action` | VARCHAR | `GRANT`, `BAN`, `RATE_CHANGE`, `HERO_CREATE`. |
| `target_user_id` | UUID | nullable. |
| `payload` | JSONB | snapshot request. |
| `ip` | INET | sumber. |
| `created_at` | TIMESTAMPTZ | UTC. |

- Audit log **immutable** (append-only, tidak bisa dihapus oleh GM).
- Tersedia endpoint baca terbatas untuk super-admin (`GET /admin/audit?...`).

### 8.6 Validasi Input

- Semua input divalidasi via DTO (class-validator NestJS) di boundary.
- Whitelist field (`whitelist: true`, `forbidNonWhitelisted: true`) — field tak dikenal ditolak.
- Sanitasi string (cegah XSS di `displayName`/guild name saat render klien).
- Batas ukuran body: 64 KB (REST), 16 KB (WS message).
- Rate numeric dalam batas wajar (cegah negative, overflow).

### 8.7 Lainnya

- **Password hashing:** Argon2id (bukan bcrypt) — salt unik per user.
- **JWT:** RS256 (asymmetric) — public key di JWKS; private key di secret manager.
- **Refresh token:** random 256-bit, hash disimpan di DB (never plaintext).
- **DB access:** least-privilege user; no raw SQL dari klien (ORM/query builder parametrized).
- **Secrets:** di Vault/secret manager; tidak di repo.
- **Logging:** jangan log token/password/PII penuh (mask).

---

## 9. Hubungan dengan Dokumen Lain

Dokumen ini (API Contract) adalah antarmuka. Dokumen pendukung:

| Dokumen | Hubungan dengan API Contract |
|---|---|
| **Save Data Schema** | Mendefinisikan struktur payload `data` (hero, inventory, party). API mengembalikan persis shape dari schema ini. Sinkronisasi: klien mengirim intent, server mutasi state sesuai schema lalu kembalikan. |
| **SRS (Software Requirements Spec)** | Sumber kebutuhan fitur (login, gacha, PvP). API Contract adalah realisasi teknis SRS. Setiap endpoint memetakan ke kebutuhan SRS (traceability). |
| **ERD (Entity Relationship Diagram)** | Skema PostgreSQL. Tabel `players`, `heroes`, `inventory`, `ledger`, `marketplace_listings`, `pvp_matches`, `guilds`, `admin_audit_log`. Ledger double-entry untuk semua mutasi currency (anti-lose, audit). |
| **State Machine** | Menjelaskan transisi status: listing (`LISTED → SOLD/CANCELLED`), match (`SEARCHING → MATCHED → RESOLVED/EXPIRED`), order (`PENDING → PAID/FAILED`), account (`ACTIVE → BANNED/DELETED`). API mengembalikan `status` sesuai state. |

### 9.1 Pemetaan Endpoint → SRS (contoh)

| SRS Req | Endpoint |
|---|---|
| FR-AUTH-01 (register) | `POST /auth/register` |
| FR-AUTH-02 (login) | `POST /auth/login` |
| FR-IDLE-01 (idle reward) | `POST /claim-idle` |
| FR-GACHA-01 (pull) | `POST /gacha/pull` |
| FR-PVP-01 (matchmaking) | `POST /pvp/match` |
| FR-MKT-01 (marketplace) | `POST /market/listings`, `/buy`, `/cancel` |
| FR-GUILD-01 (raid) | `POST /guilds/{id}/raids/{rid}/start` |
| FR-COMP-01 (GDPR export) | `GET /account/export` |
| FR-COMP-02 (delete) | `DELETE /account` |

### 9.2 Pemetaan State Machine (Marketplace)

```
LISTED ──buy──> SOLD (item transfer + fee 5%)
  │
  └─cancel──> CANCELLED (item return from escrow)

SOLD, CANCELLED = terminal
```

API: `POST /market/buy` transisi `LISTED→SOLD`; `POST /market/cancel` transisi `LISTED→CANCELLED`. Transisi ilegal → `409 CONFLICT`.

### 9.3 Ledger (ERD ringkas)

Semua perubahan currency lewat `ledger`:

```
ledger_entries:
  entry_id UUID PK
  user_id UUID FK
  txn_type VARCHAR (IDLE, GACHA, MARKET_BUY, MARKET_SELL_FEE, PVP_REWARD, GRANT, PURCHASE, ...)
  currency currency_enum (GOLD, GEM, RUNE)
  delta BIGINT (bisa negatif)
  balance_after BIGINT
  ref_id UUID (runId/orderId/listingId)
  created_at TIMESTAMPTZ
```

- Double-entry: setiap mutasi mencatat debit & kredit (fee platform dicatat ke akun platform).
- Menjamin **tidak ada kehilangan currency** & audit penuh (GDPR/PDPA, fraud).

---

## 10. Lampiran: Model Data Inti & Enum

### 10.1 Currency

| Kode | Nama | Sumber | Dapat dibeli? |
|---|---|---|---|
| `gold` | Gold | PvE, idle, quest | Tidak |
| `gem` | Gem (premium) | Top-up, event | Ya (shop) |
| `rune` | Rune | PvP, event | Tidak |

### 10.2 Rarity

```
enum Rarity { COMMON /*C*/, RARE /*R*/, EPIC /*E*/, LEGENDARY /*L*/, MYTHIC /*M*/ }
```

Gacha base rate: C=0.60, R=0.30, E=0.09, L=0.01. M hanya via event/banner khusus (rate di-override per banner).

### 10.3 Pity

```
pity = {
  sinceRare: int        # reset saat dapat R/E/L; floor R tiap 10
  sinceLegendary: int   # reset saat dapat L; soft 75, hard 90
  softPityActive: bool  # sinceLegendary >= 75
}
```

### 10.4 Party

```
party.slots: array(5) of { slot: 0..4, heroId: UUID|null }
maxSlots = 5
```

### 10.5 Hero

```
hero = {
  heroId, defId, name, rarity, level, ascension,
  equip: { weapon, armor, trinket }, power, acquiredAt
}
```

### 10.6 Battle Outcome

```
enum Outcome { WIN, LOSE, DRAW }
enum BattleStatus { QUEUED, RESOLVED, EXPIRED }
```

### 10.7 PvP Mode

```
enum PvpMode { RANKED, CASUAL }
```

### 10.8 ELO

- Awal: `1000`.
- Kemenangan: `elo += max(8, 32 * (1 - expected))` (expected dari selisih ELO).
- Kekalahan: `elo -= max(8, 32 * expected)`.
- Min ELO: `0` (tidak negatif).

---

## 11. Changelog Dokumen

| Versi | Tanggal | Perubahan |
|---|---|---|
| 1.0.0 | 2026-07-14 | Dokumen awal (kanonikal) — seluruh endpoint REST + WS, auth dual-token, idle server-authoritative, gacha pity, marketplace fee 5%, PvP ELO 1000, GDPR/PDPA. |

---

*— Akhir dokumen API Contract v1.0.0 —*

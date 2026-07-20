# ANALYTICS EVENT TAXONOMY — Fantasy Turn-Based Idle RPG (Web + PWA)

> **Dokumen:** Analytics Event Taxonomy (Event Schema & KPI Mapping)
> **Versi:** 1.0.0  |  **Status:** Canonical / Mengikat
> **Bahasa:** Indonesia (seluruh isi)
> **Konsistensi:** CDP, PRD §8 (KPI), SRS §3.4 (Observability), API_CONTRACT, SAVE_DATA_SCHEMA

---

## 1. Tujuan & Prinsip

| Prinsip | Deskripsi |
|---------|-----------|
| **Event-driven** | Semua KPI PRD §8 & sinyal operasional (SRS §3.4) diukur via event terstruktur. |
| **PII excluded by default** | Email, nama asli, IP mentah, password **tidak pernah** dikirim. `user_id` = UUID v4 server (hashing tidak perlu). |
| **Consent-gated** | Event analytics **hanya kirim** bila `rpg:flags.dataOptInAnalytics === true` (localStorage, lihat SAVE_DATA_SCHEMA §2.4). Opt-out = stop kirim + purge queue lokal. |
| **Idempoten** | Setiap event membawa `event_id` (UUID v4) → server deduplikasi via `event_id` unik per user. Retry offline aman. |
| **Server-time UTC** | `ts` = epoch ms UTC dari server (atau client synced ke NTP). Klien **tidak** boleh pakai `Date.now()` mentah untuk analitik. |
| **Batch-first** | Klien batch event (sekitar 20–50 item) → flush via `POST /analytics/events` atau WS. Offline → antre di IndexedDB `offline_queue` (SAVE_DATA_SCHEMA §3.3). |

---

## 2. Envelope & Format

### 2.1 Single Event Envelope

```json
{
  "event": "category_action",
  "event_id": "eid_01h8x7k9m2n4p6q8r0s2t4v6",
  "user_id": "usr_9a1b2c3d4e5f",
  "session_id": "ses_7k9m2n4p6q8r0s2t",
  "ts": 1752459536123,
  "props": { },
  "v": 1
}
```

| Field | Tipe | Wajib | Keterangan |
|-------|------|-------|------------|
| `event` | string | Ya | Nama event (snake_case, §3) |
| `event_id` | uuid | Ya | Idempotency key; UUID v4 client-generated |
| `user_id` | uuid | Ya | Server UUID (tidak email) |
| `session_id` | uuid | Ya | UUID v4 per sesi aplikasi (reset cold start) |
| `ts` | integer (epoch ms) | Ya | Waktu event UTC (server-synced) |
| `props` | object | Ya | Payload domain-specific (§4) |
| `v` | integer | Ya | Versi schema event (default `1`; bump saat breaking) |

### 2.2 Batch Request (REST)

```json
POST /analytics/events
Content-Type: application/json
Idempotency-Key: <batch-uuid>  // optional, untuk retry aman

{
  "events": [ <event-envelope>, ... ],
  "client_version": "1.3.0",
  "batch_id": "bat_01h8x7k9m2n4"
}
```

**Response 202:** `{ "accepted": 42, "rejected": [], "batch_id": "bat_..." }`

### 2.3 Offline Queue (IndexedDB)

- Event disimpan di store `offline_queue` (SAVE_DATA_SCHEMA §3.3) dengan `type = "analytics"`.
- `payload` = event envelope di atas.
- Flush otomatis saat `navigator.onLine` + heartbeat `/ping` sukses (SAVE_DATA_SCHEMA §8.2).
- `status`: `pending → inflight → acked / failed`. Max retry 5, lalu dead-letter lokal (tanpa PII).

---

## 3. Naming Convention

**Pattern:** `category_action` (snake_case, lowercase)

| Category | Prefix | Contoh |
|----------|--------|--------|
| Lifecycle | `user_` | `user_register`, `user_login`, `user_logout`, `user_delete` |
| Session | `session_` | `session_start`, `session_end` |
| App State | `app_` | `app_foreground`, `app_background` |
| Progression | `prog_` | `prog_stage_clear`, `prog_level_up`, `prog_hero_ascend` |
| Gacha | `gacha_` | `gacha_pull` |
| Idle | `idle_` | `idle_claim`, `idle_accrual_view` |
| Economy | `econ_` | `econ_currency_gain`, `econ_currency_spend`, `econ_purchase_iap`, `econ_ad_reward_watch` |
| Marketplace | `market_` | `market_listing_create`, `market_listing_buy`, `market_listing_cancel` |
| PvP | `pvp_` | `pvp_match_start`, `pvp_match_end` |
| Guild | `guild_` | `guild_join`, `guild_raid_start`, `guild_raid_complete` |
| Social/System | `sys_` | `sys_push_notif_received`, `sys_error_client`, `sys_setting_change` |

---

## 4. Event Catalog

Setiap tabel: **Event | Trigger | Properti (tipe) | Contoh JSON**

### 4.1 Lifecycle

| Event | Trigger | Properti | Contoh |
|-------|---------|----------|--------|
| `user_register` | Registrasi sukses (email verified) | `method: "email" \| "social"` | `{"method":"email"}` |
| `user_login` | Login sukses (token issued) | `method: "email" \| "social", platform: "web" \| "pwa"` | `{"method":"email","platform":"pwa"}` |
| `user_logout` | User klik logout / token revoke | `reason: "user" \| "expired" \| "ban"` | `{"reason":"user"}` |
| `user_delete` | GDPR/PDPA delete 실행 | `retention_days: number` | `{"retention_days":30}` |

### 4.2 Session & App State

| Event | Trigger | Properti | Contoh |
|-------|---------|----------|--------|
| `session_start` | Cold start / reload (pertama kali load shell) | `referrer: string?, build_version: string` | `{"referrer":"utm_source=google","build_version":"1.3.0"}` |
| `session_end` | Tab close / logout / 30 menit idle (session_end = ts_now) | `length_sec: integer` | `{"length_sec":483}` |
| `app_foreground` | Tab visible again (`visibilitychange`) | `from_background_sec: integer` | `{"from_background_sec":120}` |
| `app_background` | Tab hidden / minimize | `session_len_sofar_sec: integer` | `{"session_len_sofar_sec":300}` |

> **Idle handoff:** `app_background` → system mulai idle accrual (server-side, 5 detik tick, cap 24 jam). `app_foreground` → user claim idle via `idle_claim`.

### 4.3 Progression (PvE / Hero)

| Event | Trigger | Properti | Contoh |
|-------|---------|----------|--------|
| `prog_stage_clear` | Stage cleared (server validasi) | `stage_id: string, power: integer, stamina_cost: integer, stars: 1\|2\|3, wave_cleared: integer, party:<<u>, duration_sec: integer, elements_used: string[]` | `{"stage_id":"stg_05_03","power":18500,"stamina_cost":10,"stars":3,"wave_cleared":5,"duration_sec":42,"elements_used":["Fire","Earth"]}` |
| `prog_level_up` | Hero naik level | `hero_id: string, hero_def_id: string, new_level: integer, old_level: integer, xp_gained: integer` | `{"hero_id":"hero_abc","hero_def_id":"def_hero_kael","new_level":25,"old_level":24,"xp_gained":1200}` |
| `prog_hero_ascend` | Ascension sukses | `hero_id: string, ascension_tier: 1\|2\|3\|4\|5, materials_burned: {item_id:string,qty:integer}[]` | `{"hero_id":"hero_abc","ascension_tier":3,"materials_burned":[{"item_id":"mat_fire_essence","qty":5}]}` |
| `prog_hero_summon_gacha` | Pull gacha (server-side roll) | `banner_id: string, pull_type: "single" \| "multi10", rarity: "C"\|"R"\|"E"\|"L", pity_before: integer, is_pity: boolean, hero_def_id: string, instance_id: string` | `{"banner_id":"bn_std","pull_type":"multi10","rarity":"L","pity_before":89,"is_pity":true,"hero_def_id":"def_hero_lux","instance_id":"inst_xyz"}` |

> **CDP Pity:** soft pity mulai pull 75, hard pity pull 90, counter per banner (PRD §6.4, §17).

### 4.4 Idle

| Event | Trigger | Properti | Contoh |
|-------|---------|----------|--------|
| `idle_claim` | User klaim reward idle (POST /claim-idle) | `hours_claimed: number (≤24), gold_gained: integer, exp_gained: integer, items_gained: {item_id:string,qty:integer}[], capped: boolean` | `{"hours_claimed":8.5,"gold_gained":42500,"exp_gained":1200,"items_gained":[{"item_id":"mat_iron","qty":3}],"capped":false}` |
| `idle_accrual_view` | User buka panel idle (UI view) | `accrued_hours: number, projected_gold_hr: integer` | `{"accrued_hours":5.2,"projected_gold_hr":5000}` |

> **Cap 24 jam:** `hours_claimed` dikirim **clamped** 24 (server sudah clamp). `capped=true` jika elapsed > 86400 detik.

### 4.5 Economy (Currency & Monetization)

| Event | Trigger | Properti | Contoh |
|-------|---------|----------|--------|
| `econ_currency_gain` | Currency masuk (source server-side) | `currency: "gold"\|"gems"\|"eventToken", amount: integer, source: "stage_clear"\|"idle"\|"market_sell"\|"quest"\|"admin_grant"\|"gacha_duplicate"\|"pvp_reward"\|"guild_raid"\|"event_reward", ref_id?: string` | `{"currency":"gold","amount":5000,"source":"idle","ref_id":"idle_claim_123"}` |
| `econ_currency_spend` | Currency keluar (sink) | `currency: "gold"\|"gems"\|"eventToken", amount: integer, sink: "gear_upgrade"\|"market_buy"\|"market_fee"\|"hero_ascend"\|"gacha_pull"\|"stamina_refill"\|"pvp_entry"\|"consumable_use", ref_id?: string` | `{"currency":"gold","amount":25000,"sink":"gear_upgrade","ref_id":"gear_upg_456"}` |
| `econ_purchase_iap` | IAP verified (webhook /api/v1/shop/webhook) | `sku: string, product_id: string, currency_granted: "gems", amount_granted: integer, price_usd: number, region: string, transaction_id: string` | `{"sku":"sku_gem_1000","product_id":"com.rpg.gem1000","currency_granted":"gems","amount_granted":1000,"price_usd":9.99,"region":"US","transaction_id":"txn_apple_123"}` |
| `econ_ad_reward_watch` | Rewarded ad selesai (Post-MVP) | `ad_unit: string, reward_currency: "gold"\|"gems"\|"eventToken", reward_amount: integer` | `{"ad_unit":"rewarded_gem_50","reward_currency":"gems","reward_amount":50}` |

> **Currencies:** Gold (soft), Gems (premium), EventToken (time-limited, burn otomatis PRD §21).

### 4.6 Marketplace

| Event | Trigger | Properti | Contoh |
|-------|---------|----------|--------|
| `market_listing_create` | Listing dibuat (POST /market/listings) | `listing_id: string, item_instance_id: string, item_def_id: string, rarity: "C"\|"R"\|"E"\|"L", price_gold: integer, fee_gold: integer (5%), ttl_hours: integer` | `{"listing_id":"lst_789","item_instance_id":"inst_abc","item_def_id":"def_sword_fire","rarity":"E","price_gold":50000,"fee_gold":2500,"ttl_hours":48}` |
| `market_listing_buy` | Pembeli beli (POST /market/buy/{id}) | `listing_id: string, seller_id: string (hashed), buyer_id: string (self), price_gold: integer, fee_gold: integer, item_instance_id: string` | `{"listing_id":"lst_789","seller_id":"usr_xxx","buyer_id":"usr_9a1b","price_gold":50000,"fee_gold":2500,"item_instance_id":"inst_abc"}` |
| `market_listing_cancel` | Penjual cancel | `listing_id: string, seller_id: string, fee_refunded: boolean (false)` | `{"listing_id":"lst_789","seller_id":"usr_xxx","fee_refunded":false}` |

> **Fee 5%:** dipotong saat **jual berhasil** (buyer pays full, seller receives `price - fee`). Cancel → fee tidak dikembalikan (PRD §14, SRS §9.3).

### 4.7 PvP

| Event | Trigger | Properti | Contoh |
|-------|---------|----------|--------|
| `pvp_match_start` | Match ditemukan (WS `pvp_match_found`) | `match_id: string, opponent_id: string (hashed), my_elo: integer, opp_elo: integer, tier: "Bronze"\|"Silver"\|"Gold"\|"Platinum"\|"Diamond"` | `{"match_id":"mat_123","opponent_id":"usr_yyy","my_elo":1150,"opp_elo":1120,"tier":"Silver"}` |
| `pvp_match_end` | Match selesai (submit result) | `match_id: string, result: "win"\|"loss"\|"draw", elo_before: integer, elo_after: integer, elo_delta: integer, battle_duration_sec: integer` | `{"match_id":"mat_123","result":"win","elo_before":1150,"elo_after":1185,"elo_delta":35,"battle_duration_sec":180}` |

> **ELO:** start 1000, K-factor per tier (PRD §18.1). Matchmaking ±200 ELO (D-04).

### 4.8 Guild

| Event | Trigger | Properti | Contoh |
|-------|---------|----------|--------|
| `guild_join` | User join guild (POST /guilds/{id}/join) | `guild_id: string, role: "member"\|"officer"\|"leader"` | `{"guild_id":"gld_abc","role":"member"}` |
| `guild_raid_start` | Raid mingguan mulai (GM/officer) | `guild_id: string, raid_id: string, tier: 1\|2\|3, participant_count: integer` | `{"guild_id":"gld_abc","raid_id":"raid_01","tier":2,"participant_count":18}` |
| `guild_raid_complete` | Raid selesai / claim reward | `guild_id: string, raid_id: string, tier: integer, total_damage: integer, user_damage: integer, reward_gold: integer, reward_items: {item_id:string,qty:integer}[]` | `{"guild_id":"gld_abc","raid_id":"raid_01","tier":2,"total_damage":5000000,"user_damage":320000,"reward_gold":15000,"reward_items":[{"item_id":"mat_raid_token","qty":5}]}` |

### 4.9 Social / System

| Event | Trigger | Properti | Contoh |
|-------|---------|----------|--------|
| `sys_push_notif_received` | FCM/Push diterima (foreground/background) | `campaign_id: string, type: "raid"\|"pvp"\|"idle_ready"\|"event", opened: boolean` | `{"campaign_id":"camp_raid_01","type":"raid","opened":true}` |
| `sys_error_client` | Error klien (non-PII) | `code: string, message: string, context: string?, severity: "warn"\|"error"\|"fatal"` | `{"code":"WS_RECONNECT_FAILED","message":"max retries exceeded","context":"flush_queue","severity":"error"}` |
| `sys_setting_change` | User ubah setting (server-saved) | `key: "language"\|"notifications"\|"auto_battle"\|"graphics_quality", old_value: any, new_value: any` | `{"key":"language","old_value":"en","new_value":"id"}` |

---

## 5. KPI → Event Mapping (PRD §8)

| KPI (PRD §8.1) | Definisi | Event(s) Digunakan | Rumus (Konseptual) |
|----------------|----------|-------------------|---------------------|
| **D1 Retention** | % user kembali hari-1 | `user_login` (cohort join date vs login D+1) | `count(distinct user_id where login_date = install_date + 1) / count(distinct user_id where install_date = D)` |
| **D7 Retention** | % user kembali hari-7 | `user_login` | `count(distinct user_id where login_date = install_date + 7) / cohort_size` |
| **Session Length** | Rata-rata menit/sesi | `session_start`, `session_end` | `avg(session_end.ts - session_start.ts) / 60000` |
| **Idle Claim Freq** | Claim/hari per user | `idle_claim` | `count(idle_claim) / DAU` (per hari) |
| **Conversion Rate** | % DAU beli IAP | `econ_purchase_iap`, `user_login` (DAU) | `count(distinct user_id in econ_purchase_iap) / DAU` |
| **ARPDAU** | Revenue / DAU | `econ_purchase_iap.price_usd`, `user_login` (DAU) | `sum(econ_purchase_iap.price_usd) / DAU` |
| **Churn (weekly)** | % berhenti main | `user_login` (cohort drop-off) | `1 - (active_week_N / active_week_N-1)` |
| **Trading Volume** | Gold traded/hari | `market_listing_buy.price_gold` | `sum(market_listing_buy.price_gold) / day` |
| **PvP Participation** | % DAU PvP/minggu | `pvp_match_start`, `user_login` | `count(distinct user_id in pvp_match_start week) / DAU_week` |
| **Guild Join Rate** | % DAU di guild | `guild_join`, `user_login` | `count(distinct user_id with guild_id) / DAU` |

> **Security KPI (PRD §8.2)** pakai log server (bukan event klien): `cheat_block`, `duplicate_item_audit`, `gacha_dispute`, `refund_event`.

---

## 6. Consent & Privacy

| Aturan | Implementasi |
|--------|--------------|
| **Opt-in analytics** | `rpg:flags.dataOptInAnalytics === true` (localStorage, SAVE_DATA_SCHEMA §2.4). Default `false`. |
| **Opt-out** | Set flag `false` → client **hentik pengiriman**, `flushQueue` analytics dihapus, event baru tidak di-enqueue. |
| **user_delete (GDPR/PDPA)** | Server endpoint `POST /account/delete` (API_CONTRACT §3.12) → anonymize `user_id` di event store (hash/salt), hapus PII, retensi log minimal wajib hukum (12–24 bln, PRD §16.3). |
| **PII tidak dikirim** | Email, nama, IP mentah, password **tidak pernah** di `props`. `user_id` = UUID v4 server (publik, bukan rahasia). |
| **Geo/IP** | Jika butuh region untuk pricing → resolve di server via IP request header, **jangan** kirim dari client. |

---

## 7. Transport

| Jalur | Endpoint / Channel | Format | Catatan |
|-------|-------------------|--------|---------|
| **REST Batch** | `POST /analytics/events` (API_CONTRACT § baru — tambahkan) | JSON batch (§2.2) | Idempotency-Key opsional; rate limit 60 req/user/menit. |
| **WebSocket** | `wss://api.example.com/ws` (API_CONTRACT §4) | Envelope WS `type: "analytics_batch", payload: {events: [...]}` | Gunakan koneksi WS existing (auth token). Cocok real-time flush. |
| **Offline Queue** | IndexedDB `offline_queue` type=`"analytics"` (SAVE_DATA_SCHEMA §3.3) | Event envelope penuh | Flush saat `ONLINE` + heartbeat OK (§8.2 SAVE_DATA_SCHEMA). Max 500 record; purge acked >7 hari. |

**Header wajib REST:** `Content-Type: application/json`, `Authorization: Bearer <access_jwt>` (cookie httpOnly, tapi header untuk WS handshake).

---

## 8. Dashboard & Alarm (Ops)

| Metrik Wajib (Grafana) | Sumber Event | Alert Threshold (Contoh MVP) |
|------------------------|--------------|------------------------------|
| **DAU** | `user_login` (distinct/day) | Drop >30% vs 7-hari lalu |
| **GDP Gold** (Total Gold ekonomi) | `econ_currency_gain(source="idle|stage_clear|market_sell") - econ_currency_spend` snapshot harian | Naik >50%/hari (inflasi anomali) |
| **Gacha Rate Drift** | `prog_hero_summon_gacha.rarity` | Actual L% deviasi >2× dari CDP rate (soft pity) |
| **Duplicate Item** | `prog_hero_summon_gacha.instance_id` unik | >0 duplikat → alert kritis (PRD §9.1) |
| **Market GMV** | `market_listing_buy.price_gold` sum/day | Drop >40% atau spike >3× |
| **Error Rate Client** | `sys_error_client.severity="error"` | >1% sessions |
| **PvP Match Fail** | `pvp_match_start` tanpa `pvp_match_end` dalam 10 menit | >5% |

> Referensi: SRS §3.4 (NFR-OBS), INFRA.md dashboard admin (PRD §15.3), TDD observability checklist.

---

## 9. Versioning & Evolusi

| Aturan | Detail |
|--------|--------|
| **Field `v`** | Integer di envelope (§2.1). Default `1`. Bump **hanya** saat breaking change (hapus/rename field, tipe berubah). |
| **Backward compatible** | Tambah field baru **opsional** → `v` tetap. Consumer harus toleran field tak dikenal (ignore). |
| **Schema registry** | Simpan definisi JSON Schema per `event` + `v` di repo (CI validate). |
| **Migrasi klien** | Klien lama (v=1) tetap kirim v=1; server normalisasi ke v=2 internal. Klien baru kirim v=2. |
| **Deprekasi** | Event lama boleh dihentikan setelah 2 rilis mayor. Hapus dari catalog doc tapi biarkan server accept (silent drop). |

---

## 10. Lampiran: Contoh Batch Lengkap (REST)

```json
POST /analytics/events
{
  "events": [
    {
      "event": "session_start",
      "event_id": "eid_01h8x7k9m2n4p6q8r0s2t4v6",
      "user_id": "usr_9a1b2c3d4e5f",
      "session_id": "ses_7k9m2n4p6q8r0s2t",
      "ts": 1752459536123,
      "props": { "referrer": "utm_source=google", "build_version": "1.3.0" },
      "v": 1
    },
    {
      "event": "prog_stage_clear",
      "event_id": "eid_01h8x7k9m2n4p6q8r0s2t4v7",
      "user_id": "usr_9a1b2c3d4e5f",
      "session_id": "ses_7k9m2n4p6q8r0s2t",
      "ts": 175245957840456,
      "props": {
        "stage_id": "stg_05_03",
        "power": 18500,
        "stamina_cost": 10,
        "stars": 3,
        "wave_cleared": 5,
        "duration_sec": 42,
        "elements_used": ["Fire", "Earth"]
      },
      "v": 1
    },
    {
      "event": "idle_claim",
      "event_id": "eid_01h8x7k9m2n4p6q8r0s2t4v8",
      "user_id": "usr_9a1b2c3d4e5f",
      "session_id": "ses_7k9m2n4p6q8r0s2t",
      "ts": 1752459600000,
      "props": {
        "hours_claimed": 8.5,
        "gold_gained": 42500,
        "exp_gained": 1200,
        "items_gained": [{ "item_id": "mat_iron", "qty": 3 }],
        "capped": false
      },
      "v": 1
    }
  ],
  "client_version": "1.3.0",
  "batch_id": "bat_01h8x7k9m2n4"
}
```

---

> **Akhir Dokumen** — Semua event, nama, tipe, dan rumus KPI **mengikat** dan harus konsisten dengan CDP, PRD §8, SRS §3.4, API_CONTRACT, SAVE_DATA_SCHEMA. Perubahan → update versi (`v`) + changelog.
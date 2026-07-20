# THREAT_MODEL.md — Fantasy Turn-Based Idle RPG (Web + PWA)

> **Dokumen:** Threat Model & Abuse Case Catalog (Keamanan Ekonomi & Anti-Cheat)
> **Versi:** 1.0 (Kanonikal, mengikat)
> **Tanggal:** 2026-07-14
> **Bahasa:** Bahasa Indonesia
> **Owner:** Security Engineer / Threat Modeler
> **Status:** Draft untuk review (lintas dokumen: CDP, PRD, SRS, ERD, API_CONTRACT, SAVE_DATA_SCHEMA, INFRA, DECISION_REGISTER)

> **Konsistensi CDP:** Seluruh angka/rate/nama sistem mengikat parameter desain kanonik (rate gacha 60/30/9/1, pity 10/75/90 per banner, idle cap 24 jam, tick 5 detik, marketplace fee 5%, currency Gold/Gems/EventToken, server-authoritative). Bila ada konflik dengan dokumen lain, **CDP menang** — dan dokumen ini selaras dengan SRS §3.2, ERD §G, API_CONTRACT §8, dan SAVE_DATA_SCHEMA §9.

---

## 1. Ruang Lingkup & Asumsi

### 1.1 Ruang Lingkup (In Scope)

Threat model ini melengkapi SRS (NFR Security/OWASP, §3.2) dan fokus pada dua domain berisiko tinggi:

1. **Ekonomi virtual** — currency (Gold/Gems/EventToken), gacha, marketplace, idle accrual.
2. **Anti-cheat & integritas server-authoritative** — battle result, RNG, ledger, validasi intent.

Cakupan mencakup: client (Phaser 3 PWA), transport (REST + WebSocket), API (NestJS), datastore (PostgreSQL `currency_ledger` + Redis), integrasi eksternal (Payment Provider, Email, Analytics), dan Admin/GM panel.

### 1.2 Di Luar Cakupan (Out of Scope)

- Kriptografi sisi klien sebagai rahasia (dianggap tidak aman — SAVE_DATA_SCHEMA R9).
- Audit kepatuhan formal GDPR/PDPA (ditangani NFR-PRIV; di sini hanya aspek abuse PII).
- Infrastruktur fisik / cloud provider (rujuk INFRA.md).
- Ancaman supply-chain pada dependency pihak ke-3 (SRS SEC-06 rujuk, di sini hanya disinggung).

### 1.3 Batasan Desain (Asumsi Kritis)

| Kode | Asumsi | Sumber |
|---|---|---|
| SC-01 | **Client tidak dipercaya.** Klien hanya kirim *intent* (skillId, targetId), tidak pernah otoritas currency/combat/gacha. | CDP A1, SRS CON-02, SAVE_DATA_SCHEMA R1 |
| SC-02 | Waktu server (NTP-sync) adalah satu-satunya sumber waktu; klien tidak boleh kirim `now`. | API_CONTRACT §0, SRS ASP-05 |
| SC-03 | Autentikasi email/password; access JWT (15 mnt) + refresh httpOnly cookie (30 hari, rotasi). | CDP, SRS FR-AUTH-03, API_CONTRACT §2 |
| SC-04 | `currency_ledger` append-only; `wallets.balance` hanya cache via prosedur CAS. | ERD §G.1, B.6, B.7 |
| SC-05 | Marketplace escrow atomik, fee 5%, 1 listing = 1 transaksi final. | SRS §6.3, ERD §G.3, API_CONTRACT §3.6 |
| SC-06 | Gacha RNG seeded server, pity per-banner ter-lock, `gacha_pulls` immutable. | SRS §6.4, ERD §G.4, FR-GAC-06/07 |
| SC-07 | Idle accrual dihitung server saat claim: `clamp(now - last_seen, 0, 24h)`. | SRS §6.1, D-08 (skew ±2 mnt), SAVE_DATA_SCHEMA §9.1 |
| SC-08 | Multi-akun di 1 device hanya dideteksi ringan (device fingerprint); belum anti-abuse canggih. | SRS ASP-01, SEC-16 |

### 1.4 Aset Terlindungi (Protected Assets)

| Aset | Dampak jika kompromi | Prioritas |
|---|---|---|
| **Currency** (Gold/Gems/EventToken) | Inflasi, ekonomi hancur, monetisasi bocor | KRITIS |
| **Item unik / hero instance** | Duplikasi, rugi pemain, kehilangan nilai pasar | TINGGI |
| **Integritas gacha** (rate, pity, seed) | Kehilangan kepercayaan, dispute hukum | KRITIS |
| **Account & identitas** (JWT, refresh, PII) | Takeover, pencurian aset, pelanggaran privasi | KRITIS |
| **Battle result / state** | PvP/adil rusak, leaderboard manipulasi | TINGGI |
| **Ledger & audit log** | Tamper tutup jejak fraud, tidak auditabel | TINGGI |
| **Admin panel / GM** | Grant ilegal, ban sewenang-wenang, penyalahgunaan | TINGGI |

---

## 2. Metodologi

### 2.1 Pendekatan

Menggunakan **STRIDE** (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege) untuk mengkategorikan ancaman, dilengkapi **attack tree** per domain ekonomi untuk memetakan jalur konkret penyerang.

### 2.2 STRIDE — Ringkasan Pemetaan

| Kategori STRIDE | Contoh ancaman dalam game ini |
|---|---|
| **S — Spoofing** | Session hijack WS, replay token, impersonasi akun, fake webhook payment. |
| **T — Tampering** | Ledger tamper, manipulasi gacha seed, edit indexledDB cache, market listing palsu. |
| **R — Repudiation** | Aksi GM tanpa audit, double-claim tanpa idempotensi, transaksi tanpa nonce. |
| **I — Information Disclosure** | Leak PII via error, XSS baca token, bocor rate gacha internal, enumeration akun. |
| **D — Denial of Service** | Bot idle-farm, spam endpoint, race condition lock, payload abuse. |
| **E — Elevation of Privilege** | Brute force login, credential stuffing, privilege escalation admin, self-trade. |

### 2.3 Matriks STRIDE × Aset Terlindungi

| Aset \ STRIDE | S | T | R | I | D | E |
|---|---|---|---|---|---|---|
| Currency | | T5 | T1,T2 | | T12 | |
| Item unik | | T2 | T2 | | | |
| Integritas gacha | | T3 | T3 | I | | |
| Account/Identitas | T7,T8 | | T7 | T9 | | T11 |
| Battle result | | T6 | T6 | | | |
| Ledger & audit | | T5 | T1,T3 | | | T11 |
| Admin/GM | | | T11 | | | T11 |

> Matriks ini mengarahkan pengujian: setiap sel terisi punya abuse case terpadu di §3. Sel kosong = risiko rendah / tidak relevan untuk aset tersebut.

### 2.4 Prinsip Mitigasi Universal (berlaku semua threat)

1. **Never trust client** — seluruh kebenaran di server (SC-01, SRS CON-02).
2. **Atomic & append-only** — mutasi currency 1 tx; ledger tidak bisa diubah (ERD §G).
3. **Idempotency + anti-replay** — `Idempotency-Key` + `nonce`/`timestamp` (API_CONTRACT §8.3/§8.4).
4. **Defense in depth** — boundary ganda (edge rate-limit + API validate + DB constraint).
5. **Tamper-evidence** — HMAC signature di ledger & pull untuk verifikasi pasif/aktif.
6. **Auditability** — log immutable (`admin_audit_log`, `gacha_pulls`, `currency_ledger`).

### 2.3 Attack Tree — Domain Ekonomi Utama

#### 2.3.1 Marketplace Double-Spend (Goal: dapat item gratis / duplikat uang)
```
[ROOT] Dapat item tanpa bayar / duplikat currency
 ├─(T) Race: 2 pembeli kirim buy bersamaan ke 1 listing
 │     └─ mitigasi: SELECT...FOR UPDATE + UQ(listing_id) dalam 1 tx SERIALIZABLE (ERD §G.3)
 ├─(R) Replay buy dengan Idempotency-Key sama setelah sukses
 │     └─ mitigasi: key→response cache TTL 24j; key beda body → 422 (API_CONTRACT §8.3)
 ├─(T) Manipulasi escrow (item kembali lalu dijual 2x)
 │     └─ mitigasi: ownership lock + escrow_holds status HELD (SRS §6.3)
 └─(T) Self-trade antar akun sendiri untuk cuci currency
       └─ mitigasi: deteksi pola + device fingerprint (SRS ASP-01, SEC-16)
```

#### 2.3.2 Item Duplication (Goal: clone hero/item)
```
[ROOT] Duplikasi item/hero instance
 ├─(R) Replay ledger entry (nonce dipakai ulang)
 │     └─ mitigasi: UNIQUE(user_id, nonce) di currency_ledger (ERD B.7)
 ├─(T) Rollback transaksi setelah komit (optimistic lock bypass)
 │     └─ mitigasi: CAS version++ + append-only, no direct UPDATE (ERD §G.1)
 └─(D) Daily scan duplikat terlewat
       └─ mitigasi: job scan duplikat instance (PRD §35 anti-fraud runbook, ERD)
```

#### 2.3.3 Gacha RNG Manipulation (Goal: force Legendary)
```
[ROOT] Manipulasi hasil gacha
 ├─(I) Leak seed / prediksi pull
 │     └─ mitigasi: seed di server, tidak dikirim klien (SRS §6.4, FR-GAC-06)
 ├─(T) Edit pity counter (client) sebelum pull
 │     └─ mitigasi: pity_state ter-lock dalam tx pull (ERD §G.4)
 ├─(R) Pull tanpa potong currency (ledger_ref kosong)
 │     └─ mitigasi: gacha_pulls.ledger_ref_id mengikat potongan GEMS (ERD §G.4)
 └─(T) Manipulasi rate via replay batch_id
       └─ mitigasi: batch_id + pull_index unique per batch (ERD §G.4)
```

#### 2.3.4 Idle Accrual Spoof (Goal: dapat reward 24j berulang)
```
[ROOT] Spoof akrual idle
 ├─(T) Kirim last_seen palsu / ubah clock klien
 │     └─ mitigasi: server pakai now sendiri; clamp 24h (SRS §6.1, D-08)
 ├─(R) Claim berkali-kali dalam 1 menit (elapsed kecil)
 │     └─ mitigasi: threshold <60s → reward 0; update last_seen atomik (SRS §6.1)
 └─(T) Replay claimIdle queue (IndexedDB) saat offline
       └─ mitigasi: ticket valid + idempotencyKey (SAVE_DATA_SCHEMA §9.1, §9.2)
```

---

## 3. Abuse Cases / Threat Catalog

Format tiap entri: **Vector | Dampak | Kontrol Ada | Risiko Residu | Mitigasi**.

### 3.0 Indeks Abuse Case (ringkasan)

| ID | Nama | STRIDE | Severity (lihat §5) | Kontrol Utama |
|---|---|---|---|---|
| T1 | Marketplace double-spend / race | T,R,D | High | Escrow atomik + UQ + idempotency |
| T2 | Item duplication | T,R | High | Nonce UQ + daily scan |
| T3 | Gacha RNG manipulation | T,R,I | High | Seeded RNG + pity lock + monitor |
| T4 | Idle accrual spoof | T,R | Med | Server clamp + nonce/ts |
| T5 | Currency ledger tamper | T | High | Append-only + HMAC + CAS + RLS |
| T6 | Battle result forge | T,R | Med | Validate-result + sanity (D-16) |
| T7 | Auth brute force / stuffing | S,E | High | Lockout + rate limit + captcha |
| T8 | WS replay / session hijack | S,R | High | Anti-replay + single session |
| T9 | XSS / CSRF | I,E | Med | Sanitasi + CSP + SameSite |
| T10 | SQL/injection | T,I | High | Parameterized + DTO whitelist |
| T11 | Privilege escalation admin | E,R | High | RBAC + MFA + audit immutable |
| T12 | Bot / farm / auto-clicker | D | Med | Fingerprint + captcha + GDP alert |
| T13 | Payment fraud / chargeback | S,T | High | Webhook HMAC + idempotensi |

### 3.1 Marketplace Double-Spend / Race

| Item | Detail |
|---|---|
| **Vector** | Dua pembeli mengirim `POST /market/buy/{listingId}` bersamaan ke listing yang sama; atau replay buy dengan Idempotency-Key usai sukses. |
| **Dampak** | Item transfer ke 2 pembeli (duplikasi), penjual rugi, atau currency pembeli berkurang 2x tanpa item. Ekonomi marketplace runtuh. |
| **Kontrol Ada** | Escrow atomik 1 tx `SERIALIZABLE` (SRS §6.3, ERD §G.3); `marketplace_transactions.listing_id` UNIQUE cegah double-sell; `Idempotency-Key` → cache response TTL 24j, mismatch body → 422 (API_CONTRACT §8.3); item lock saat listing (FR-HERO-09). |
| **Risiko Residu** | Deadlock/timeout tx pada beban tinggi; replay lintas batas TTL 24j (jarang); self-trade antar akun sendiri. |
| **Mitigasi** | Row lock `SELECT...FOR UPDATE` + retry backoff; batas concurrency per listing; deteksi self-trade via device fingerprint (SRS ASP-01, SEC-16); alert volume buy anomali (§6). |

### 3.2 Item Duplication

| Item | Detail |
|---|---|
| **Vector** | Replay ledger entry (nonce dipakai ulang); rollback transaksi setelah commit; clone instance via bug equip/transfer. |
| **Dampak** | Infinite currency/item; inflasi; rugi pemain lain; marketplace tidak adil. |
| **Kontrol Ada** | `currency_ledger` append-only + `UNIQUE(user_id, nonce)` (ERD B.7, §G.2); `wallets.version` CAS cegah lost-update (ERD B.6); instance unik per `hero_instances`/`inventory.id` (ERD B.11/B.14); atomic lock. |
| **Risiko Residu** | Duplikat instance dari race non-ledger (mis. equipment slot partial UQ); scan butuh job berkala. |
| **Mitigasi** | **Daily scan duplikat** instance/quantity (PRD §35 runbook anti-fraud, ERD); `quantity` invariant `CHECK >=0`; alert bila balance ≠ SUM(delta) per user (reconcile ledger vs wallet). |

### 3.3 Gacha RNG Manipulation

| Item | Detail |
|---|---|
| **Vector** | Leak/prediksi seed; edit `pity_state` di client; pull tanpa potong currency; replay `batch_id`. |
| **Dampak** | Pemain dapat Legendary gratis; statistik rate melenceng; dispute & kehilangan trust; potensi fraud monetisasi. |
| **Kontrol Ada** | Seeded RNG server, seed tidak dikirim klien (FR-GAC-06, SRS §6.4); `gacha_pulls` immutable per pull (ERD §G.4); `pity_state` ter-lock dalam tx pull; `ledger_ref_id` ikat potongan GEMS; log auditable (FR-GAC-07). |
| **Risiko Residu** | Seed persistence leak dari log/backup; bias RNG tak terdeteksi; regulasi proof-of-fairness (ASP-09). |
| **Mitigasi** | Audit HMAC signature tiap pull; **monitor deviasi rate** vs 60/30/9/1 (§6 alert); upgrade ke verifiable RNG bila regulasi ketat (ASP-09); jangan log seed lengkap di plaintext. |

### 3.4 Idle Accrual Spoof

| Item | Detail |
|---|---|
| **Vector** | Kirim `last_seen` palsu / ubah clock sistem klien; claim berulang dalam selang pendek; replay `claimIdle` dari IndexedDB queue offline. |
| **Dampak** | Farm Gold tak terbatas; inflasi ekonomi soft currency. |
| **Kontrol Ada** | Server hitung `elapsed = clamp(now - last_seen, 0, 24h)` (SRS §6.1); toleransi skew D-08 ±2 menit; klien tidak kirim `now` (API_CONTRACT §0); claim < 60s → reward 0; `last_seen` update atomik pasca-claim. |
| **Risiko Residu** | Drift NTP server besar (ASP-05); replay queue offline sebelum sink; pemain tutup-tab tiap 24j lalu claim. |
| **Mitigasi** | `X-Request-Timestamp` ±30 dtk + nonce WS (API_CONTRACT §8.4); `idempotencyKey` tiap claimIdle (SAVE_DATA_SCHEMA §9.2); alert akun claim 24j tiap <5 menit (SAVE_DATA_SCHEMA §9.5). |

### 3.5 Currency Ledger Tamper

| Item | Detail |
|---|---|
| **Vector** | UPDATE langsung `wallets.balance` (bypass ledger); insert delta negatif agar balance negatif; modifikasi baris ledger (rollback). |
| **Dampak** | Saldo negatif/ghost; ekonomi tidak konsisten; jejak audit rusak. |
| **Kontrol Ada** | `currency_ledger` append-only (no UPDATE/DELETE) (ERD §G.2); `balance_after CHECK >= 0`; `wallets.balance` hanya via prosedur CAS `WHERE version=:expected` (ERD §G.1); `signature` HMAC tamper-evidence (ERD B.7, §G.2). |
| **Risiko Residu** | Compromise DB credential → bypass prosedur; HMAC secret leak. |
| **Mitigasi** | DB user least-privilege, revoke akses langsung via RLS/stored procedure (SRS SEC-13); secret di Vault/secret manager (SEC-10); periodic verify HMAC seluruh baris (job); alert bila `SUM(delta) != balance`. |

### 3.6 Battle Result Forge

| Item | Detail |
|---|---|
| **Vector** | Klien kirim hasil (`HP musuh = 0`, `outcome=WIN`, reward besar) sebagai ganti intent. |
| **Dampak** | Menang PvP palsu, naik ELO ilegal, farm reward stage tanpa main; leaderboard manipulasi. |
| **Kontrol Ada** | Server-authoritative: client hanya kirim `skillId`+`targetId` (SRS §6.2, FR-CMB-01); validate-result + sanity check (D-16); reject intent tidak valid (PRD US-SEC-01); replay `(seed, inputs[])` untuk forensik (SRS §6.2-5). |
| **Risiko Residu** | Sanity check tidak cukup untuk full-replay (D-16 pilih validate-result untuk MVP); exploit logic gap di simulasi server. |
| **Mitigasi** | Upgrade ke full-replay verification (D-16 post-MVP); anomali win-rate / stat absurd → review manual (SRS §3.2.3); reject reward client-side. |

### 3.7 Auth — Brute Force / Credential Stuffing

| Item | Detail |
|---|---|
| **Vector** | Spray password banyak akun; credential stuffing dari breach eksternal; enumerasi email via respon login. |
| **Dampak** | Account takeover; pencurian aset; penyalahgunaan monetisasi. |
| **Kontrol Ada** | Lock 5 gagal / 15 menit per email+IP (FR-AUTH-04, SEC-04); rate limit login (SEC-03); argon2id (FR-AUTH-02, SEC-02); respon ambigu (FR-AUTH-11, 409 tanpa bocorkan keberadaan); refresh rotation + reuse detect → logout all (FR-AUTH-08/09). |
| **Risiko Residu** | Stuffing pakai proxy pool lolos per-IP limit; akun lemah tetap rentan. |
| **Mitigasi** | Captcha pada gagal ke-N; rate limit per-akun + per-IP (token bucket Redis); monitor password spray pattern; anjurkan 2FA (post-MVP PRD F10). |

### 3.8 WebSocket Replay / Session Hijack

| Item | Detail |
|---|---|
| **Vector** | Capture pesan WS (`pvp_action`, `claimIdle`) lalu replay; curi token via MITM; reuse sesi di device lain. |
| **Dampak** | Aksi berulang (damage/claim ganda); takeover sesi; pencurian aset. |
| **Kontrol Ada** | `wss://` wajib + HSTS (API_CONTRACT §8.1); token di WS upgrade (bearer/cookie); `X-Request-Timestamp` ±30 dtk + `X-Nonce` unik per request, reuse → 401 REPLAY_DETECTED (§8.4); single session; expiry access 15 mnt. |
| **Risiko Residu** | Token di memori JS tetap bisa dibaca ekstensi berbahaya; nonce store Redis TTL 60 dtk terbatas. |
| **Mitigasi** | Refresh httpOnly cookie (R5); CSP ketat (SEC-09); deteksi sesi ganda (device/IP anomali) → force re-auth; batas 1 WS connection aktif per session. |

### 3.9 XSS / CSRF

| Item | Detail |
|---|---|
| **Vector** | Chat guild / marketplace listing berisi payload; injection ke `displayName`/guild name; CSRF ke endpoint state-changing. |
| **Dampak** | XSS curi token (bila token di JS), deface, redirect; CSRF lakukan aksi atas nama korban. |
| **Kontrol Ada** | Sanitasi string di boundary (SEC-08, API_CONTRACT §8.6); CSP nonces (SEC-09); Phaser text aman (no `innerHTML`); refresh cookie `SameSite=Strict` + `HttpOnly` (API_CONTRACT §2.1); output encoding. |
| **Risiko Residu** | XSS stored di konten user (chat belum ada di MVP F17); CSP misconfig; token access di memori JS masih reachable. |
| **Mitigasi** | Whitelist field + reject non-whitelisted (§8.6); escape semua render user-content; `SameSite=Strict` cegah CSRF; WAF filter payload; DOMPurify bila ada chat (post-MVP). |

### 3.10 SQL Injection / Injection

| Item | Detail |
|---|---|
| **Vector** | String concat ke query; parameter body tidak divalidasi; `ref_id`/search injection. |
| **Dampak** | Bypass authz, leak PII, modifikasi ledger, RCE via DB. |
| **Kontrol Ada** | Parameterized query / ORM (SEC-07, SRS A03); DTO class-validator whitelist (API_CONTRACT §8.6); `forbidNonWhitelisted`; batas ukuran body 64 KB REST / 16 KB WS. |
| **Risiko Residu** | ORM misuse (raw query) di 1 endpoint; NoSQL-injection pada Redis key. |
| **Mitigasi** | Linter larang `raw query` tanpa review; fuzz test input; dependency scan (SEC-06); review kueri dinamis. |

### 3.11 Privilege Escalation Admin

| Item | Detail |
|---|---|
| **Vector** | GM rogue grant currency/item; bypass RBAC ke `/admin/*`; modifikasi audit log. |
| **Dampak** | Ekonomi banjir aset; penyalahgunaan; tutup jejak. |
| **Kontrol Ada** | RBAC (Player/Admin/GM), deny-by-default (SRS A01); MFA admin (SEC-07); `admin_audit_log` append-only immutable (API_CONTRACT §8.5); akses terbatas super-admin. |
| **Risiko Residu** | Super-admin compromise; log diakses pihak tak berwenang; GM sosial-engineering. |
| **Mitigasi** | 2FA wajib GM + step-up auth gran; alert grant besar / anomali; segregasi duty; rotate admin credential; review log berkala (NFR-PRIV-08). |

### 3.12 Bot / Farm / Auto-Clicker Idle

| Item | Detail |
|---|---|
| **Vector** | Multi-akun 1 device idle-farm; auto-clicker klaim; script bot main terus. |
| **Dampak** | Inflasi Gold/Gems; distorsi metrik DAU/revenue; rugi pemain legit. |
| **Kontrol Ada** | Device fingerprint ringan (FR-AUTH-13, ASP-01); rate limit flush (SAVE_DATA_SCHEMA §9.4); anomaly detection (SEC-16); threshold claim kecil. |
| **Risiko Residu** | Fingerprint mudah di-spoof (ASP-01 belum canggih); bot cerdas lolos pola statis. |
| **Mitigasi** | Captcha periodik pada pola mencurigakan; anomaly GDP alert (§6); korelasi device↔akun banyak; behavioral ML-lite (post-MVP F16). |

### 3.13 Payment Fraud / Chargeback

| Item | Detail |
|---|---|
| **Vector** | Fake webhook payment (grant Gems tanpa bayar); replay webhook; chargeback pasca top-up. |
| **Dampak** | Gems gratis (monetisasi bocor); kerugian finansial studio. |
| **Kontrol Ada** | Webhook HMAC verify (SEC-12, ASP-02); idempotensi order (cegah double-grant); Gems hanya masuk setelah webhook terverifikasi (SRS §4.4-5, FR-SHOP-05); batas withdraw/purchase. |
| **Risiko Residu** | Secret webhook leak; chargeback gateway; race order↔webhook. |
| **Mitigasi** | Verifikasi signature + status final gateway; idempotency key per order; pending→paid state machine; alert webhook gagal/asu; batas top-up harian (spending limit PRD F11). |

---

## 4. Trust Boundaries

### 4.1 Diagram ASCII

```
                          ┌─────────────────────────────────────────────┐
                          │  CLIENT (Phaser 3 PWA) — TIDAK DIPERCAYA     │
                          │  cache IndexedDB, intent-only, no secrets    │
                          └───────────────┬───────────────────┬─────────┘
                                          │ HTTPS / WS        │  (boundary A)
                                          │ intent + nonce    │
                                          ▼                   ▼
                          ┌─────────────────────────────────────────────┐
                          │  EDGE: CDN + Reverse Proxy + Rate Limiter    │
                          │  TLS, HSTS, WAF, CORS whitelist, limit_req   │
                          └───────────────┬─────────────────────────────┘
                                          │  (boundary B)
                                          ▼
              ┌───────────────────────────────────────────────────────────┐
              │  API LAYER (NestJS) — trust client input, verify boundary  │
              │  Auth(JWT+HMAC), DTO whitelist, idempotency, anti-replay    │
              │  RBAC, validate-intent, resolve server-authoritative        │
              └───────┬───────────────────────────────┬───────────────────┘
                      │  (boundary C)                 │  (boundary D)
                      ▼                               ▼
        ┌──────────────────────┐          ┌──────────────────────────────┐
        │  PostgreSQL          │          │  Redis                        │
        │  currency_ledger     │          │  session, nonce, rate-limit,  │
        │  (append-only)       │          │  lock distribusi, idempotency │
        │  wallets (CAS)       │          └──────────────┬───────────────┘
        │  gacha_pulls (immut) │                         │ BullMQ (queue)
        │  marketplace escrow  │                         ▼
        └──────────┬───────────┘          ┌──────────────────────────────┐
                   │  (boundary E)        │  External: Payment / Email /  │
                   ▼                      │  Analytics (webhook HMAC)     │
        ┌──────────────────────┐          └──────────────────────────────┘
        │  Admin/GM Panel       │
        │  (RBAC + MFA + audit) │
        └──────────────────────┘
```

### 4.2 Asumsi per Batas

| Batas | Asumsi Kepercayaan |
|---|---|
| **A — Client↔Edge** | Client tidak dipercaya (SC-01). Semua input dianggap beracun. Transport wajib `wss://`/HTTPS + HMAC anti-replay. |
| **B — Edge↔API** | Edge mempercayai TLS valid & rate-limit lolos. API tidak mempercayai header (kecuali signature terverifikasi). CORS whitelist cegah origin asing. |
| **C — API↔DB** | API mempercayai DB ACID. DB **tidak** mempercayai akses langsung — hanya via prosedur ledger (RLS). Least-privilege user (SEC-13). |
| **D — API↔Redis** | Redis simpan nonce/idempotency/session; dianggap volatile tapi konsisten. Tidak menyimpan secret. |
| **E — API↔Admin** | Admin dipercayai sebagian (RBAC + MFA), tapi **setiap aksi tercatat** audit immutable (SEC-11, API_CONTRACT §8.5). |
| **F — API↔External** | Payment/Email tidak dipercaya tanpa signature HMAC + idempotensi (SEC-12, ASP-02). |

---

## 5. Risk Rating

| # | Threat | Likelihood | Impact | Severity | Mitigasi Prioritas |
|---|---|---|---|---|---|
| T1 | Marketplace double-spend / race | Med | High | **High** | Atomic tx + UQ + idempotency (segera) |
| T2 | Item duplication | Med | High | **High** | Nonce UQ + daily scan (segera) |
| T3 | Gacha RNG manipulation | Low | High | **High** | Seeded RNG + pity lock + rate monitor |
| T4 | Idle accrual spoof | Med | Med | **Med** | Server clamp + nonce/ts + threshold |
| T5 | Ledger tamper | Low | High | **High** | Append-only + HMAC + CAS + RLS |
| T6 | Battle result forge | Med | Med | **Med** | Validate-result + sanity (D-16) |
| T7 | Auth brute force / stuffing | High | High | **High** | Lockout + rate limit + captcha |
| T8 | WS replay / session hijack | Med | High | **High** | Anti-replay + single session + httpOnly |
| T9 | XSS / CSRF | Med | Med | **Med** | Sanitasi + CSP + SameSite |
| T10 | SQL/injection | Low | High | **High** | Parameterized + DTO whitelist |
| T11 | Privilege escalation admin | Low | High | **High** | RBAC + MFA + audit immutable |
| T12 | Bot / farm / auto-clicker | High | Med | **Med** | Fingerprint + captcha + GDP alert |
| T13 | Payment fraud / chargeback | Low | High | **High** | Webhook HMAC + idempotensi + limit |

**Definisi:** Likelihood = frekuensi realistis (Low/Med/High). Impact = kerugian ekonomi/privasi (Low/Med/High). Severity = max(Likelihood, Impact) dengan bobot ekonomi (gacha/ledger/marketplace naik kelas).

---

## 6. Detection & Response

### 6.1 Alert & Signal

| Signal | Sumber | Aksi |
|---|---|---|
| Duplikat instance / `balance ≠ SUM(delta)` | Job scan harian (PRD §35, ERD §G) | Flag akun, freeze wallet, notif Security |
| Deviasi rate gacha dari 60/30/9/1 > batas | `gacha_pulls` aggregate (ERD §G.4) | Audit seed/pity; rollback bila terbukti bug |
| GDP anomaly (Gold/Gems melonjak per akun) | ledger aggregate per window | Threshold alert → review (SEC-16) |
| Idle claim 24j tiap <5 menit | `claimIdle` log (SAVE_DATA_SCHEMA §9.5) | Flag farm; captcha periodik |
| Login gagal >5 / spray pattern | `failed_login_count` (ERD B.2) | Lock + captcha + IP monitor |
| WS nonce reuse / ts kadaluarsa | API_CONTRACT §8.4 | 401 + flag device |
| Admin grant besar / anomali | `admin_audit_log` (§8.5) | Alert super-admin; review |
| Webhook payment gagal/asin | Payment provider | Pending hold; alert Ops |

### 6.2 Runbook (rujuk PRD §35 + INFRA)

1. **Duplikat terdeteksi** → freeze akun + wallet read-only; jalankan reconcile ledger vs `wallets.balance`; konfirmasi duplikat; rollback instance ilegal ke saldo valid; catat di audit log; buka investigasi (PRD §35).
2. **Gacha rate melenceng** → stop banner sementara; verifikasi HMAC per pull; cek `pity_state` konsisten; bila bug, hotfix + kompensasi transparan; bila curang, ban akun (SRS §3.2.3).
3. **GDP anomaly** → isolate akun; telusuri `currency_ledger` (ref_type); bila ledger tamper → restore dari backup + rotasi HMAC secret; bila bot → captcha + limit (SEC-16).
4. **Session hijack dicurigai** → revoke semua sesi user (FR-AUTH-07); force re-auth; cek IP/device anomali (INFRA rate-limit + WS stickiness).
5. **Payment fraud** → hold Gems (pending); verifikasi ulang signature webhook; bila palsu → batalkan grant, ban akun, laporkan gateway.

### 6.3 Observability (rujuk INFRA.md / NFR-OBS)

- Metric `ws_connections_active`, `api_error_rate`, `gacha_pulls_total`, `ledger_delta_sum` (Prometheus/Grafana).
- Structured log JSON (Pino/Loki); jangan log token/PII (SEC-07, NFR-OBS-01).
- Distributed tracing OpenTelemetry untuk request lintas service (NFR-OBS-03).

---

## 7. Residual Risk & Rekomendasi

### 7.1 Yang Belum Tertutup (Residual Risk)

| Area | Kesenjangan | Risiko |
|---|---|---|
| Multi-akun / farm | Fingerprint masih ringan (ASP-01); belum device-binding kuat | Med — inflasi lambat |
| Gacha fairness | Seed auditable tapi belum third-party verifiable (ASP-09) | Low — trust internal |
| Battle authority | Validate-result (D-16), belum full-replay di MVP | Med — exploit logic gap |
| Client hardening | Obfuscation best-effort, mudah dibongkar (R9, §9.4) | Low — server tetap otoritatif |
| Admin | 2FA ada di MVP? (PRD F10 sebut 2FA post-MVP) | Med — GM rogue |
| Webhook | Idempotensi ada, tapi race order↔webhook perlu uji beban | Low |
| Chat/XSS | Chat guild belum MVP (F17); saat masuk jadi vektor baru | Low (belum ada) |

### 7.2 Rekomendasi

1. **Penetration test ringan pra-launch** (SRS SEC-14, OWASP ZAP + abuse script §3.2.4) — wajib sebelum soft-launch.
2. **Security code review** fokus modul ekonomi: ledger procedure, marketplace tx, gacha service, auth (SEC-07).
3. **Tingkatkan 2FA admin ke MVP** (bukan post-MVP) mengingat risiko GM rogue (T11).
4. **Monitor deviasi rate gacha** dijadikan alert produksi (bukan sekadar audit) — lihat §6.1.
5. **Device-binding kuat** (device attestation) untuk kurangi multi-akun farm (ASP-01 upgrade).
6. **Full-replay battle verification** (D-16) segera setelah MVP stabil.
7. **Ledger HMAC verify berkala** (job) sebagai tamper-evidence aktif, bukan pasif.
8. **Chaos/load test** race condition marketplace & ledger pada spike 10x (SRS §7.1).

### 7.3 Skenario End-to-End (contoh integrasi kontrol)

**Skenario A — Penyerang coba duplikasi Gold via replay idle claim.**
1. Client (dimanipulasi) kirim `claimIdle` dengan `idempotencyKey` yang sudah pernah sukses → API tolak (SAVE_DATA_SCHEMA §9.2) atau kembalikan respons cached (API_CONTRACT §8.3).
2. Penyerang ubah clock lokal agar `last_seen` tampak 24j lalu → server abaikan, pakai `now` sendiri + clamp 24h (SRS §6.1, D-08).
3. Penyerang kirim 2x dalam 1 menit → reward < 60s = 0 (SRS §6.1).
4. Jika lolos, ledger tulis `+delta` dengan `nonce` unik; `balance_after CHECK >=0`; HMAC diverifikasi job (ERD §G.2). Anomali GDP → alert (§6.1).

**Skenario B — Penyerang coba double-buy 1 listing.**
1. Dua request `POST /market/buy` bersamaan → kedua masuk tx `SERIALIZABLE`; `SELECT...FOR UPDATE` kunci listing; satu sukses, lainnya `410` (sudah laku) (ERD §G.3, API_CONTRACT §3.6).
2. Replay dengan Idempotency-Key sama → respons cached, tidak eksekusi ganda (§8.3).
3. Self-trade antar 2 akun penyerang → device fingerprint + pola anomali → flag (SEC-16, ASP-01).

**Skenario C — GM rogue grant 1 juta Gems.**
1. Aksi lewat `/admin/grant` → tertulis `admin_audit_log` immutable (API_CONTRACT §8.5).
2. Grant besar → alert super-admin (§6.1); tanpa MFA (jika belum di MVP) risiko lebih tinggi → rekomendasi 2FA di MVP (§7.2-3).
3. Ledger `+1jt GEMS` dengan `ref_type=ADMIN` + signature → reconcile harian selaras (T5).

### 7.4 Kesimpulan

Arsitektur **server-authoritative** + **ledger append-only** + **escrow atomik** + **idempotency/anti-replay** menutup sebagian besar vektor ekonomi kritis (T1–T5, T13). Sisa risiko terkonsentrasi di **anti-abuse multi-akun** (T12), **admin trust** (T11), dan **battle verification** (T6) — seluruhnya membutuhkan pentest + code review pra-launch (SEC-14) sebelum rilis. Dokumen ini living document: perbarui tiap ada perubahan CDP/SRS/ERD/API_CONTRACT.

---

> **Traceability:** SRS §3.2 (OWASP/SEC/Anti-Cheat) · ERD §G (ledger) · API_CONTRACT §8 (security) · SAVE_DATA_SCHEMA §9 (anti-cheat) · PRD §35 (runbook) · DECISION_REGISTER D-08/D-16 · INFRA (monitoring).

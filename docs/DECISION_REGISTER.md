# Decision Register — RPG Fantasy Turn-Based Idle (Web + PWA)

**Status:** FINAL (locked) · **Tanggal:** 2026-07-14 · **Versi:** 1.0
**Sumber:** PRD §30 (Q1–Q8), SRS §8.4 (OQ-01..08), TDD §10.4 (OQ1..OQ7) — dideduplikasi menjadi **20 keputusan unik**.
**Aturan tunggal:** semua nilai harus tetap konsisten dengan *Canonical Design Parameters* (CDP). Jika bertentangan, **CDP menang**.

---

## 1. Cara Pakai

- Setiap keputusan punya ID `D-01`..`D-20`, nilai final, tier, pemilik, dan tanggal.
- Mengubah keputusan hanya lewat *change request* — catat di **§5 Log Perubahan** (jangan edit inline historis).
- Tier: **T1 = blokir (sebelum coding)** · **T2 = sebelum MVP (pakai default, tune di beta)** · **T3 = non-blokir (ops/infra)**.

---

## 2. Tabel Keputusan (20)

| ID | Topik | Nilai Final | Tier | Pemilik | Sumber |
|----|-------|-------------|------|---------|--------|
| D-01 | Matrix elemen (6×6) | Adopt draft: strong ×1.5 / netral ×1.0 / weak ×0.75 (cycle Fire→Water→Earth→Wind→Light↔Dark) | T2 | Game Designer | PRD Q1 / SRS OQ-03 |
| D-02 | Slope soft-pity | Ramp linier dari pull ke-75, naive rate naik tiap pull hingga hard-pity 90 | T2 | Game Designer | PRD Q2 |
| D-03 | Harga IAP & region + pajak | Gem pack + (pasca) Battle Pass; region mulai Asia (PDPA) lalu EU (GDPR); pajak *pass-through* via payment provider | T1 | Product / Biz / Legal | PRD Q3 |
| D-04 | Batas listing & bracket ELO | Max 50 listing aktif/user; bracket ELO tiap 200 poin | T2 | Game Designer | PRD Q4 |
| D-05 | Flow login / consent | Email/password + checkbox consent wajib; email-verify *soft* (tidak blokir gameplay) | T1 | Product | PRD Q5 |
| D-06 | Threshold refund & spending limit | Spending limit default **nonaktif** (opt-in); refund ikut kebijakan platform (≤48 jam) | T2 | Product / Legal | PRD Q6 |
| D-07 | Scope Battle Pass | **PASCA-LAUNCH** (tidak di MVP) | T1 | Product | PRD Q7 |
| D-08 | Toleransi skew server time | ±2 menit untuk perhitungan idle/offline | T2 | Backend | PRD Q8 |
| D-09 | Level cap hero & jumlah stage | Hero level cap **60** / Stage awal **100** | T2 | Game Designer | SRS OQ-01 |
| D-10 | Stamina di PvE | **YA** (P1): stamina harian, regen real-time, cap harian, untuk pacing + monetisasi | T1 | Product / Design | SRS OQ-02 |
| D-11 | K-factor ELO | **K = 32** | T2 | Game Designer | SRS OQ-04 |
| D-12 | Marketplace fee | **5%** per transaksi sukses | T2 | Product | SRS OQ-05 |
| D-13 | Dupe hero → progression | **YA** (P2): hero duplikat menaikkan star / ascension pemilik | T2 | Game Designer | SRS OQ-06 |
| D-14 | Mock server untuk dev | **YA**: MSW (atau stub `net/`) untuk dev & E2E saat backend belum siap | T1 | Backend / Client | SRS OQ-07 / TDD OQ1 |
| D-15 | Mode PvP | **Relay real-time via WebSocket** (turn-based, frekuensi rendah) untuk ladder MVP | T1 | Game Designer | SRS OQ-08 / TDD OQ2 |
| D-16 | Model otoritas battle | MVP: **server validasi hasil + sanity check**; full-replay (client kirim action log, server sim ulang) bila tingkat cheat naik (P2) | T1 | Tech Lead | TDD OQ3 |
| D-17 | Shard / region | Client agnostic: API URL via `env`; tidak ada shard logic di client | T3 | Backend / Ops | TDD OQ4 |
| D-18 | Sourcemap production | **YA**, namun privat (tidak publik) untuk error tracking | T3 | Ops | TDD OQ5 |
| D-19 | Strategi versi & forced update | Client cek `/meta/config` (version); forced update bila CDP/balance berubah breaking | T3 | Client Lead | TDD OQ6 |
| D-20 | Region store / compliance | **Dual**: consent UI dukung GDPR & PDPA; default **PDPA (Asia)** di MVP, toggle **EU (GDPR)** saat rilis | T1 | Legal / PM | TDD OQ7 |

---

## 3. Detail Tier 1 (Pilihan Genuin — butuh konfirmasi pemilik)

- **D-03 (IAP & region):** MVP hanya jual **Gem pack** (mis. 60/330/780/2000 Gems ≈ $0.99/$4.99/$9.99/$49.99). Battle Pass (D-07) menyusul pasca-launch. Pajak & regional pricing ditangani payment provider (Stripe/PlayStore); jangan hardcode di client.
- **D-05 (login/consent):** Register wajib ada 1 checkbox consent (GDPR/PDPA). Email verifikasi dikirim, tapi akun bisa main meski belum verify (soft). Fitur sensitif (trade besar, withdraw) baru butuh email terverifikasi.
- **D-07 (battle pass):** Tidak masuk MVP → kurangi scope tim kecil. Monetisasi MVP = Gem IAP + ads opsional (rewarded).
- **D-10 (stamina):** Stamina mencegah power-leveling & jadi sink sekaligus driver IAP (beli stamina). Regen 1/3 menit, cap 120, refill penuh tiap reset harian.
- **D-14 (mock server):** Wajib agar client & backend bisa jalan paralel. `MSW` intercept `/api/*` dengan fixture dari `API_CONTRACT.md`.
- **D-15 (PvP):** Relay WS dipilih (bukan async mail) agar ladder terasa hidup & mendukung notifikasi real-time; beban rendah karena turn-based.
- **D-16 (battle authority):** Validate-result cukup untuk MVP (latensi rendah, simpel). Bila deteksi cheat naik, naikkan ke full-replay. Client tetap **bukan** sumber kebenaran (currency/drop/gacha selalu server).
- **D-20 (compliance):** Satu build, consent layer bisa switch region. PDPA (Asia) default karena target DAU awal Asia; siapkan flag EU sebelum rilis ke Eropa.

---

## 4. Sign-Off

| Peran | Nama | Tanggal | Status |
|-------|------|---------|--------|
| Product / PM | ___________ | __________ | ☐ |
| Game Designer | ___________ | __________ | ☐ |
| Tech Lead | ___________ | __________ | ☐ |
| Backend Lead | ___________ | __________ | ☐ |
| Legal / Compliance | ___________ | __________ | ☐ |

> Setelah semua ☐ terisi, dokumen PRD/SRS/TDD berstatus **FINAL** dan section Open Questions dihapus (diganti referensi ke file ini).

---

## 5. Log Perubahan

| Tanggal | ID | Perubahan | Pembuat |
|---------|----|-----------|---------|
| 2026-07-14 | — | Initial lock D-01..D-20 (dari PRD Q1–8, SRS OQ-01–08, TDD OQ1–7) | Orchestrator |

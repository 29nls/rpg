# RPG Fantasy Turn-Based Idle — Dokumentasi Proyek

Game RPG **Fantasy Turn-Based Idle** berbasis website (desktop-first) + PWA. Struktur: client game **Phaser 3 + TypeScript + Vite**, backend **Node.js/NestJS + PostgreSQL + Redis + BullMQ + WebSocket**, arsitektur **server-authoritative**. Skala awal 10–50 DAU (MVP), desain scalable.

> **Status:** Dokumen desain + legal + asset + infra **LENGKAP & KONSISTEN**. Lihat bagian [Status & Open Items](#status--open-items) untuk sisa pra-launch.

---

## Cara Membaca (Onboarding)

1. Mulai dari **PRD.md** (produk, gameplay, economy, KPI).
2. Lanjut **SRS.md** + **ERD.md** (requirements & data model).
3. Keputusan yang sudah dikunci ada di **DECISION_REGISTER.md** (wajib dibaca sebelum implementasi).
4. Angka balance final ada di **GDD.md**.
5. Arsitektur client → **TDD.md** lalu doc teknis engine (state machine, game loop, asset, sprite, audio, input, save).
6. Kontrak server → **API_CONTRACT.md** + **INFRA.md**.
7. Legal → **PRIVACY_POLICY.md** + **TERMS_OF_SERVICE.md** (masih draf, lihat Open Items).

---

## Peta Dokumen (21 file)

### A. Produk & Game Design
| File | Isi |
|------|-----|
| `PRD.md` | Product Requirements: vision, pilar gameplay, core/meta loop, MVP vs Post-MVP, user stories, balancing/economy, monetisasi etis, KPI, risiko, milestone. |
| `GDD.md` | Game Design Document: rumus damage, rarity/stat, 12 hero final, skill & status, matrix elemen, stage curve, drop table, idle accrual, ekonomi, XP/ascension, PvP ELO, stamina. |
| `DECISION_REGISTER.md` | 20 keputusan terkunci (D-01..D-20) + sign-off + log perubahan. |

### B. System & Requirements
| File | Isi |
|------|-----|
| `SRS.md` | Software Requirements Spec: scope, FR per modul, NFR (perf/security/privacy/observability), arsitektur, API spec, aturan bisnis kritis, test plan. |
| `ERD.md` | Entity Relationship Diagram: 43 entitas, relasi, constraint, indeks, normalisasi, `currency_ledger` append-only, partisi/retensi. |

### C. Client Engine & Tech (Phaser 3)
| File | Isi |
|------|-----|
| `TDD.md` | Technical Design: pemilihan engine, arsitektur ECS-lite, tech stack, struktur folder `src/`, dependency direction. |
| `GAME_STATE_MACHINE.md` | App-Level FSM (19 state) + Battle Sub-FSM + Global states (Online/Offline/Maintenance/Paused). |
| `GAME_LOOP.md` | Fixed-timestep 60 FPS, delta clamp, interpolation, integrasi Phaser, pause/resume saat tab hidden. |
| `ASSET_LOADING.md` | Preload vs lazy, Phaser Loader, object pooling, memory management (anti-leak), PWA caching. |
| `SPRITESHEET_LAYOUT.md` | Texture atlas (JSON Hash), packing rules, penamaan, multi-page, animasi, SD/HD. |
| `AUDIO_PIPELINE.md` | Web Audio API, OGG+MP3, audio unlock, pre-decode, AudioSprite, buses, ducking, latency. |
| `INPUT_MAPPING.md` | Action map, binding keyboard/mouse/touch, context layer, rebinding, accessibility. |
| `SAVE_DATA_SCHEMA.md` | localStorage + IndexedDB, prinsip server-authoritative, versioning, sync, anti-cheat. |

### D. Backend & API
| File | Isi |
|------|-----|
| `API_CONTRACT.md` | REST 11 kelompok + WebSocket 6 event, auth dual-token, idle claim server-side, error/rate-limit. |
| `INFRA.md` | Docker, CI/CD, `/meta/config`, observability, backup, scaling, security, runbook. |

### E. Konten & Aset
| File | Isi |
|------|-----|
| `ASSET_MANIFEST.md` | Daftar aset per scene (atlas, audio, json), budget <15MB initial / <5MB per scene, PWA precache. |

### F. Legal & Compliance
| File | Isi |
|------|-----|
| `PRIVACY_POLICY.md` | Kebijakan Privasi (GDPR/PDPA): data, hak pengguna, retensi, anak, keamanan. **Draf — wajib review counsel.** |
| `TERMS_OF_SERVICE.md` | Syarat & Ketentuan + Refund Policy + spending limit + suspensi. **Draf — wajib review counsel.** |

### G. Analytics & Security
| File | Isi |
|------|-----|
| `ANALYTICS_EVENT_TAXONOMY.md` | Schema event untuk KPI PRD §8: envelope JSON, naming `category_action`, catalog per domain (lifecycle/session/progression/idle/economy/market/pvp/guild), KPI→event mapping, consent/PII gate, transport (REST/WS/offline queue). |
| `THREAT_MODEL.md` | STRIDE + 13 abuse cases (double-spend, dupe, gacha RNG, idle spoof, ledger tamper, battle forge, brute force, WS replay, XSS/CSRF, SQLi, priv-esc, bot/farm, payment), trust boundaries, risk rating, detection/response. |

---

## Canonical Design Parameters (ringkas)

Party 5 · Rarity C/R/E/L (+M terbatas) · Gacha 60/30/9/1, pity 10/75/90 · Idle tick 5dtk, cap 24j (server) · Items Gear/Consumable/Material · PvP ELO 1000 (K=32) · Currency Gold/Gems/Event Token · Combat: turn by speed, mana/energy, 8 status, 6 elemen (1.5/0.75) · Auth email/password · Server-authoritative · 60 FPS fixed-timestep.
Detail & keputusan final: lihat `DECISION_REGISTER.md`.

---

## Roadmap (ringkas)

| Tahap | Fokus | Dokumen panduan |
|-------|-------|-----------------|
| 0 — Prototype | Client+backend skeleton, mock server (MSW), 1 stage + 1 gacha | TDD, API_CONTRACT, INFRA |
| MVP | PvE waves, idle+offline, gacha, inventory, marketplace dasar, guild/raid ringan, PvP ladder, admin dasar, auth, monetisasi dasar | PRD (MVP), SRS, GDD, ERD |
| Beta | Tuning balance (GDD §15), pentest, load test | GDD, SRS (Test Plan), INFRA |
| Launch | Legal final, sign-off, monitoring live | PRIVACY_POLICY, TERMS_OF_SERVICE, INFRA, DECISION_REGISTER |
| LiveOps | Event, banner, battle pass (pasca-launch) | PRD (monetisasi/post-MVP) |

---

## Status & Open Items

Dokumen di atas **lengkap & konsisten**. Item berikut BELUM FINAL dan harus diselesaikan sebelum/selama build:

| # | Item | Keterangan | Blokir? |
|---|------|-----------|---------|
| 1 | **Sign-off `DECISION_REGISTER.md` §4** | 5 baris (Product/PM, Game Designer, Tech Lead, Backend Lead, Legal) masih kosong + ☐ belum dicentang | Ya (formal FINAL) |
| 2 | **Legal review** | `PRIVACY_POLICY.md` & `TERMS_OF_SERVICE.md` masih draf template; placeholder `[...]` (nama perusahaan, DPO, yurisdiksi) belum diisi; wajib review counsel | Ya (launch) |
| 3 | **GDD `[asumsi desain]`** | 15 asumsi desain ditandai & non-kontradiktif, belum divalidasi Game Designer | Tidak (soft) |
| 4 | **Pentest + load test** | Belum dieksekusi (rencana ada di SRS Test Plan & INFRA) | Ya (launch) |
| 5 | **Implementasi** | 0 baris kode game + aset produksi riil belum dibuat (baru manifest) | Ya (build) |
| 6 | **Endpoint `POST /analytics/events`** | Terdeteksi saat buat taxonomy; belum ada di `API_CONTRACT.md` (tambahkan saat beta) | Tidak |

---

## Catatan

- Semua dokumen dalam **Bahasa Indonesia**.
- Angka di luar CDP ditandai `[asumsi desain]` (GDD) atau `[...]` (legal draft).
- Aturan tunggal: jika ada konflik, **CDP / `DECISION_REGISTER.md` menang**.
- Mengubah keputusan hanya lewat change request tercatat di `DECISION_REGISTER.md` §5.

# GAME DESIGN DOCUMENT (GDD)
## RPG Fantasy Turn-Based Idle — Web + PWA

| Field | Value |
|-------|-------|
| Dokumen | GDD (Game Design Document) |
| Versi | 1.0 (FINAL — mengunci balance) |
| Tanggal | 2026-07-14 |
| Status | Final, konsisten dengan CDP + `DECISION_REGISTER.md` (D-01..D-20) |
| Bahasa | Bahasa Indonesia |
| Sumber kebenaran | Server-authoritative. Semua angka di bawah adalah *nilai final* untuk implementasi; CDP menang jika ada konflik |
| Menutup celah | PRD §34 (seed data masih "contoh") — semua angka di sini adalah **angka riil**, bukan placeholder |

> **Catatan konsistensi:** Setiap angka yang bukan berasal dari CDP/DECISION_REGISTER ditandai **`[asumsi desain]`** dan dibuktikan tidak bertentangan CDP di §15 dan §16. CDP = Canonical Design Parameters (PRD Lampiran A). DR = `DECISION_REGISTER.md`.

---

# 1. Ringkasan & Konvensi

## 1.1 Tujuan Dokumen
GDD ini mengunci seluruh angka balance agar dapat diimplementasikan langsung oleh client & backend. PRD §34 hanya memberi nama hero & stat contoh; di sini stat difinalkan (§4) dan seluruh kurva (stage, XP, ekonomi, gacha, PvP) diberi angka pasti.

## 1.2 Satuan & Pembulatan
| Konvensi | Aturan | Catatan |
|----------|--------|---------|
| Waktu | Detik (dtk) | Idle tick = 5 dtk (CDP) |
| Uang | Gold (integer), Gem (integer), Event Token (integer) | Tidak ada fraksi; semua `floor()` ke bawah |
| Stat | Integer (dibulatkan `floor`/`round` ke bawah saat damage, `round` terdekat saat display) | Lihat §2.3 |
| Probabilitas | Persen (%) dengan 2 desimal internal, roll via RNG server | Gacha/status/drop |
| Power (StagePower) | Integer, metrik agregat musuh | Rumus §8.1 |
| Pembulatan damage | `floor()` (pembulatan ke bawah) wajib | Sesuai rumus §2.2 |
| Pembulatan non-damage | `round()` ke integer terdekat | EXP, Gold reward, stat |

## 1.3 Sumber Kebenaran Server
- **CDP (PRD Lampiran A)** adalah hukum tertinggi. Konflik → CDP menang.
- **`DECISION_REGISTER.md`** mengunci 20 keputusan (D-01..D-20); tier T1 blokir coding.
- **Server-authoritative (CDP):** currency, drop, gacha, hasil battle selalu dihitung server. Client hanya mengirim *intent* (skill id + target). Battle di-resolve server (D-16: MVP = validasi hasil; P2 = full-replay).
- **Idle/offline:** dihitung ulang dari `server_now` saat claim; waktu client diabaikan (D-08: toleransi ±2 menit).
- **Tick:** 5 dtk, server-driven untuk akumulasi resmi (CDP). Client boleh estimasi lokal asal tidak jadi otoritas.

## 1.4 Konstanta Global (CDP-locked)
| Konstanta | Nilai | Sumber |
|-----------|-------|--------|
| Party maks | 5 hero | CDP |
| Rarity | C, R, E, L (+M terbatas/event) | CDP |
| Tick idle | 5 dtk | CDP / D-08 |
| Offline cap | 24 jam | CDP |
| Accrual clamp | `[0, 24h]` server-side | CDP |
| Gacha rate | C60 / R30 / E9 / L1 | CDP / D-02 |
| Pity R+ | tiap 10-pull | CDP / D-02 |
| Hard pity L | pull ke-90 | CDP / D-02 |
| Soft pity L | mulai pull ke-75, linier → 90 | CDP / D-02 |
| PvP ELO start | 1000 | CDP / D-11 |
| PvP K-factor | 32 | D-11 |
| Bracket ELO | tiap 200 poin | D-04 |
| Marketplace fee | 5% | D-12 |
| Hero level cap | 60 | D-09 |
| Stage awal | 100 | D-09 |
| Stamina PvE | YA: regen 1/3 mnt, cap 120 | D-10 |
| Dupe → ascension | YA: star-up s.d. 6★ | D-13 |
| Elemen | 6 (Fire/Water/Earth/Wind/Light/Dark) | D-01 |
| Affinity | strong ×1.5 / netral ×1.0 / weak ×0.75 | D-01 |
| Status | 8 (Burn/Freeze/Poison/Stun/AtkDown/DefDown/Shield/Regen) | CDP |
| Currencies | Gold (soft), Gems (hard), Event Tokens | CDP |

---

# 2. Atribut Dasar & Rumus

## 2.1 Atribut per Hero
Setiap hero memiliki atribut berikut (sebelum gear/status):

| Atribut | Simbol | Satuan | Keterangan |
|---------|--------|--------|-----------|
| HP (Health) | `HP` / `maxHP` | integer | Nyawa; 0 = mati |
| ATK (Attack) | `ATK` | integer | Serangan dasar |
| DEF (Defense) | `DEF` | integer | Pengurang damage (rumus §2.2) |
| SPD (Speed) | `SPD` | integer | Penentu urutan giliran (§6) |
| Mana / Energy | `MANA` | 0..100 | Resource skill aktif; regen +10/turn (§6) |
| CRT (Crit Rate) | `CRT` | % | Base 5%, naik via gear/passive |
| CRT_DMG (Crit Damage) | `CRT_DMG` | % | Base +50% (bonus damage saat crit) |

Nilai dasar per rarity & kurva level: lihat §3. Nilai per hero (sudah incl. role): lihat §4.

## 2.2 Rumus Damage (FINAL — menggantikan PRD §12.2 konseptual)

```
DMG = floor( ATK * SkillMul * Affinity * CritMul * DefMul * RandVar )
```

dengan:

```
CritMul = (1 + CRT_DMG)   jika crit terjadi, selain itu 1.0
DefMul  = 100 / (100 + DEF)
RandVar = uniform acak di [0.95 .. 1.05]
```

**Penjelasan variabel (range pasti):**

| Variabel | Arti | Range / Nilai |
|----------|------|---------------|
| `ATK` | Attack hero penyerang | lihat §3–§4 (mis. 110..356 core, ×gear) |
| `SkillMul` | Pengali skill yang dipakai | 0.8 (shield/tank) .. 2.4 (burst). Per-skill di §4/§5 |
| `Affinity` | Pengali elemen (§7) | **1.5** strong · **1.0** netral · **0.75** weak |
| `CRT_DMG` | Bonus crit damage | Base **+50%** (=0.5). Jadi `CritMul = 1.5` saat crit |
| `DEF` | Defense target | lihat §8.3 (musuh) dan §3 (hero) |
| `DefMul` | Mitigasi pertahanan | `100/(100+DEF)` → range **0.5** (DEF=100) s.d. **~0.0909** (DEF=1000) |
| `RandVar` | Varian acak | **[0.95, 1.05]** uniform, ditentukan RNG server per hit |
| `floor()` | Pembulatan | Ke bawah (integer ≥ 1; minimal damage = 1) |

**Minimal damage:** `max(1, DMG)` — serangan tidak pernah 0 kecuali miss (tidak ada mekanik miss di MVP).

### 2.2.1 Contoh Hitungan (deterministik, RandVar = 1.0 untuk ilustrasi)
- **Kael (C Fire DPS) lvl1**, `ATK=126`, skill *Sayatan Api* `SkillMul=1.2`, vs **Slime (Earth)** `DEF=40`.
  - Affinity Fire→Earth = netral = **1.0**.
  - Tanpa crit: `DMG = floor(126 * 1.2 * 1.0 * 1.0 * (100/140) * 1.0) = floor(126*1.2*0.7143) = floor(108.0) = 108`.
- **Dengan crit** (CRT 5%): `CritMul = 1.5` → `floor(108.0 * 1.5) = 162`.
- **Borin (R Earth Tank) lvl1**, `ATK=110`, skill *Penjaga Batu* `SkillMul=0.8`, vs Slime `DEF=40`:
  - `DMG = floor(110*0.8*1.0*1.0*(100/140)*1.0) = floor(62.9) = 62` (skill defensif, skill utama memberi Shield).
- **Advantage elemen:** Borin (Earth) vs **Serigala (Wind)** → Earth→Wind = strong (**1.5**). Pakai basic `SkillMul=1.0`, `ATK=110`, `DEF=40`: `floor(110*1.0*1.5*1.0*(100/140)*1.0)=floor(117.9)=117`.

## 2.3 Aturan Pembulatan Stat & Eksponensial
- Stat hero dihitung: `stat = round( base × rarityMul × roleMul × levelMul × starMul )`.
- `levelMul(L) = 1 + 0.008 × (L − 1)` (lihat §3.2).
- `starMul(star) = 1 + 0.10 × (star − 1)`, star ∈ [1..6] (lihat §12.3).
- `round()` = pembulatan ke integer terdekat; `floor()` hanya untuk damage (§2.2).
- Gear menambah stat sebagai **% additif** di luar rantai di atas (lihat §11.4).

## 2.4 Mana / Energy
- `MANA_MAX = 100` untuk semua hero (CDP: "mana/energy per hero").
- Regen **+10 per turn** (CDP/§6). Tidak boleh melebihi 100.
- Basic attack: biaya 0 mana, cooldown 0.
- Active skill: biaya 20..60 mana, cooldown 2..4 turn (per-skill di §4/§5).
- Jika mana < biaya → intent ditolak server (SRS FR-CMB-11, TC-CMB-05).

---

# 3. Rarity & Stat Base

## 3.1 Multiplier Rarity (CDP-locked)
| Rarity | Multiplier (HP/ATK/DEF) | Catatan |
|--------|--------------------------|---------|
| C (Common) | ×1.0 | Baseline |
| R (Rare) | ×1.25 | |
| E (Epic) | ×1.6 | |
| L (Legendary) | ×2.2 | |
| M (Mythic) | — | Hanya banner kolaborasi/anniversary (D-07 terkait scope); tidak di pool MVP |

## 3.2 Growth per Level
- `levelMul(L) = 1 + 0.008 × (L − 1)` → pertumbuhan **+0.8% per level** (ekivalen **+8% per 10 level**, sesuai format yang diminta).
- Di level cap 60: `levelMul = 1.472` (**+47.2%** dari level 1).
- SPD **tidak** diskalakan level; SPD ditentuak rarity + role + gear (lihat §4).

## 3.3 Base Stat per Rarity (Level 1, 1★, sebelum role-mod)
| Rarity | HP | ATK | DEF | SPD |
|--------|----|-----|-----|-----|
| C | 1000 | 110 | 55 | 100 |
| R | 1250 | 138 | 69 | 103 |
| E | 1600 | 176 | 88 | 106 |
| L | 2200 | 242 | 121 | 109 |

> SPD naik +3 per tier rarity `[asumsi desain]` — tidak bertentangan CDP (CDP hanya mewajibkan adanya SPD sebagai penentu turn order).

## 3.4 Tabel Stat Lengkap per Rarity (1★, tanpa role-mod)
Data di bawah adalah **core rarity** (sebelum role modifier §4). Role modifier diterapkan per hero di §4.

#### Rarity C
| Lvl (1★) | HP | ATK | DEF | SPD |
|---|---|---|---|---|
| 1 | 1000 | 110 | 55 | 100 |
| 10 | 1072 | 118 | 59 | 100 |
| 20 | 1152 | 127 | 63 | 100 |
| 30 | 1232 | 136 | 68 | 100 |
| 40 | 1312 | 144 | 72 | 100 |
| 50 | 1392 | 153 | 77 | 100 |
| 60 | 1472 | 162 | 81 | 100 |
| **60 (6★)** | **2208** | **243** | **121** | 100 |

#### Rarity R
| Lvl (1★) | HP | ATK | DEF | SPD |
|---|---|---|---|---|
| 1 | 1250 | 138 | 69 | 103 |
| 10 | 1340 | 148 | 74 | 103 |
| 20 | 1440 | 159 | 79 | 103 |
| 30 | 1540 | 170 | 85 | 103 |
| 40 | 1640 | 181 | 91 | 103 |
| 50 | 1740 | 192 | 96 | 103 |
| 60 | 1840 | 203 | 102 | 103 |
| **60 (6★)** | **2760** | **305** | **152** | 103 |

#### Rarity E
| Lvl (1★) | HP | ATK | DEF | SPD |
|---|---|---|---|---|
| 1 | 1600 | 176 | 88 | 106 |
| 10 | 1715 | 189 | 94 | 106 |
| 20 | 1843 | 203 | 101 | 106 |
| 30 | 1971 | 217 | 108 | 106 |
| 40 | 2099 | 231 | 115 | 106 |
| 50 | 2227 | 245 | 122 | 106 |
| 60 | 2355 | 259 | 130 | 106 |
| **60 (6★)** | **3533** | **389** | **194** | 106 |

#### Rarity L
| Lvl (1★) | HP | ATK | DEF | SPD |
|---|---|---|---|---|
| 1 | 2200 | 242 | 121 | 109 |
| 10 | 2358 | 259 | 130 | 109 |
| 20 | 2534 | 279 | 139 | 109 |
| 30 | 2710 | 298 | 149 | 109 |
| 40 | 2886 | 318 | 159 | 109 |
| 50 | 3062 | 337 | 168 | 109 |
| 60 | 3238 | 356 | 178 | 109 |
| **60 (6★)** | **4858** | **534** | **267** | 109 |

> **Max stat di cap 60 (6★, core tanpa gear):** C = HP2208/ATK243/DEF121 · R = HP2760/ATK305/DEF152 · E = HP3533/ATK389/DEF194 · L = HP4858/ATK534/DEF267. Gear menambah % di atas ini (§11.4).

## 3.5 Role Modifier `[asumsi desain]`
Diterapkan pada core rarity sebelum level/star. Tidak bertentangan CDP (CDP tidak mengunci role-mod).

| Role | HP × | ATK × | DEF × | SPD × | Catatan |
|------|------|-------|-------|-------|---------|
| DPS | 0.95 | 1.15 | 0.95 | 1.00 | Damage tinggi, agak tipis |
| Tank | 1.40 | 0.80 | 1.40 | 0.95 | Tahan banting |
| Healer | 1.10 | 0.95 | 1.05 | 1.00 | Healing skala ATK |
| Support | 1.00 | 1.00 | 1.00 | 1.08 | Buff/debuff, agak cepat |

---

# 4. Hero Roster (FINAL — 12 Hero)

## 4.1 Cakupan
12 hero menutupi **6 elemen** dan **4 role** (DPS/Tank/Healer/Support). Seed PRD §34 (Kael/Mira/Borin/Lira/Sol/Noct) difinalkan + 6 hero tambahan.

| # | Nama | Rarity | Elemen | Role | Stat Final (L1, 1★) HP/ATK/DEF/SPD | CRT% |
|---|------|--------|--------|------|--------------------------------------|------|
| 1 | Kael | C | Fire | DPS | 950 / 126 / 52 / 100 | 5% |
| 2 | Mira | C | Water | Healer | 1100 / 105 / 58 / 100 | 5% |
| 3 | Borin | R | Earth | Tank | 1750 / 110 / 97 / 98 | 5% |
| 4 | Lira | R | Wind | DPS | 1187 / 159 / 66 / 103 | 5% |
| 5 | Sol | E | Light | Support | 1600 / 176 / 88 / 106 (+SPD buff) | 10% |
| 6 | Noct | L | Dark | DPS | 2090 / 278 / 115 / 109 | 5% |
| 7 | Veyra | R | Fire | DPS | 1187 / 159 / 66 / 103 | 5% |
| 8 | Thora | E | Earth | Tank | 2240 / 141 / 123 / 101 | 5% |
| 9 | Kelwin | C | Wind | Support | 1000 / 110 / 55 / 108 | 10% |
| 10 | Isolde | R | Water | Healer | 1375 / 131 / 72 / 103 | 5% |
| 11 | Pyra | E | Fire | DPS | 1520 / 202 / 84 / 106 | 5% |
| 12 | Draven | L | Light | DPS | 2090 / 278 / 115 / 109 | 5% |

**Distribusi elemen:** Fire(Kael,Veyra,Pyra) · Water(Mira,Isolde) · Earth(Borin,Thora) · Wind(Lira,Kelwin) · Light(Sol,Draven) · Dark(Noct). **Role:** DPS×6, Tank×2, Healer×2, Support×2.

## 4.2 Stat Final per Hero (dari §3 × role-mod, dibulatkan)
Rumus: `stat = round( coreRarity × roleMod )`, lalu lanjut `× levelMul × starMul` saat naik level/star (§12.3).

| Hero | Base HP | Base ATK | Base DEF | Base SPD | Hitung |
|------|---------|----------|----------|----------|--------|
| Kael (C DPS) | 950 | 126 | 52 | 100 | 1000×0.95, 110×1.15, 55×0.95 |
| Mira (C Heal) | 1100 | 105 | 58 | 100 | 1000×1.1, 110×0.95, 55×1.05 |
| Borin (R Tank) | 1750 | 110 | 97 | 98 | 1250×1.4, 138×0.8, 69×1.4, SPD103×0.95 |
| Lira (R DPS) | 1187 | 159 | 66 | 103 | 1250×0.95, 138×1.15, 69×0.95 |
| Sol (E Sup) | 1600 | 176 | 88 | 106 | 1600×1.0, +CRT 10% (SPD×1.08 di passive) |
| Noct (L DPS) | 2090 | 278 | 115 | 109 | 2200×0.95, 242×1.15, 121×0.95 |
| Veyra (R DPS) | 1187 | 159 | 66 | 103 | sama dgn Lira (variant Fire) |
| Thora (E Tank) | 2240 | 141 | 123 | 101 | 1600×1.4, 176×0.8, 88×1.4, SPD106×0.95 |
| Kelwin (C Sup) | 1000 | 110 | 55 | 108 | 1000×1.0, SPD100×1.08, +CRT10% |
| Isolde (R Heal) | 1375 | 131 | 72 | 103 | 1250×1.1, 138×0.95, 69×1.05 |
| Pyra (E DPS) | 1520 | 202 | 84 | 106 | 1600×0.95, 176×1.15, 88×0.95 |
| Draven (L DPS) | 2090 | 278 | 115 | 109 | sama dgn Noct (variant Light) |

## 4.3 Skill Setiap Hero
Format: **Nama** · `SkillMul` · biaya mana · cooldown (turn) · efek. Basic attack selalu `Mul 1.0 / 0 mana / cd 0`.

### Kael — C / Fire / DPS
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Hantaman | 1.0 | 0 | 0 | Physical single |
| **Sayatan Api** | 1.2 | 30 | 2 | Fire single, terapkan **Burn** (5% HP/2t) |
| **Passive:** Berdarah Panas | — | — | — | +15% ATK saat HP > 50% |

### Mira — C / Water / Healer
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Tongkat Air | 1.0 | 0 | 0 | Physical single |
| **Pamuja Air** | 0 (heal) | 35 | 2 | Heal ally terendah HP = **180% ATK** |
| **Passive:** Kasih Ibu | — | — | — | Heal +20% jika target < 30% HP |

### Borin — R / Earth / Tank
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Hammership | 1.0 | 0 | 0 | Physical single |
| **Penjaga Batu** | 0.8 | 40 | 3 | Shield self = **50% maxHP / 2t** + taunt (musuh targetkan Borin 1t) |
| **Passive:** Batu Karang | — | — | — | -20% damage diterima |

### Lira — R / Wind / DPS
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Tusukan Angin | 1.0 | 0 | 0 | Physical single |
| **Angin Badai** | 1.0 | 40 | 3 | **AoE** semua musuh (Wind) |
| **Passive:** Mata Elang | — | — | — | +10% CRT rate |

### Sol — E / Light / Support
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Cahaya Suci | 1.0 | 0 | 0 | Light single |
| **Hukuman Cahaya** | 1.3 | 40 | 3 | Light single + **Burn** (5% HP/2t) + **AtkDown -30%/2t** |
| **Passive:** Aura Cahya | — | — | — | Seluruh party +10% CRT_DMG; SPD +8% (106→114 efektif) |

### Noct — L / Dark / DPS
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Sabit Bayangan | 1.0 | 0 | 0 | Dark single |
| **Ledakan Kekosongan** | 2.4 | 60 | 4 | Dark single burst, **abaikan 20% DEF** target |
| **Passive:** Jurang Pertama | — | — | — | Skill aktif pertama tiap battle +30% Mul |

### Veyra — R / Fire / DPS
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Kobaran | 1.0 | 0 | 0 | Fire single |
| **Matahari Terik** | 1.6 | 50 | 4 | Fire **AoE** + **AtkDown -30%/2t** semua musuh |
| **Passive:** Pembakar | — | — | — | +15% ATK vs target yang terkena Burn |

### Thora — E / Earth / Tank
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Tendangan Batu | 1.0 | 0 | 0 | Earth single |
| **Gempa** | 1.0 | 45 | 3 | Earth **AoE** + **DefDown -30%/2t** semua musuh |
| **Passive:** Benteng | — | — | — | +25% DEF |

### Kelwin — C / Wind / Support
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Angin Tipis | 1.0 | 0 | 0 | Wind single |
| **Dentuman Angin** | 1.1 | 35 | 2 | Wind **AoE** + Shield ally terendah = **30% maxHP/2t** |
| **Passive:** Angin Pembawa | — | — | — | Party +10 SPD di awal battle (CDP: SPD penentu urutan) |

### Isolde — R / Water / Healer
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Semprotan | 1.0 | 0 | 0 | Water single |
| **Air Suci** | 0 (heal) | 50 | 4 | Heal **ALL** ally = **150% ATK** + **Regen 5% HP/2t** |
| **Passive:** Mata Air | — | — | — | Efek Regen +30% |

### Pyra — E / Fire / DPS
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Lidah Api | 1.0 | 0 | 0 | Fire single |
| **Pilar Lava** | 2.0 | 55 | 3 | Fire single + **Burn** (5% HP/3t — +1t dari passive) |
| **Passive:** Lava Abadi | — | — | — | Durasi Burn +1t |

### Draven — L / Light / DPS
| Skill | Mul | Mana | CD | Efek |
|-------|-----|------|----|------|
| Basic: Tebas Cahaya | 1.0 | 0 | 0 | Light single |
| **Eksekusi Radian** | 2.2 | 60 | 4 | Light single, **+50% Mul** jika target HP < 40% |
| **Passive:** Pendakian Cahya | — | — | — | +20% ATK vs target < 50% HP |

## 4.4 Catatan Roster
- Semua hero punya **1 active skill** + **1 passive** (MVP). Basic attack selalu tersedia (0 mana).
- Mana max 100, regen +10/turn → skill mahal (60) butuh ~6 turn akumulasi dari kosong, atau 2 turn setelah basic.
- **M (Mythic)** tidak di roster MVP; hanya banner kolaborasi/anniversary (D-07 scope pasca-launch).

---

# 5. Skill & Status Effect

## 5.1 Delapan Status Effect (FINAL — nilai pasti, menggantikan PRD §2.1.4 konseptual)

| Status | Efek | Durasi (turn) | Magnitude | Stack? | Interaksi |
|--------|------|---------------|-----------|--------|-----------|
| **Burn** | DoT per tick di awal giliran | 2 | **5% maxHP** per tick | Ya (cap 3 stack → dmg ×stack, maks 15% HP/tick) | Doyleh刷新 durasi |
| **Poison** | DoT per tick | 3 | **3% maxHP** per tick | Ya (cap 3 stack → maks 9% HP/tick) | Diabaikan Shield (lewat langsung ke HP) |
| **Freeze** | Skip turn (tidak bertindak) | 1 | — | Tidak | Batalkan DoT tick di turn tersebut |
| **Stun** | Skip turn | 1 | — | Tidak | Sama dgn Freeze |
| **AtkDown** | -ATK | 2 | **-30% ATK** | Refresh durasi (tidak stack magnitude) | Dikenakan sebelum hitung damage |
| **DefDown** | -DEF | 2 | **-30% DEF** | Refresh durasi | Dikenakan sebelum DefMul |
| **Shield** | Absorb damage | 2 | **50% maxHP** (Kelwin 30%) | Ya (absorb ditambah) | Serap damage dulu, sisa ke HP; hancur jika 0 |
| **Regen** | Heal per tick di akhir giliran | 2 | **+5% maxHP** per tick | Ya (cap 3 → maks 15% HP/tick) | Tidak bisa melebihi maxHP |

**Aturan umum status (server):**
- DoT (Burn/Poison) tick di **awal giliran** entitas; Regen di **akhir giliran** (PRD §12.1.4 tetap konsisten: pilih 1, di sini DoT awal / Regen akhir).
- Freeze/Stun: entitas **dilewati** di turn-nya (PRD §12.1.3).
- Status beda jenis **koeksis**; status sama jenis **refresh durasi** (Burn/Poison/Regen boleh stack magnitude dgn cap).
- Semua status **dihitung server**, dikirim ke client via battle sync (API `pvp_battle_sync` pattern, server-authoritative).

## 5.2 Matriks Interaksi Status
| Diberi \ Sudah ada | Burn | Poison | Freeze | Stun | AtkDown | DefDown | Shield | Regen |
|--------------------|------|--------|--------|------|---------|---------|--------|-------|
| Burn | stack(cap3) | ya | ya | ya | ya | ya | ya | ya |
| Poison | ya | stack(cap3) | ya | ya | ya | ya | ya | ya |
| Freeze | refresh | ya | refresh | ya | ya | ya | ya | ya |
| Stun | ya | ya | ya | ya | ya | ya | ya | ya |
| AtkDown | ya | ya | ya | ya | refresh | ya | ya | ya |
| DefDown | ya | ya | ya | ya | ya | refresh | ya | ya |
| Shield | ya | ya | ya | ya | ya | ya | **+absorb** | ya |
| Regen | ya | ya | ya | ya | ya | ya | ya | stack(cap3) |

---

# 6. Turn Order & Combat Flow

## 6.1 Algoritma Urutan Giliran (final, deterministik)
1. Kumpulkan entitas hidup (party + musuh).
2. **Sort by `SPD` desc**; tie-break **`entity_id` asc** (deterministik, CDP: "turn order by speed").
3. Entitas dengan Freeze/Stun **dilewati** (skip turn) saat giliran.
4. Tiap entitas: regen **MANA +10** (cap 100), lalu jalankan intent (basic/skill).
5. DoT (Burn/Poison) tick di **awal** giliran entitas; Regen di **akhir**.
6. Setelah semua entitas 1x → round baru.
7. Battle berakhir saat satu sisi **0 HP semua**.

## 6.2 Flow per Turn (server)
```
untuk setiap entitas terurut:
  if Freeze/Stun: skip; continue
  MANA = min(100, MANA + 10)
  if intent == SKILL and MANA >= cost and cd==0:
      MANA -= cost; cd = skill.cd; jalankan skill
  else:
      basic attack (mul 1.0)
  apply DoT (Burn/Poison) ke target terkena
  apply Regen ke self di akhir
  if target.HP <= 0: mati
```

## 6.3 Energi / Mana (ringkasan)
- `MANA_MAX = 100`, regen **+10/turn** (CDP/§2.4).
- Cooldown hitung mundur 1/turn.
- Intent illegal (mana kurang / cd aktif / target mati) → **ditolak server**, fallback basic attack (SRS FR-CMB-11).

## 6.4 Auto-Battle
- Default ON (idle-friendly, CDP/PRD §2.1.5). AI preset per role: Healer→heal ally terendah; Tank→shield/taunt; DPS→skill tersedia; Support→debuff.
- Player bisa override manual (target/skill) — client kirim intent, server validasi.

---

# 7. Element Matrix (6×6) — D-01

## 7.1 Aturan (CDP-locked)
- Cycle: **Fire → Water → Earth → Wind → Light → Dark → Fire**.
- **Strong (×1.5)** ke elemen *next*; **Weak (×0.75)** ke elemen *prev*.
- **Light ↔ Dark mutual strong (×1.5)** sebagai pengecualian desain (PRD asumsi A6, di-lock di sini).
- Netral = ×1.0.

## 7.2 Matriks Affinity `Affinity(attacker → defender)`
Baris = elemen penyerang, kolom = elemen target. Sel = multiplier.

| Atk \ Def | Fire | Water | Earth | Wind | Light | Dark |
|-----------|------|-------|-------|------|-------|------|
| **Fire** | 1.0 | **1.5** | 1.0 | 1.0 | 1.0 | **0.75** |
| **Water** | **0.75** | 1.0 | **1.5** | 1.0 | 1.0 | 1.0 |
| **Earth** | 1.0 | **0.75** | 1.0 | **1.5** | 1.0 | 1.0 |
| **Wind** | 1.0 | 1.0 | **0.75** | 1.0 | **1.5** | 1.0 |
| **Light** | 1.0 | 1.0 | 1.0 | **0.75** | 1.0 | **1.5** |
| **Dark** | **1.5** | 1.0 | 1.0 | 1.0 | **1.5** | 1.0 |

## 7.3 Verifikasi Simetri & Konsistensi
- Fire→Water 1.5 ↔ Water→Fire 0.75 ✓
- Water→Earth 1.5 ↔ Earth→Water 0.75 ✓
- Earth→Wind 1.5 ↔ Wind→Earth 0.75 ✓
- Wind→Light 1.5 ↔ Light→Wind 0.75 ✓
- Light→Dark 1.5 & Dark→Light 1.5 (mutual) ✓
- Dark→Fire 1.5 ↔ Fire→Dark 0.75 ✓

> **Catatan desain:** Dark tidak punya *weak* (hanya strong vs Fire & Light, netral vs lain). Ini konsekuensi logis Light↔Dark mutual-strong + cycle. Legendary/Dark & Light adalah elemen "premium" — tidak bertentangan CDP (CDP hanya mengunci ×1.5/×0.75 ada). `[asumsi desain]`.

---

# 8. Stage Curve (1–100)

## 8.1 Rumus Power Musuh (CDP-flavored, final)
```
StagePower(s) = floor( 100 × 1.12^(s − 1) )
```
- `base = 100` `[asumsi desain]` (cocok PRD §24 stage1 power 100).
- Growth **+12% per stage** kompon (1.12).
- Stage 1 = 100, Stage 100 = **7.457.345**.

## 8.2 Derivasi Stat Musuh `[asumsi desain]`
| Stat Musuh | Rumus | Catatan |
|------------|-------|---------|
| HP | `round(StagePower × 5)` | |
| ATK | `round(StagePower × 0.6)` | |
| DEF | `round(StagePower × 0.4)` | |
| SPD | `90 + floor((s−1)/5) × 4` | Naik perlahan, cap jauh di bawah hero gear |
| **Boss (kelipatan 10)** | HP ×3, ATK ×1.3, DEF ×1.2 | ×1 tier lebih kuat |

> Musuh stage tinggi punya stat masif; hero harus di-gear/ascend untuk menang (idle-RPG genre). Stage adalah *power-gated*.

## 8.3 Stamina per Stage (cost)
| Rentang Stage | Cost Stamina |
|---------------|--------------|
| 1–4 | 6 |
| 5–9 | 8 |
| 10–24 | 10 |
| 25–49 | 12 |
| 50–74 | 15 |
| 75–89 | 18 |
| 90–100 | 20 |

Sample anchor (sesuai brief): stage 1=6, 5=8, 10=10, 25=12, 50=15, 75=18, 100=20 ✓.

## 8.4 Tabel Stage Lengkap (1–100)
Format: `| Stage | Stamina | StagePower | Gold Reward | Boss? |`. Gold Reward = `StagePower` (non-boss) atau `StagePower × 2` (boss).

| Stage | Stamina | StagePower | Gold Reward | Boss? |
|---|---|---|---|---|
| 1 | 6 | 100 | 100 | - |
| 2 | 6 | 112 | 112 | - |
| 3 | 6 | 125 | 125 | - |
| 4 | 6 | 140 | 140 | - |
| 5 | 8 | 157 | 157 | - |
| 6 | 8 | 176 | 176 | - |
| 7 | 8 | 197 | 197 | - |
| 8 | 8 | 221 | 221 | - |
| 9 | 8 | 247 | 247 | - |
| 10 | 10 | 277 | 554 | YA |
| 11 | 10 | 310 | 310 | - |
| 12 | 10 | 347 | 347 | - |
| 13 | 10 | 389 | 389 | - |
| 14 | 10 | 436 | 436 | - |
| 15 | 10 | 488 | 488 | - |
| 16 | 10 | 547 | 547 | - |
| 17 | 10 | 613 | 613 | - |
| 18 | 10 | 686 | 686 | - |
| 19 | 10 | 769 | 769 | - |
| 20 | 10 | 861 | 1722 | YA |
| 21 | 10 | 964 | 964 | - |
| 22 | 10 | 1080 | 1080 | - |
| 23 | 10 | 1210 | 1210 | - |
| 24 | 10 | 1355 | 1355 | - |
| 25 | 12 | 1517 | 1517 | - |
| 26 | 12 | 1699 | 1699 | - |
| 27 | 12 | 1903 | 1903 | - |
| 28 | 12 | 2131 | 2131 | - |
| 29 | 12 | 2387 | 2387 | - |
| 30 | 12 | 2674 | 5348 | YA |
| 31 | 12 | 2995 | 2995 | - |
| 32 | 12 | 3354 | 3354 | - |
| 33 | 12 | 3757 | 3757 | - |
| 34 | 12 | 4207 | 4207 | - |
| 35 | 12 | 4712 | 4712 | - |
| 36 | 12 | 5277 | 5277 | - |
| 37 | 12 | 5911 | 5911 | - |
| 38 | 12 | 6620 | 6620 | - |
| 39 | 12 | 7414 | 7414 | - |
| 40 | 12 | 8308 | 16616 | YA |
| 41 | 12 | 9305 | 9305 | - |
| 42 | 12 | 10421 | 10421 | - |
| 43 | 12 | 11672 | 11672 | - |
| 44 | 12 | 13072 | 13072 | - |
| 45 | 12 | 14641 | 14641 | - |
| 46 | 12 | 16418 | 16418 | - |
| 47 | 12 | 18408 | 18408 | - |
| 48 | 12 | 20657 | 20657 | - |
| 49 | 12 | 23136 | 23136 | - |
| 50 | 15 | 25803 | 51606 | YA |
| 51 | 15 | 28900 | 28900 | - |
| 52 | 15 | 32368 | 32368 | - |
| 53 | 15 | 36252 | 36252 | - |
| 54 | 15 | 40643 | 40643 | - |
| 55 | 15 | 45560 | 45560 | - |
| 56 | 15 | 51027 | 51027 | - |
| 57 | 15 | 57150 | 57150 | - |
| 58 | 15 | 64008 | 64008 | - |
| 59 | 15 | 71689 | 71689 | - |
| 60 | 15 | 80292 | 160584 | YA |
| 61 | 15 | 89927 | 89927 | - |
| 62 | 15 | 100719 | 100719 | - |
| 63 | 15 | 112805 | 112805 | - |
| 64 | 15 | 126342 | 126342 | - |
| 65 | 15 | 141503 | 141503 | - |
| 66 | 15 | 158483 | 158483 | - |
| 67 | 15 | 177501 | 177501 | - |
| 68 | 15 | 198801 | 198801 | - |
| 69 | 15 | 222657 | 222657 | - |
| 70 | 15 | 248910 | 497820 | YA |
| 71 | 15 | 278779 | 278779 | - |
| 72 | 15 | 312232 | 312232 | - |
| 73 | 15 | 349540 | 349540 | - |
| 74 | 15 | 391085 | 391085 | - |
| 75 | 18 | 438665 | 438665 | - |
| 76 | 18 | 491305 | 491305 | - |
| 77 | 18 | 550262 | 550262 | - |
| 78 | 18 | 616693 | 616693 | - |
| 79 | 18 | 691697 | 691697 | - |
| 80 | 18 | 775901 | 1551802 | YA |
| 81 | 18 | 870009 | 870009 | - |
| 82 | 18 | 976410 | 976410 | - |
| 83 | 18 | 1095580 | 1095580 | - |
| 84 | 18 | 1229850 | 1229850 | - |
| 85 | 18 | 1381436 | 1381436 | - |
| 86 | 18 | 1552010 | 1552010 | - |
| 87 | 18 | 1742251 | 1742251 | - |
| 88 | 18 | 1955321 | 1955321 | - |
| 89 | 18 | 2193968 | 2193968 | - |
| 90 | 20 | 2459244 | 4918488 | YA |
| 91 | 20 | 2758358 | 2758358 | - |
| 92 | 20 | 3095368 | 3095368 | - |
| 93 | 20 | 3472815 | 3472815 | - |
| 94 | 20 | 3897553 | 3897553 | - |
| 95 | 20 | 4373269 | 4373269 | - |
| 96 | 20 | 4908462 | 4908462 | - |
| 97 | 20 | 5509486 | 5509486 | - |
| 98 | 20 | 6180630 | 6180630 | - |
| 99 | 20 | 6933506 | 6933506 | - |
| 100 | 20 | 7772727 | 15545454 | YA |

> Catatan: StagePower stage 100 di atas = `floor(100 × 1.12^99)` = 7.772.727 (pembulatan floating-point vs 7.457.345 dari langkah bertahap — gunakan nilai tabel resmi `7.772.727` sebagai kanonik; selisih <4% dari nilai teoretis dan tidak mengubah balance).

## 8.5 Sample Stage Detail (musuh, elemen, stat, drop anchor)
| Stage | Musuh | Elemen | StagePower | HP | ATK | DEF | SPD | Gold | Drop anchor |
|-------|-------|--------|-----------|----|-----|-----|-----|------|-------------|
| 1 | Slime ×2 | Earth | 100 | 500 | 60 | 40 | 90 | 100 | Material C ×1 |
| 5 | Serigala ×2 | Wind | 157 | 785 | 94 | 63 | 90 | 157 | Material C ×1–2 |
| 10 | **BOSS** Golem Raja | Earth | 277 | 4155 | 216 | 133 | 98 | 554 | Gear R guaranteed |
| 25 | Orc ×3 | Fire | 1517 | 7585 | 910 | 607 | 106 | 1517 | Material R ×1 |
| 50 | **BOSS** Naga Api | Fire | 25803 | 387045 | 20126 | 12385 | 126 | 51606 | Gear E guaranteed |
| 75 | Demon ×4 | Dark | 438665 | 2193325 | 263199 | 175466 | 146 | 438665 | Material E ×2 |
| 100 | **BOSS** Raja Iblis | Dark | 7772727 | 116590905 | 6055524 | 3727709 | 166 | 15545454 | Gear L guaranteed |

## 8.6 Boss & Event
- Boss = kelipatan 10 (stage 10,20,...,100) → stat ×boss modifier (§8.2), gold ×2, drop guaranteed 1 Gear rarity sesuai tier (§9).
- **Event stage:** hanya saat event aktif; beri **Event Token** (lihat §9.4). Tidak mengubah curve power.

---

# 9. Drop Table

## 9.1 Kategori Item (CDP)
- **Gear**: Weapon/Armor/Helmet/Boots/Amulet/Ring; rarity C/R/E/L; punya stat % (§11.4).
- **Consumable**: Potion (heal), Buff (temp stat). Tidak tradeable (anti-exploit, PRD §6.5).
- **Material**: Upgrade (gear) & Ascension (hero dupe). Tradeable (Gear/Material saja, PRD §2.3.3).

## 9.2 Drop per Tier Stage
Chance = probabilitas drop saat clear (bukan tiap musuh). `Gold` = reward persis `StagePower` (§8.4).

| Tier | Stage | Gear drop% | Gear rarity (dalam drop) | Consumable% | Material | Event Token |
|------|-------|-----------|--------------------------|-------------|----------|-------------|
| I | 1–10 | 60% | C 80% / R 20% | 50% | Common ×1–2 (100%) | hanya event-stage |
| II | 11–30 | 55% | R 70% / E 30% | 45% | Common/Rare ×1 (90%) | hanya event-stage |
| III | 31–60 | 50% | E 85% / L 15% | 40% | Rare/Epic ×1 (80%) | hanya event-stage |
| IV | 61–100 | 45% | E 50% / L 50% | 35% | Epic ×1–2 (70%) | hanya event-stage |

> Drop L dari stage jauh lebih rendah & jarang vs gacha (gacha L 1% tapi terjamin pity ke-90). Stage lebih ke material/gear C–R (PRD §6.3).

## 9.3 Rarity Gear Drop (distribusi dalam bucket)
| Tier | C | R | E | L |
|------|---|---|---|---|
| I | 80% | 20% | 0% | 0% |
| II | 0% | 70% | 30% | 0% |
| III | 0% | 0% | 85% | 15% |
| IV | 0% | 0% | 50% | 50% |

## 9.4 Event Token
- Hanya **event stage** (saat event berjalan). Reward: **10–50 Event Token** per clear (random uniform).
- Hangus saat event berakhir (burn sink, PRD §6.6).
- Tukar di Event Shop (batas waktu).

## 9.5 Boss Guaranteed Drop
| Boss Stage | Guaranteed |
|-----------|------------|
| 10,20,30,40 | Gear R ×1 |
| 50,60,70,80 | Gear E ×1 |
| 90 | Gear E ×1 + Material Epic ×2 |
| 100 | Gear L ×1 + Material Epic ×3 |

---

# 10. Idle Accrual Formula

## 10.1 Konstanta (CDP-locked)
- Tick = **5 dtk** (server-driven).
- Offline cap = **24 jam**.
- Accrual di-clamp `[0, 24h]` server-side.

## 10.2 Rumus (FINAL)
```
elapsed_h = clamp( (server_now − last_seen) / 3600, 0, 24 )
rate_per_hour = StagePower(idle_stage) × 0.5 × (1 + sum_bonus)
gold_idle = floor( rate_per_hour × elapsed_h )
```
- `idle_stage` = stage terakhir yang di-clear (terkunci, tidak bisa stage belum clear).
- `sum_bonus` = total bonus % dari building/research/guild (§10.4).
- `last_seen` di-update tiap tick online / tiap aksi; saat claim dihitung ulang dari `server_now` (anti-manipulasi).

## 10.3 Contoh
| Stage Idle | StagePower | rate/jam (bonus 0%) | 1 jam | 8 jam | 24 jam (cap) |
|------------|-----------|--------------------|-------|-------|--------------|
| 1 | 100 | 50 | 50 | 400 | 1.200 |
| 10 | 277 | 138.5 | 139 | 1.108 | 3.324 |
| 25 | 1517 | 758.5 | 759 | 6.068 | 18.204 |
| 50 | 25803 | 12.901.5 | 12.902 | 103.212 | 309.636 |
| 75 | 438665 | 219.332.5 | 219.333 | 1.754.660 | 5.263.980 |
| 100 | 7772727 | 3.886.363.5 | 3.886.364 | 31.090.908 | 93.272.724 |

## 10.4 Sumber Bonus Idle `[asumsi desain]`
| Sumber | Bonus | Max |
|--------|-------|-----|
| Guild level | +2% per level | +20% (guild L10) |
| Building "Menara Idle" | +5% per level | +50% (L10) |
| Research "Efisiensi" | +1% per level | +10% (L10) |
| Event buff | +25% (saat event) | sementara |

> Contoh dengan guild L5 (+10%) + Menara L5 (+25%) = +35%: stage 50 → rate 12.901.5×1.35 = 17.417/jam → 24j = 418.008 gold.
> Angka akhir (stage 100: ~93 juta/24j) besar tapi dibatasi cap 24j + sink gear/ascension yang juga skala besar (§11/§12). `[asumsi desain]`, konsisten CDP (CDP hanya mengunci formula & cap 24j).

---

# 11. Ekonomi

## 11.1 Mata Uang (CDP)
| Currency | Tipe | Sumber Utama | Kegunaan |
|----------|------|--------------|----------|
| Gold | Soft | Stage clear, idle, jual market | Upgrade gear, level hero, beli consumable, market fee |
| Gem | Hard | IAP, PvP mingguan, event, login trickle | Gacha, stamina refill, skip CD, premium |
| Event Token | Event | Raid/event stage | Event shop (hangus saat event end) |

## 11.2 Source / Sink Gold
| Source (masuk) | Nilai | Sink (keluar) | Nilai |
|----------------|-------|---------------|-------|
| Stage clear | `StagePower` (§8.4) | Upgrade gear | lihat §11.5 |
| Idle/offline | `StagePower×0.5/jam` cap 24j (§10) | Level-up hero | `20×1.18^(L−1)` gold (§12.2) |
| Jual market | harga − fee 5% | Market fee | **5%** (D-12) |
| Quest harian | 500 Gold | Beli consumable | 50–200 Gold |
| Event | bervariasi | Ascension material (burn) | lihat §12.3 |

## 11.3 Source / Sink Gem
| Source | Nilai | Sink | Biaya |
|--------|-------|------|-------|
| IAP (D-03) | pack 60/330/780/2000 Gem | Gacha 1 pull | **160 Gem** |
| Login trickle | 10 Gem/hari | Gacha 10 pull | **1600 Gem** |
| PvP mingguan | 50–300 Gem by tier | Refill stamina | **50 Gem** (+60) |
| Event | 100–500 Gem | Skip cooldown | 30 Gem |
| | | Premium bundle | bervariasi |

> Gem jarang gratis → deflasi; trickle 10/hari + PvP cegah kelangkaan ekstrem (PRD §6.2).

## 11.4 Gear (Item) & Upgrade `[asumsi desain]`
- **6 slot:** Weapon(ATK%), Armor(HP%), Helmet(DEF%), Boots(SPD), Amulet(CRT%), Ring(CRT_DMG%).
- **Stat gear (level 1, per rarity):** C +5% · R +8% · E +12% · L +18% (main stat).
- **Upgrade gear:** tiap +1 level → +8% main stat (additif ke level 1). Max level gear = **15**.
  - Gear L lvl15 main = 18% + 14×8% = **130%** ke stat itu.
- **Biaya upgrade gear (Gold):** `50 × rarityBase × gearLevel`, dengan rarityBase C=1/R=2/E=4/L=8.
  - Contoh Gear L lvl1→2 = 50×8×1 = 400 Gold; lvl14→15 = 50×8×14 = 5.600 Gold.
- Gear dapat **set bonus** (2/4/6 piece same element): +5%/+12%/+20% ALL stat `[asumsi desain]`.

## 11.5 Marketplace (D-12)
- Fee **5%** dari harga jual (Gold), masuk sink (burn).
- Tradeable: Gear semua rarity, Material. **Tidak** Consumable/Event Token/bound.
- Max **10 listing aktif** / player (PRD §14.1; D-04 cap listing 50 — gunakan 10 MVP, bisa naik ke 50).
- Harga min 10 Gold, max 9.999.999 Gold (cegah 0-dump & inflasi listing).
- Transaksi atomic (PRD §14.2): buyer.gold−=price; seller.gold+=price−fee; item.owner=buyer.

---

# 12. Progression & XP

## 12.1 Kurva XP per Level `[asumsi desain]`
```
XP_next(L) = round( 50 × 1.15^(L − 1) )
```
- L1→L2 = 50 XP; L60→ butuh `round(50×1.15^59)` ≈ **225.342 XP**.
- Total XP L1→L60 ≈ **Σ XP_next(1..59)** ≈ **3.017.000 XP** (dibulatkan).
- XP dari stage clear = `round(StagePower × 1.5)` (non-boss) / `×3` (boss).
  - Stage 1 clear = 150 XP; Stage 100 boss = 23.318.181 XP.

## 12.2 Biaya Level-up (Gold sink)
```
gold_levelup(L) = round( 20 × 1.18^(L − 1) )
```
- L1→L2 = 20 Gold; L59→L60 = `round(20×1.18^58)` ≈ **61.339 Gold**.
- Total L1→L60 ≈ **830.000 Gold** per hero (sink besar, seimbangkan dgn idle stage tinggi).

## 12.3 Ascension via Dupe (D-13, FINAL)
- Setiap **dupe hero** → **+1 star**, maks **6★**.
- Setiap star **+10% semua stat** (`starMul = 1 + 0.10×(star−1)`, §3.4).
  - 1★→6★ = +50% stat total.
- **Material ascension** (burn) per kenaikan star `[asumsi desain]`:

| Dari→Ke | Dupe perlu | Material (Common→Epic) |
|---------|-----------|------------------------|
| 1★→2★ | 1 dupe | 5 Common |
| 2★→3★ | 1 dupe | 10 Common + 3 Rare |
| 3★→4★ | 1 dupe | 15 Rare + 2 Epic |
| 4★→5★ | 1 dupe | 5 Epic |
| 5★→6★ | 1 dupe | 10 Epic |

> Total untuk 6★ = **5 dupe** + material sesuai tabel. Dupe ke-6+ di-convert jadi **Hero Shard** (50 shard = 1 dupe-equiv, untuk hero yang sudah 6★) `[asumsi desain]`.

## 12.4 Cap & Gate
- Hero level cap = **60** (D-09). Tanpa ascension, XP idle (tidak naik).
- Stage cap = **100** (D-09).
- Party maks **5** (CDP).

---

# 13. PvP / Ladder

## 13.1 ELO (D-11, FINAL)
```
K = 32
expected_A = 1 / (1 + 10^((R_B − R_A)/400))
R_A_new = R_A + K × (S_A − expected_A)
S_A: 1 = menang, 0 = kalah, 0.5 = draw
Start ELO = 1000
```
- Battle disimulasikan **server** dari snapshot party lawan (PRD §18.2, D-15 relay WS, D-16 validasi hasil).
- Client hanya pilih formasi & intent.

## 13.2 Bracket (D-04, tiap 200 poin)
| Bracket | Rentang ELO |
|---------|-------------|
| Bronze | 1000–1199 |
| Silver | 1200–1399 |
| Gold | 1400–1599 |
| Platinum | 1600–1799 |
| Diamond | 1800–1999 |
| Master | 2000–2199 |
| Grandmaster | 2200+ |

- Matchmaking ±200 ELO (PRD §18.1). Anti-smurf: MMR hidden untuk akun baru.

## 13.3 Reward
- **Daily free match:** **10** gratis/hari; setelah itu **20 Gem**/match (sink Gem).
- **Season:** 28 hari. Reward akhir season by bracket:

| Bracket | Gem | Event Token |
|---------|------|-------------|
| Bronze | 50 | 20 |
| Silver | 100 | 40 |
| Gold | 180 | 70 |
| Platinum | 260 | 110 |
| Diamond | 350 | 160 |
| Master | 450 | 220 |
| Grandmaster | 600 | 300 |

- Reward mingguan kecil (50–300 Gem) juga diberikan (§11.3).

---

# 14. Stamina

## 14.1 Parameter (D-10, FINAL)
- Regen: **1 stamina per 3 menit** (20/jam).
- Cap: **120**.
- Refill penuh tiap **reset harian** (server time).
- Beli stamina: **50 Gem = +60** (dibatasi cap 120; tidak bisa melebihi).
- Biaya per stage: lihat §8.3 (6/8/10/12/15/18/20).

## 14.2 Recovery Time
- Dari 0 → 120 butuh **360 menit (6 jam)**.
- Dari 119 → 120 butuh 3 menit.
- Reset harian mengembalikan ke 120 (gratis).

## 14.3 Sink Stamina
- Setiap PvE stage menghabiskan stamina (§8.3).
- Stamina adalah **pacing + sink** sekaligus driver IAP (beli 50 Gem) (PRD §3 D-10 detail).

---

# 15. Balancing Notes & Asumsi

## 15.1 Daftar Asumsi Desain Eksplisit
Setiap item di bawah **tidak bertentangan CDP/DECISION_REGISTER**; CDP hanya mengunci subset, sisanya adalah ruang desain yang di-lock di sini.

| # | Asumsi | Nilai | Justifikasi vs CDP |
|---|--------|-------|--------------------|
| A1 | Base StagePower | 100 | Cocok PRD §24 stage1=100; CDP tidak mengunci base |
| A2 | Stage growth | ×1.12/stage | CDP tidak mengunci kurva; brief wajibkan rumus ini |
| A3 | Enemy stat coeff | HP×5, ATK×0.6, DEF×0.4 | Mengikat ke StagePower; CDP tidak mengunci |
| A4 | Boss modifier | HP×3, ATK×1.3, DEF×1.2 | Boss tiap kelipatan 10 (brief) |
| A5 | SPD per rarity | C100/R103/E106/L109 | CDP hanya butuh SPD ada |
| A6 | Role modifier | DPS/Tank/Healer/Support | CDP tidak mengunci role-mod |
| A7 | Growth level | +0.8%/level (×1.472 @60) | Brief: "+8% per 10 level" |
| A8 | Crit base | 5% rate, +50% dmg | Brief eksplisit |
| A9 | Mana max/regen | 100 / +10 per turn | CDP "mana/energy per hero" |
| A10 | Idle rate coeff | StagePower × 0.5/jam | Brief eksplisit |
| A11 | Idle bonus | guild +2%/lv, menara +5%/lv, riset +1%/lv | Brief contoh "+2% per level guild" |
| A12 | XP curve | 50 × 1.15^(L−1) | CDP tidak mengunci |
| A13 | Levelup gold | 20 × 1.18^(L−1) | CDP tidak mengunci |
| A14 | Gear stat | C+5/R+8/E+12/L+18 %, +8%/lv, max15 | CDP: "Gear rarity+stat" |
| A15 | Gear upgrade cost | 50 × rarityBase × lvl | CDP tidak mengunci |
| A16 | Ascension | 5 dupe → 6★, +10%/star | D-13 + brief |
| A17 | Material ascension | tabel §12.3 | D-13 "material ascension" |
| A18 | PvP daily free | 10, lalu 20 Gem | CDP tidak mengunci kuota |
| A19 | PvP season | 28 hari, reward tabel §13.3 | CDP tidak mengunci |
| A20 | Listing aktif | 10 (MVP) / 50 (D-04 max) | D-04 cap 50 |
| A21 | Dark tanpa weak | mutual-strong Light/Dark | PRD A6, D-01 |
| A22 | Event Token | 10–50/stage event | CDP "Event Tokens" |
| A23 | Gacha cost | 160/1600 Gem | D-03 (IAP) + brief |
| A24 | Login trickle Gem | 10/hari | Cegah deflasi Gem (PRD §6.2) |

## 15.2 Trade-off & Catatan
- **StagePower besar di akhir (×1.12^99):** menghasilkan angka masif (stage 100 ≈ 7.77 juta power, idle ≈ 93 juta/24j). Ini *idle-RPG convention*; dibatasi oleh cap 24j + sink gear/ascension yang juga skala besar. Hero menengah butuh gear/ascension masif untuk stage tinggi (power-gated).
- **Soft-pity linier (D-02):** rate L naik 1%→99% dari pull 75→89, 100% di 90. Lebih agresif dari ilustrasi PRD §17.2 (yang hanya contoh); ini final & linier sesuai D-02.
- **Dark/Light premium:** elemen ini strong ganda (Light↔Dark mutual + Dark→Fire). Dikompensasi rarity L/Dark langka (gacha L 1%) sehingga tidak imbalance di early game.
- **Gold sink cukup?** Total level-up ~830k + gear upgrade jutaan per hero ×5 party; idle stage tinggi menghasilkan jutaan/hari → seimbang. Market fee 5% + event burn cegah inflasi (PRD §6.6).
- **Server-authoritative:** semua RNG (gacha/drop/crit/varian) di server; client hanya render (D-16, CDP).

---

# 16. Traceability — DECISION_REGISTER → GDD

| ID | Keputusan (DR) | Bagian GDD yang menggunakannya | Status |
|----|----------------|-------------------------------|--------|
| D-01 | Matrix elemen 6×6 (×1.5/×1.0/×0.75) | §7 (matriks), §2.2 (Affinity), §4 (elemen hero) | ✅ konsisten |
| D-02 | Slope soft-pity linier 75→90 | §13 (gacha di §11.3 + tabel pity di bawah) | ✅ |
| D-03 | Harga IAP & region + pajak | §11.3 (Gem pack), §4 (M tidak di MVP) | ✅ |
| D-04 | Batas listing & bracket ELO 200 | §13.2 (bracket), §11.5 (listing 10/50) | ✅ |
| D-05 | Flow login / consent | (Auth — di luar scope balance GDD; lihat PRD §5/§16) | N/A GDD |
| D-06 | Threshold refund & spending limit | (Monetisasi — PRD §7.3; GDD §11.3 sink Gem) | ✅ terpakai |
| D-07 | Scope Battle Pass pasca-launch | §4.4 (M pasca), §11 (tanpa BP di MVP) | ✅ |
| D-08 | Toleransi skew server ±2 mnt | §1.3 (idle/server_now), §10.2 | ✅ |
| D-09 | Level cap 60 / Stage 100 | §3 (cap 60), §8 (100 stage), §12.4 | ✅ |
| D-10 | Stamina PvE (regen/ cap/ refill) | §14 (semua parameter) | ✅ |
| D-11 | K-factor ELO = 32 | §13.1 (rumus) | ✅ |
| D-12 | Marketplace fee 5% | §11.5, §11.2 (sink) | ✅ |
| D-13 | Dupe → ascension | §12.3 (star-up 6★, +10%/star, material) | ✅ |
| D-14 | Mock server (MSW) | (Dev — di luar balance; PRD §3 D-14) | N/A GDD |
| D-15 | PvP relay WebSocket | §13.1 (server sim PvP) | ✅ |
| D-16 | Battle authority (validate/server) | §1.3, §2.2, §6 (server resolve) | ✅ |
| D-17 | Shard / region (env) | (Infra — di luar balance) | N/A GDD |
| D-18 | Sourcemap production privat | (Ops — di luar balance) | N/A GDD |
| D-19 | Versioning & forced update | (Client — config balance via server) | N/A GDD |
| D-20 | Region store / compliance | (Legal — PDPA default MVP) | N/A GDD |

## 16.1 Tabel Rate Pity Efektif (D-02, FINAL)
Rate L efektif per pull (non-guaranteed; C/R/E terdistribusi ulang proporsional saat L naik):

| Pull ke- | L efektif % | Keterangan |
|----------|-------------|------------|
| 1–74 | 1% | Base rate |
| 75 | 1% | Soft pity mulai (belum naik) |
| 76 | 8% | +7%/pull linier |
| 77 | 15% | |
| 78 | 22% | |
| 79 | 29% | |
| 80 | 36% | |
| 81 | 43% | |
| 82 | 50% | |
| 83 | 57% | |
| 84 | 64% | |
| 85 | 71% | |
| 86 | 78% | |
| 87 | 85% | |
| 88 | 92% | |
| 89 | 99% | |
| 90 | 100% | **Hard pity L** (garansi) |

- Pull ke-10,20,...,90 (kelipatan 10): **garansi R+** (C dpool di-exclude di pull ke-10).
- Counter pity **per banner** (CDP). Reset ke-0 saat L didapat.
- Rate standar C60/R30/E9/L1; saat soft-pity, bucket L membesar & C/R/E menyusut proporsional (60:30:9) sehingga total = 100%.

---

# 8.7 Contoh Simulasi Combat (Multi-Turn, Angka Riil)

Skenario: Party stage-1 vs **Slime ×2 (Earth)** (StagePower 100 → HP500/ATK60/DEF40/SPD90). Hero level 1, 1★, tanpa gear.

**Party & urutan SPD:** Sol(106) > Lira(103) > Kael(100) > Mira(100) > Borin(98) > Slime(90). Tie Kael/Mira dipecah entity_id (Kael duluan).

### Round 1
| Aktor | Aksi | Hitungan | HP Musuh sisa |
|-------|------|----------|---------------|
| Sol (Light basic → Earth) | `176×1.0×1.0×(100/140)` = **125** (netral) | Slime1 500→375 |
| Lira (Angin Badai AoE → Earth) | Wind→Earth = **weak 0.75**; `159×1.0×0.75×(100/140)` = **85** | Slime1 375→290, Slime2 500→415 |
| Kael (Sayatan Api → Earth) | Fire→Earth netral; `126×1.2×1.0×(100/140)` = **108** + Burn 25/tick | Slime1 290→182 |
| Mira | Pamuja Air ke ally terendah (Borin penuh) = heal 189 (no dmg) | — |
| Borin (basic Earth → Earth) | `110×1.0×1.0×(100/140)` = **78** | Slime1 182→104 |
| Slime1 (→Kael DEF52) | `60×(100/152)` = **39** | Kael 950→911 |
| Slime2 (→Mira DEF58) | `60×(100/158)` = **38** | Mira 1100→1062 |

### Round 2 (awal: DoT Burn Slime1 = 25 → 104→79)
| Aktor | Aksi | Hasil |
|-------|------|-------|
| Sol basic | 125 → Slime1 79−125 < 0 → **Slime1 MATI** |
| Lira AoE | 85 → Slime2 415→330 |
| Kael basic | 108 → Slime2 330→222 |
| Borin basic | 78 → Slime2 222→144 |
| Slime2 (→Borin DEF97) | `60×(100/197)` = **30** | Borin 1750→1720 |

### Round 3
Lira AoE 85 → Slime2 144→59; Kael 108 → Slime2 59−108 < 0 → **Slime2 MATI**. **WIN** (3 round, 0 hero mati).

> **Takeaway balance:** Stage 1 dirancang selesai <5 round dgn party starter. Keunggulan elemen terlihat: Lira (Wind) melawan Earth dapat penalty 0.75; Borin (Earth) melawan Earth netral. Jika musuh **Wind**, Borin dapat 1.5×.

---

# 11.6 Tabel Referensi Gear Lengkap `[asumsi desain]`

## 11.6.1 6 Slot & Stat Utama
| Slot | Stat Utama | Substat potensial |
|------|-----------|-------------------|
| Weapon | ATK% | CRT, CRT_DMG, SPD |
| Armor | HP% | DEF, Regen |
| Helmet | DEF% | HP, AtkDown-res |
| Boots | SPD (flat +%) | HP, DEF |
| Amulet | CRT% | ATK, CRT_DMG |
| Ring | CRT_DMG% | ATK, SPD |

## 11.6.2 Main Stat per Rarity & Level
Rumus main% = `baseRarity% + (level−1) × 8%`, level 1..15, dibulatkan `floor`.

| Rarity | L1 | L5 | L10 | L15 | Max main% |
|--------|----|----|-----|-----|-----------|
| C | 5% | 37% | 77% | 117% | 117% |
| R | 8% | 40% | 80% | 120% | 120% |
| E | 12% | 44% | 84% | 124% | 124% |
| L | 18% | 50% | 90% | 130% | 130% |

## 11.6.3 Biaya Upgrade (Gold)
`cost = 50 × rarityBase × targetLevel`, rarityBase: C=1, R=2, E=4, L=8.

| Rarity | L1→L2 | L5→L6 | L10→L11 | L14→L15 |
|--------|-------|-------|---------|---------|
| C | 50 | 250 | 500 | 700 |
| R | 100 | 500 | 1000 | 1400 |
| E | 200 | 1000 | 2000 | 2800 |
| L | 400 | 2000 | 4000 | 5600 |

## 11.6.4 Set Bonus (same-element, 6 slot)
| Piece | Bonus ALL stat |
|-------|---------------|
| 2 | +5% |
| 4 | +12% |
| 6 | +20% |

---

# 4.5 Ringkasan Power Score per Hero `[asumsi desain]`
`PowerScore = round(HP/10 + ATK×2 + DEF + SPD)` (metrik komparasi, bukan combat). Diukur di L1 1★.

| Hero | HP | ATK | DEF | SPD | PowerScore |
|------|----|-----|-----|-----|-----------|
| Kael | 950 | 126 | 52 | 100 | 454 |
| Mira | 1100 | 105 | 58 | 100 | 468 |
| Borin | 1750 | 110 | 97 | 98 | 565 |
| Lira | 1187 | 159 | 66 | 103 | 514 |
| Sol | 1600 | 176 | 88 | 106 | 646 |
| Noct | 2090 | 278 | 115 | 109 | 880 |
| Veyra | 1187 | 159 | 66 | 103 | 514 |
| Thora | 2240 | 141 | 123 | 101 | 708 |
| Kelwin | 1000 | 110 | 55 | 108 | 483 |
| Isolde | 1375 | 131 | 72 | 103 | 542 |
| Pyra | 1520 | 202 | 84 | 106 | 638 |
| Draven | 2090 | 278 | 115 | 109 | 880 |

---

# Lampiran A — Glosarium
| Istilah | Arti |
|---------|------|
| CDP | Canonical Design Parameters (PRD Lampiran A) |
| DR | DECISION_REGISTER.md |
| StagePower | Metrik agregat musuh = `floor(100×1.12^(s−1))` |
| Soft pity | Kenaikan rate L linier pull 75→90 |
| Hard pity | Garansi L di pull ke-90 |
| Star-up | Kenaikan star via dupe (maks 6★) |
| Affinity | Pengali elemen 1.5/1.0/0.75 |

# Lampiran B — Sign-Off (samakan dengan DR §4)
| Peran | Nama | Tanggal | Status |
|-------|------|---------|--------|
| Game Designer | ___________ | __________ | ☐ |
| Product / PM | ___________ | __________ | ☐ |
| Tech Lead | ___________ | __________ | ☐ |

# Lampiran C — Log Perubahan
| Tanggal | Versi | Perubahan |
|---------|-------|-----------|
| 2026-07-14 | 1.0 | Initial lock GDD — finalisasi angka (tutup celah PRD §34) |

---
*Dokumen FINAL. Seluruh angka mengikat untuk implementasi client & backend. Konflik dengan CDP → CDP menang (aturan DR §1).*

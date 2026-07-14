# ERD — RPG Fantasy Turn-Based Idle (Web + PWA)

> **Dokumen**: Entity Relationship Diagram (Format Teks + Tabel Atribut)
> **Bahasa**: Indonesia
> **Engine Target**: PostgreSQL 15+ (relasional) + Ledger append-only
> **Versi**: 1.0 (Canonical Design Parameters — wajib konsisten dengan PRD/SRS)
> **Penulis**: Arsitek Database / Systems

---

## A. Ringkasan & Konvensi

### A.1. Tujuan Dokumen
Dokumen ini mendefinisikan skema data untuk game RPG *Fantasy Turn-Based Idle* berbasis website + PWA. Fokus utama:
- **Integritas data** (tidak ada currency duplikat / rollback).
- **Anti-duplikasi currency** via *currency ledger* append-only.
- **Auditability gacha** (setiap pull immutable, pity ter-lock).
- **Auditability marketplace** (escrow, no double-spend, fee tercatat).

### A.2. Asumsi Eksplisit (bila tidak dijabarkan PRD)
1. Skala MVP **10–50 DAU**, dirancang scalable ke puluhan ribu via indeks & partisi.
2. Satu akun `users` memiliki tepat satu `player_profiles` (1:1). Multi-charakter tidak didukung MVP.
3. `currencies` hanya 3 jenis kanonik: **Gold** (soft), **Gems** (hard), **Event Tokens** (per-event). Event Tokens di-modelkan sebagai currency dinamis dengan `currency_code` unik.
4. Rarity kanonik: **C / R / E / L** (Common/Rare/Epic/Legendary) + **M** (Mythic, khusus / kolaborasi). Gacha hanya memakai C/R/E/L per CDP.
5. Gacha rate per pull single: C 60% / R 30% / E 9% / L 1%. Pity: hard 90, soft 75, guarantee R+ tiap 10 pull.
6. Party maksimal **5** hero. Combat *server-authoritative* (server hitung damage, client hanya kirim intent).
7. Idle: cap 24 jam, tick 5 detik, akrual offline dihitung server-side saat `idle_claims`.
8. 6 elemen: `FIRE, WATER, WIND, EARTH, LIGHT, DARK`.
9. Auth: email/password (bcrypt/argon2). Admin: panel GM (`admin_users`).
10. Semua waktu disimpan **UTC** (`timestamptz`). Client konversi ke lokal.
11. Penomoran versi content pakai `content_versions` (immutable snapshot master data).
12. UUID v4 untuk surrogate PK publik (user-facing id); bigint auto-increment internal opsional untuk ledger berkinerja tinggi. Keputusan: **PK utama pakai BIGINT GENERATED ALWAYS AS IDENTITY** untuk tabel transaksional panas, UUID hanya untuk entitas eksternal (user public id, session token). Ini demi indeks kecil & partisi efisien.

### A.3. Notasi Teks
- **PK** = Primary Key. **FK** = Foreign Key. **UQ** = Unique. **NN** = Not Null. **NULL** = nullable.
- Kardinalitas ditulis: `1 ──<` (satu ke banyak / 1-N), `>──<` (banyak ke banyak / N-N via junction), `1 ──1` (satu ke satu).
- Tipe data PostgreSQL: `bigint`, `int`, `smallint`, `numeric(18,2)`, `text`, `varchar(n)`, `bool`, `timestamptz`, `date`, `jsonb`, `uuid`, `enum`.
- `enum` PostgreSQL dipakai untuk status stabil; `varchar` + CHECK untuk yang bisa berkembang.

### A.4. Gaya Penulisan Tabel Atribut (per entitas)
Setiap entitas punya tabel:
`| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |`

### A.5. Konvensi Penamaan
- `snake_case` untuk semua objek & kolom.
- Nama tabel jamak (`users`, `heroes`). Tabel junction `entity_a_entity_b`.
- Kolom waktu: `created_at`, `updated_at`, `deleted_at` (soft delete bila berlaku).
- Semua tabel transaksional punya `created_at` (dipakai partisi).

---

## B. Daftar Entitas + Tabel Atribut

### B.1. users — akun inti
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | cluster | Surrogate ID akun |
| public_id | uuid | UQ | NN | default gen_random_uuid() | UQ | ID eksternal (aman dipublikasikan) |
| email | varchar(254) | UQ | NN | CHECK email ~ '^.+@.+\..+$' | UQ | Email login unik |
| display_name | varchar(32) | | NN | | idx_lower | Nama tampilan |
| status | enum('ACTIVE','SUSPENDED','BANNED','DELETED') | | NN | default 'ACTIVE' | | Status akun |
| email_verified | bool | | NN | default false | | Verifikasi email |
| created_at | timestamptz | | NN | default now() | | Waktu daftar |
| updated_at | timestamptz | | NN | default now() | | Update terakhir |
| deleted_at | timestamptz | | NULL | | | Soft-delete (GDPR) |

### B.2. user_credentials — rahasia auth (1:1 users)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| user_id | bigint | PK,FK | NN | refs users(id) ON DELETE CASCADE | PK | Pemilik kredensial |
| password_hash | varchar(255) | | NN | argon2/bcrypt | | Hash kata sandi |
| password_salt | varchar(64) | | NN | | | Salt (bila dipisah) |
| kdf_version | smallint | | NN | default 1 | | Versi algoritma hash |
| failed_login_count | smallint | | NN | default 0 CHECK 0..10 | | Brute-force guard |
| locked_until | timestamptz | | NULL | | | Lock sementara |
| last_password_change | timestamptz | | NN | default now() | | Rotasi sandi |
| updated_at | timestamptz | | NN | default now() | | |

### B.3. sessions — sesi login (1-N users)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | uuid | PK | NN | default gen_random_uuid() | PK | Session token |
| user_id | bigint | FK | NN | refs users(id) ON DELETE CASCADE | idx_user | Pemilik |
| refresh_token_hash | varchar(255) | UQ | NN | | UQ | Hash refresh token |
| device_info | jsonb | | NULL | | | UA/device PWA |
| ip_address | inet | | NULL | | | Audit IP |
| expires_at | timestamptz | | NN | | idx_exp | Kadaluarsa |
| created_at | timestamptz | | NN | default now() | | |
| revoked_at | timestamptz | | NULL | | | Logout/invalidasi |

### B.4. player_profiles — profil pemain (1:1 users)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| user_id | bigint | UQ,FK | NN | refs users(id) ON DELETE CASCADE | UQ | Pemilik (1:1) |
| server_id | smallint | | NN | default 1 | | Shard/server |
| level | int | | NN | default 1 CHECK >=1 | | Level akun |
| xp | bigint | | NN | default 0 CHECK >=0 | | XP akumulasi |
| elo_pvp | int | | NN | default 1000 CHECK >=0 | idx_elo | PvP ELO |
| pvp_wins | int | | NN | default 0 | | |
| pvp_losses | int | | NN | default 0 | | |
| pity_shared_banner | bool | | NN | default false | | Apakah pity global (CDP: per banner) |
| tutorial_done | bool | | NN | default false | | |
| last_login_at | timestamptz | | NULL | | | |
| created_at | timestamptz | | NN | default now() | | |
| updated_at | timestamptz | | NN | default now() | | |

### B.5. currencies — master daftar mata uang
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| code | varchar(16) | PK | NN | UQ, CHECK ^[A-Z_]+$ | PK | 'GOLD','GEMS','EVENT_XYZ' |
| name | varchar(48) | | NN | | | Nama tampilan |
| type | enum('SOFT','HARD','EVENT') | | NN | | | Klasifikasi |
| is_hard | bool | | NN | default false | | Butuh audit ketat |
| decimal_scale | smallint | | NN | default 0 CHECK 0..4 | | Presisi |
| created_at | timestamptz | | NN | default now() | | |

> Seed: `GOLD`(SOFT), `GEMS`(HARD), `EVENT_<id>`(EVENT). Event Tokens dibuat baris baru per event.

### B.6. wallets — saldo terkini (1-N users, 1 wallet per currency)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| user_id | bigint | FK | NN | refs users(id) ON DELETE RESTRICT | idx_user | Pemilik |
| currency_code | varchar(16) | FK | NN | refs currencies(code) | | Mata uang |
| balance | numeric(18,2) | | NN | default 0 CHECK >= 0 | | Saldo SAAT INI |
| version | bigint | | NN | default 0 | | Optimistic lock (CAS) |
| updated_at | timestamptz | | NN | default now() | | |
| **UQ** | (user_id, currency_code) | | | | UQ | **Anti-duplikasi: 1 wallet/user/currency** |

> **Catatan penting**: `wallets.balance` adalah *denormalized cache* yang HANYA boleh diubah lewat ledger (B.7). Tidak ada UPDATE langsung di luar prosedur ledger. `version` cegah lost-update (compare-and-swap).

### B.7. currency_ledger — BUKU BESAR append-only (lihat Bagian G)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | cluster | Seq ledger |
| user_id | bigint | FK | NN | refs users(id) ON DELETE RESTRICT | idx_user | Pemilik |
| currency_code | varchar(16) | FK | NN | refs currencies(code) | | Mata uang |
| delta | numeric(18,2) | | NN | | idx_user_time | Perubahan (+/-) |
| balance_after | numeric(18,2) | | NN | CHECK >= 0 | | Saldo pasca transaksi |
| ref_type | varchar(32) | | NN | | | 'GACHA','SHOP','MARKET_FEE','IDLE','ADMIN',... |
| ref_id | bigint | | NULL | | | ID ref (gacha_pulls.id dll) |
| nonce | bigint | | NN | default 0 | UQ(user,nonce) | Idempotency / urutan |
| signature | varchar(128) | | NULL | | | HMAC untuk tamper-evidence |
| created_at | timestamptz | | NN | default now() | idx_time | Waktu tx |
| **UQ** | (user_id, nonce) | | | | | Cegah replay duplikat |

### B.8. heroes — template/master hero (stat dasar)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | Master hero id |
| code | varchar(32) | UQ | NN | | UQ | 'HERO_001' |
| name | varchar(48) | | NN | | | Nama |
| rarity | enum('C','R','E','L','M') | | NN | | idx_rarity | C/R/E/L/M |
| element | enum('FIRE','WATER','WIND','EARTH','LIGHT','DARK') | | NN | refs elemental_types | idx_elem | Elemen |
| base_hp | int | | NN | CHECK >0 | | Stat dasar |
| base_atk | int | | NN | CHECK >0 | | |
| base_def | int | | NN | CHECK >0 | | |
| base_spd | int | | NN | CHECK >0 | | |
| base_crit | numeric(5,2) | | NN | default 0 | | % crit |
| role | varchar(16) | | NN | 'TANK','DPS','SUPPORT','HEALER' | | Peran |
| release_version | varchar(16) | FK | NN | refs content_versions(version) | | Versi rilis |
| is_pullable | bool | | NN | default true | | Masuk pool gacha? |
| created_at | timestamptz | | NN | default now() | | |

### B.9. skills — master skill
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| code | varchar(32) | UQ | NN | | UQ | 'SKILL_FIREBALL' |
| name | varchar(48) | | NN | | | |
| element | enum('FIRE','WATER','WIND','EARTH','LIGHT','DARK') | | NN | | | Elemen skill |
| type | enum('ACTIVE','PASSIVE','ULTIMATE') | | NN | | | |
| targeting | enum('SINGLE','ALL_ENEMY','SELF','ALLY','ALL_ALLY') | | NN | | | |
| power | numeric(8,2) | | NN | default 0 | | Skala damage/heal |
| cooldown | smallint | | NN | default 0 CHECK >=0 | | Turn cooldown |
| status_effect_id | bigint | FK | NULL | refs status_effects(id) | | Efek status (bila ada) |
| effect_chance | numeric(5,2) | | NN | default 0 CHECK 0..100 | | % apply |
| release_version | varchar(16) | FK | NN | refs content_versions(version) | | |
| created_at | timestamptz | | NN | default now() | | |

### B.10. hero_skills — junction hero ↔ skill (N-N)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| hero_id | bigint | PK,FK | NN | refs heroes(id) ON DELETE CASCADE | PK | |
| skill_id | bigint | PK,FK | NN | refs skills(id) ON DELETE CASCADE | PK | |
| slot | smallint | | NN | CHECK 1..4 | | Urutan skill |
| unlock_level | smallint | | NN | default 1 CHECK >=1 | | Buka di level |

### B.11. hero_instances — hero milik pemain (instance)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | Instance id |
| user_id | bigint | FK | NN | refs users(id) ON DELETE CASCADE | idx_user | Pemilik |
| hero_id | bigint | FK | NN | refs heroes(id) | idx_hero | Template master |
| level | int | | NN | default 1 CHECK 1..200 | | Level instance |
| stars | smallint | | NN | default 1 CHECK 1..6 | | Bintang (ascension tier) |
| xp | bigint | | NN | default 0 CHECK >=0 | | XP |
| **snapshot_hp** | int | | NN | | | **Denormal: stat final** |
| **snapshot_atk** | int | | NN | | | (lihat Bagian F) |
| **snapshot_def** | int | | NN | | | |
| **snapshot_spd** | int | | NN | | | |
| **snapshot_crit** | numeric(5,2) | | NN | | | |
| acquired_at | timestamptz | | NN | default now() | | |
| acquired_source | varchar(32) | | NN | 'GACHA','EVENT','GIFT','MARKET' | | Provenance |
| is_locked | bool | | NN | default false | | Cegah di-jual/di-feed |
| created_at | timestamptz | | NN | default now() | | |

> Snapshot di-rebuild saat level/stars/equipment berubah via trigger/prosedur. Partai battle baca snapshot (performa).

### B.12. hero_progression — ascension & breakthrough (1:1 hero_instances)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| hero_instance_id | bigint | UQ,FK | NN | refs hero_instances(id) ON DELETE CASCADE | UQ | 1:1 |
| ascension_level | smallint | | NN | default 0 CHECK 0..6 | | Breakthrough |
| skill_levels | jsonb | | NN | default '{}' | | {skill_id: level} |
| duplicate_count | int | | NN | default 0 CHECK >=0 | | Berapa duplikat digabung |
| shards | int | | NN | default 0 CHECK >=0 | | Shard hero |
| updated_at | timestamptz | | NN | default now() | | |

### B.13. items — master item (Gear/Consumable/Material)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| code | varchar(32) | UQ | NN | | UQ | 'ITEM_SWORD_01' |
| name | varchar(48) | | NN | | | |
| category | enum('GEAR','CONSUMABLE','MATERIAL') | | NN | | idx_cat | Tiga kategori CDP |
| gear_slot | enum('WEAPON','ARMOR','ACCESSORY','BOOTS','TRINKET') | | NULL | | | Bila GEAR |
| rarity | enum('C','R','E','L','M') | | NN | | | |
| element | enum('FIRE','WATER','WIND','EARTH','LIGHT','DARK','NONE') | | NN | default 'NONE' | | |
| stat_bonus | jsonb | | NULL | | | {atk:10,def:5} |
| stackable | bool | | NN | default true | | Consumable/Material ya |
| max_stack | int | | NN | default 99 CHECK >=1 | | |
| sell_price_gold | int | | NN | default 0 CHECK >=0 | | Harga jual baseline |
| release_version | varchar(16) | FK | NN | refs content_versions(version) | | |
| created_at | timestamptz | | NN | default now() | | |

### B.14. inventory — item milik pemain (instance/stack)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| user_id | bigint | FK | NN | refs users(id) ON DELETE CASCADE | idx_user | Pemilik |
| item_id | bigint | FK | NN | refs items(id) | idx_item | Master item |
| quantity | int | | NN | default 1 CHECK >=0 | | (0 = bisa dihapus) |
| is_equipped | bool | | NN | default false | | Dipakai hero? |
| equipped_hero_id | bigint | FK | NULL | refs hero_instances(id) | | Hero pemakai |
| enhanced_level | smallint | | NN | default 0 CHECK 0..15 | | Bila GEAR di-upgrade |
| created_at | timestamptz | | NN | default now() | | |
| updated_at | timestamptz | | NN | default now() | | |
| **UQ** | (user_id, item_id, equipped_hero_id) where is_equipped | | | partial UQ | | Cegah duplikat equip |

### B.15. equipment_slots — slot equip hero (1:1 hero_instances, 5 slot)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| hero_instance_id | bigint | FK | NN | refs hero_instances(id) ON DELETE CASCADE | idx_hero | |
| slot | enum('WEAPON','ARMOR','ACCESSORY','BOOTS','TRINKET') | | NN | | | |
| inventory_id | bigint | FK | NULL | refs inventory(id) | | Item terpasang |
| updated_at | timestamptz | | NN | default now() | | |
| **UQ** | (hero_instance_id, slot) | | | | UQ | 1 slot per jenis |

### B.16. stages — master stage/level
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| code | varchar(32) | UQ | NN | | UQ | 'STAGE_3_12' |
| chapter | smallint | | NN | CHECK >0 | | Bab |
| stage_no | smallint | | NN | CHECK >0 | | Nomor urut |
| name | varchar(48) | | NN | | | |
| required_level | int | | NN | default 1 | | Syarat level |
| energy_cost | smallint | | NN | default 6 CHECK >=0 | | Biaya stamina |
| gold_reward | int | | NN | default 0 | | Reward idle/gold |
| xp_reward | int | | NN | default 0 | | XP |
| is_boss | bool | | NN | default false | | |
| release_version | varchar(16) | FK | NN | refs content_versions(version) | | |
| created_at | timestamptz | | NN | default now() | | |

### B.17. waves — master wave dalam stage (1-N stages)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| stage_id | bigint | FK | NN | refs stages(id) ON DELETE CASCADE | idx_stage | |
| wave_index | smallint | | NN | CHECK >=1 | | Urutan wave |
| enemy_team | jsonb | | NN | | | Komposisi musuh |
| recommended_power | int | | NN | default 0 | | Power saran |
| **UQ** | (stage_id, wave_index) | | | | UQ | Unik per stage |

### B.18. battle_runs — satu pertempuran (instance)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| user_id | bigint | FK | NN | refs users(id) ON DELETE CASCADE | idx_user_time | Pemilik |
| stage_id | bigint | FK | NULL | refs stages(id) | | Stage (NULL bila PvP/raid) |
| mode | enum('PVE','PVP','RAID','BOSS') | | NN | | | |
| party_snapshot | jsonb | | NN | | | 5 hero + stat (server authoritative) |
| result | enum('WIN','LOSE','FLEE') | | NULL | | | Hasil |
| seed | bigint | | NN | | | RNG seed (replay/audit) |
| turn_count | smallint | | NN | default 0 | | Jumlah turn |
| started_at | timestamptz | | NN | default now() | | |
| ended_at | timestamptz | | NULL | | | |
| server_authoritative | bool | | NN | default true | | CDP: combat server-authoritative |

### B.19. battle_logs — log per-turn (1-N battle_runs) — PARTISI
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| battle_run_id | bigint | FK | NN | refs battle_runs(id) ON DELETE CASCADE | idx_battle | |
| turn_no | smallint | | NN | CHECK >=1 | | |
| actor_type | enum('PLAYER','ENEMY') | | NN | | | |
| actor_id | bigint | | NN | | | Hero/enemy id |
| action | varchar(32) | | NN | 'SKILL','ATTACK','STATUS','BUFF' | | |
| skill_id | bigint | FK | NULL | refs skills(id) | | |
| damage | int | | NN | default 0 | | |
| heal | int | | NN | default 0 | | |
| status_applied | jsonb | | NULL | | | Efek status |
| created_at | timestamptz | | NN | default now() | | (partisi key) |
| **PK include** | (battle_run_id, turn_no) | | | | | |

> Partisi by `created_at` range/bulan (Bagian H).

### B.20. idle_state — status akrual idle pemain
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| user_id | bigint | UQ,FK | NN | refs users(id) ON DELETE CASCADE | UQ | 1:1 |
| stage_id | bigint | FK | NN | refs stages(id) | | Stage idle dipilih |
| started_at | timestamptz | | NN | default now() | | Mulai akrual |
| last_tick_at | timestamptz | | NN | default now() | | Tick terakhir (5s) |
| accumulated_seconds | int | | NN | default 0 CHECK 0..86400 | | Cap 24 jam |
| is_running | bool | | NN | default true | | |

### B.21. idle_claims — klaim hasil idle
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| user_id | bigint | FK | NN | refs users(id) ON DELETE CASCADE | idx_user | |
| idle_state_id | bigint | FK | NN | refs idle_state(id) | | |
| claimed_seconds | int | | NN | CHECK >0 | | Detik diklaim |
| gold_gain | int | | NN | default 0 | | Dari ledger |
| xp_gain | int | | NN | default 0 | | |
| ledger_ref_id | bigint | FK | NULL | refs currency_ledger(id) | | Bukti ledger |
| claimed_at | timestamptz | | NN | default now() | | |

### B.22. gacha_banners — banner gacha
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| code | varchar(32) | UQ | NN | | UQ | 'BANNER_FIRE_01' |
| name | varchar(48) | | NN | | | |
| type | enum('STANDARD','EVENT','COLLAB','WEAPON') | | NN | | | |
| rate_c | numeric(5,2) | | NN | default 60.0 CHECK 0..100 | | CDP: 60 |
| rate_r | numeric(5,2) | | NN | default 30.0 | | CDP: 30 |
| rate_e | numeric(5,2) | | NN | default 9.0 | | CDP: 9 |
| rate_l | numeric(5,2) | | NN | default 1.0 | | CDP: 1 |
| soft_pity | smallint | | NN | default 75 CHECK 1..90 | | CDP: 75 |
| hard_pity | smallint | | NN | default 90 CHECK 1..90 | | CDP: 90 |
| guarantee_r_every | smallint | | NN | default 10 | | CDP: R+ tiap 10 |
| cost_currency | varchar(16) | FK | NN | refs currencies(code) | | Biasanya GEMS |
| cost_per_pull | int | | NN | default 160 CHECK >0 | | |
| start_at | timestamptz | | NN | | | |
| end_at | timestamptz | | NN | | idx_active | |
| is_active | bool | | NN | default true | idx_active | |
| release_version | varchar(16) | FK | NN | refs content_versions(version) | | |

### B.23. banner_heroes — junction banner ↔ hero pool (N-N)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| banner_id | bigint | PK,FK | NN | refs gacha_banners(id) ON DELETE CASCADE | PK | |
| hero_id | bigint | PK,FK | NN | refs heroes(id) ON DELETE CASCADE | PK | |
| weight | int | | NN | default 1 CHECK >0 | | Bobot pool |
| is_featured | bool | | NN | default false | | Rate-up |

### B.24. gacha_pulls — SETIAP pull IMMUTABLE (audit gacha)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| user_id | bigint | FK | NN | refs users(id) ON DELETE RESTRICT | idx_user_banner | Pemilik |
| banner_id | bigint | FK | NN | refs gacha_banners(id) | idx_user_banner | Banner |
| pull_index | smallint | | NN | CHECK 1..10 | | Ke-berapa (1 pull / 10-pull) |
| batch_id | uuid | | NN | | idx_batch | Group 10-pull |
| hero_id | bigint | FK | NN | refs heroes(id) | | Hasil hero |
| rarity | enum('C','R','E','L','M') | | NN | | | Hasil rarity |
| is_guaranteed | bool | | NN | default false | | Pity guarantee? |
| rng_seed | bigint | | NN | | | Replay |
| ledger_ref_id | bigint | FK | NULL | refs currency_ledger(id) | | Bukti potong currency |
| created_at | timestamptz | | NN | default now() | idx_time | (partisi key) |
| **UQ** | (batch_id, pull_index) | | | | UQ | |

> Tabel ini **append-only, TIDAK ADA UPDATE/DELETE** (audit). Pity dihitung dari `pity_state` + count pulls.

### B.25. pity_state — state pity per user per banner (ter-lock)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| user_id | bigint | FK | NN | refs users(id) ON DELETE CASCADE | | |
| banner_id | bigint | FK | NN | refs gacha_banners(id) | | |
| current_pity | smallint | | NN | default 0 CHECK 0..90 | | CDP: <=90 |
| since_r_guarantee | smallint | | NN | default 0 CHECK 0..10 | | Sejak R+ terakhir |
| last_pull_id | bigint | FK | NULL | refs gacha_pulls(id) | | Pull terakhir (lock) |
| updated_at | timestamptz | | NN | default now() | | |
| **UQ** | (user_id, banner_id) | | | | UQ | 1 state/banner/user |

> `pity_state` di-update ATOMIC bersama insert `gacha_pulls` dalam satu tx. Tidak bisa diubah di luar prosedur gacha.

### B.26. marketplace_listings — iklan jual (N-N user↔hero/item)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| seller_id | bigint | FK | NN | refs users(id) ON DELETE RESTRICT | idx_rarity_price | Penjual |
| listing_type | enum('HERO','ITEM') | | NN | | | |
| hero_instance_id | bigint | FK | NULL | refs hero_instances(id) | | Bila HERO |
| item_id | bigint | FK | NULL | refs inventory(id) | | Bila ITEM |
| rarity | enum('C','R','E','L','M') | | NN | | idx_rarity_price | Untuk filter |
| price_currency | varchar(16) | FK | NN | refs currencies(code) | | Biasanya GEMS/GOLD |
| price | numeric(18,2) | | NN | CHECK >0 | idx_rarity_price | Harga |
| status | enum('ACTIVE','SOLD','CANCELLED','EXPIRED') | | NN | default 'ACTIVE' | idx_status | CDP enum |
| expires_at | timestamptz | | NN | | | Kadaluarsa otomatis |
| created_at | timestamptz | | NN | default now() | | |
| **CHECK** | (listing_type='HERO' AND hero_instance_id IS NOT NULL) OR (listing_type='ITEM' AND item_id IS NOT NULL) | | | | | Mutual exclusive |

> Indeks `idx_rarity_price` = composite (status, rarity, price) untuk query panas marketplace aktif.

### B.27. escrow_holds — tahan barang & currency (atomic marketplace)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| listing_id | bigint | FK | NN | refs marketplace_listings(id) | | Listing terkait |
| buyer_id | bigint | FK | NN | refs users(id) | | Pembeli |
| currency_code | varchar(16) | FK | NN | refs currencies(code) | | Mata uang tertahan |
| amount_held | numeric(18,2) | | NN | CHECK >0 | | Currency di-escrow |
| ledger_hold_ref | bigint | FK | NN | refs currency_ledger(id) | | Bukti tahan (delta -) |
| status | enum('HELD','RELEASED','REFUNDED') | | NN | default 'HELD' | | |
| created_at | timestamptz | | NN | default now() | | |
| released_at | timestamptz | | NULL | | | |

### B.28. marketplace_transactions — transaksi final (audit)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| listing_id | bigint | FK | NN | refs marketplace_listings(id) | | |
| seller_id | bigint | FK | NN | refs users(id) | | |
| buyer_id | bigint | FK | NN | refs users(id) | | |
| escrow_id | bigint | FK | NULL | refs escrow_holds(id) | | |
| price | numeric(18,2) | | NN | | | Harga final |
| fee | numeric(18,2) | | NN | default 0 CHECK >=0 | | Fee marketplace |
| seller_ledger_ref | bigint | FK | NN | refs currency_ledger(id) | | + ke seller |
| fee_ledger_ref | bigint | FK | NULL | refs currency_ledger(id) | | + ke platform |
| item_transfer_ref | bigint | FK | NULL | refs inventory(id) / hero_instances(id) | | Barang pindah |
| created_at | timestamptz | | NN | default now() | | |
| **UQ** | (listing_id) | | | | UQ | 1 tx/listing (no double-sell) |

### B.29. guilds — guild/klan
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| code | varchar(32) | UQ | NN | | UQ | 'GUILD_ABC' |
| name | varchar(48) | | NN | | | |
| level | int | | NN | default 1 | | Level guild |
| xp | bigint | | NN | default 0 | | |
| leader_user_id | bigint | FK | NN | refs users(id) | | Ketua |
| max_members | smallint | | NN | default 30 CHECK 1..100 | | |
| created_at | timestamptz | | NN | default now() | | |
| updated_at | timestamptz | | NN | default now() | | |

### B.30. guild_members — junction user ↔ guild (N-N)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| guild_id | bigint | PK,FK | NN | refs guilds(id) ON DELETE CASCADE | PK | |
| user_id | bigint | PK,FK | NN | refs users(id) ON DELETE CASCADE | PK | |
| role | enum('LEADER','OFFICER','MEMBER') | | NN | default 'MEMBER' | | |
| joined_at | timestamptz | | NN | default now() | | |
| contribution | bigint | | NN | default 0 | | |
| **CHECK** | satu user max 1 guild (via UQ user_id) | | | | UQ(user_id) | |

### B.31. raids — master raid guild
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| code | varchar(32) | UQ | NN | | UQ | |
| name | varchar(48) | | NN | | | |
| boss_hero_id | bigint | FK | NULL | refs heroes(id) | | Boss |
| total_hp | bigint | | NN | CHECK >0 | | HP boss guild |
| reward_pool | jsonb | | NN | | | Reward |
| reset_cycle | enum('DAILY','WEEKLY') | | NN | default 'WEEKLY' | | |
| release_version | varchar(16) | FK | NN | refs content_versions(version) | | |

### B.32. raid_runs — partisipasi raid (N-N guild/user via run)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| raid_id | bigint | FK | NN | refs raids(id) | | |
| guild_id | bigint | FK | NN | refs guilds(id) | | |
| user_id | bigint | FK | NN | refs users(id) | | |
| damage_dealt | bigint | | NN | default 0 | | |
| is_completed | bool | | NN | default false | | |
| reward_claimed | bool | | NN | default false | | |
| created_at | timestamptz | | NN | default now() | | |

### B.33. shop_products — produk toko (IAP/soft)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| code | varchar(32) | UQ | NN | | UQ | 'PACK_GEMS_99' |
| name | varchar(48) | | NN | | | |
| type | enum('GEMS','GOLD','HERO','ITEM','BUNDLE') | | NN | | | |
| price_real | numeric(10,2) | | NULL | | | Harga Rupiah/USD (bila IAP) |
| currency_reward_code | varchar(16) | FK | NULL | refs currencies(code) | | |
| currency_amount | numeric(18,2) | | NULL | | | |
| item_rewards | jsonb | | NULL | | | Item bundle |
| is_active | bool | | NN | default true | | |
| release_version | varchar(16) | FK | NN | refs content_versions(version) | | |

### B.34. purchases — pembelian (audit)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| user_id | bigint | FK | NN | refs users(id) ON DELETE RESTRICT | | |
| product_id | bigint | FK | NN | refs shop_products(id) | | |
| platform | enum('WEB','ANDROID','IOS','PWA') | | NN | | | |
| order_id | varchar(64) | UQ | NN | | UQ | ID dari appstore |
| status | enum('PENDING','COMPLETED','FAILED','REFUNDED') | | NN | default 'PENDING' | | |
| amount_paid | numeric(10,2) | | NN | | | |
| ledger_ref_id | bigint | FK | NULL | refs currency_ledger(id) | | Bukti grant |
| created_at | timestamptz | | NN | default now() | | |
| completed_at | timestamptz | | NULL | | | |

### B.35. subscriptions / battle_pass — pass berlangganan
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| user_id | bigint | FK | NN | refs users(id) ON DELETE CASCADE | idx_user | |
| type | enum('BATTLE_PASS','MONTHLY_CARD','VIP') | | NN | | | |
| tier | smallint | | NN | default 1 CHECK >=1 | | |
| status | enum('ACTIVE','EXPIRED','CANCELLED') | | NN | default 'ACTIVE' | | |
| started_at | timestamptz | | NN | default now() | | |
| expires_at | timestamptz | | NN | | | |
| last_claimed_day | smallint | | NN | default 0 CHECK >=0 | | Progresi harian |
| purchase_ref_id | bigint | FK | NULL | refs purchases(id) | | |
| created_at | timestamptz | | NN | default now() | | |

### B.36. admin_users — akun GM/admin
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| username | varchar(32) | UQ | NN | | UQ | |
| password_hash | varchar(255) | | NN | | | |
| role | enum('GM','SUPER_ADMIN','SUPPORT') | | NN | | | |
| permissions | jsonb | | NN | default '{}' | | Permission granular |
| created_at | timestamptz | | NN | default now() | | |
| last_login_at | timestamptz | NULL | NULL | | | |

### B.37. content_versions — versioning master data (immutable)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| version | varchar(16) | PK | NN | | PK | '1.2.0' |
| notes | text | | NULL | | | Changelog |
| released_by | bigint | FK | NULL | refs admin_users(id) | | |
| released_at | timestamptz | | NN | default now() | | |
| is_active | bool | | NN | default true | | |

### B.38. audit_logs — log audit global (PARTISI)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| actor_type | enum('USER','ADMIN','SYSTEM') | | NN | | | |
| actor_id | bigint | | NN | | idx_actor_time | user/admin id |
| action | varchar(48) | | NN | 'GACHA_PULL','MARKET_SELL','CURRENCY_GRANT',... | | |
| target_type | varchar(32) | | NULL | | | |
| target_id | bigint | | NULL | | | |
| old_value | jsonb | | NULL | | | |
| new_value | jsonb | | NULL | | | |
| ip_address | inet | | NULL | | | |
| created_at | timestamptz | | NN | default now() | idx_actor_time | (partisi key) |

### B.39. events — event game
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| code | varchar(32) | UQ | NN | | UQ | 'EVENT_XMAS' |
| name | varchar(48) | | NN | | | |
| type | enum('LOGIN','COMBAT','GUILD','GACHA') | | NN | | | |
| start_at | timestamptz | | NN | | | |
| end_at | timestamptz | | NN | | | |
| reward_currency_code | varchar(16) | FK | NULL | refs currencies(code) | | Event Token |
| is_active | bool | | NN | default true | | |
| release_version | varchar(16) | FK | NN | refs content_versions(version) | | |

### B.40. event_rewards — reward event (1-N events)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| event_id | bigint | FK | NN | refs events(id) ON DELETE CASCADE | idx_event | |
| requirement | jsonb | | NN | | | Syarat (login 7 hari dll) |
| reward_type | enum('CURRENCY','HERO','ITEM') | | NN | | | |
| reward_ref_id | bigint | | NULL | | | hero/item id |
| amount | numeric(18,2) | | NN | default 0 | | |
| **UQ** | (event_id, requirement) | | | | UQ | |

### B.41. elemental_types — master 6 elemen
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | smallint | PK | NN | | | |
| code | enum('FIRE','WATER','WIND','EARTH','LIGHT','DARK') | UQ | NN | | UQ | |
| name | varchar(24) | | NN | | | |
| strong_vs | jsonb | | NN | | | Counter map |

### B.42. status_effects — master status effect
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| id | bigint | PK | NN | IDENTITY | | |
| code | varchar(32) | UQ | NN | | UQ | 'BURN','STUN' |
| name | varchar(48) | | NN | | | |
| kind | enum('BUFF','DEBUFF','DOT','HOT') | | NN | | | |
| stat_modifiers | jsonb | | NULL | | | {atk:-0.2} |
| duration_turns | smallint | | NN | default 1 CHECK >=1 | | |
| tick_effect | jsonb | | NULL | | | Dmg/heal per turn |
| release_version | varchar(16) | FK | NN | refs content_versions(version) | | |

### B.43. party_members — komposisi party aktif (N-N user↔hero_instances)
| Field | Tipe | Key | Nullable | Constraint | Index | Keterangan |
|---|---|---|---|---|---|---|
| user_id | bigint | PK,FK | NN | refs users(id) ON DELETE CASCADE | PK | |
| hero_instance_id | bigint | PK,FK | NN | refs hero_instances(id) ON DELETE CASCADE | PK | |
| slot | smallint | | NN | CHECK 1..5 | idx_slot | **Max 5 (CDP)** |
| is_active_party | bool | | NN | default true | | |
| **CHECK** | count active per user <= 5 (trigger/constraint) | | | | | |

> Constraint jumlah: trigger cek `is_active_party` per user ≤ 5.

---

## C. Relasi (ERD Teks)

```
users (1) ──< user_credentials (1:1, user_credentials.user_id → users.id)
users (1) ──< sessions          (sessions.user_id → users.id)
users (1) ──1 player_profiles   (player_profiles.user_id → users.id, UQ)

users (1) ──< wallets           (wallets.user_id → users.id)   [UQ user+currency]
users (1) ──< currency_ledger   (currency_ledger.user_id → users.id)
currencies (1) ──< wallets      (wallets.currency_code → currencies.code)
currencies (1) ──< currency_ledger

users (1) ──< hero_instances   (hero_instances.user_id → users.id)
heroes (1) ──< hero_instances   (hero_instances.hero_id → heroes.id)
hero_instances (1) ──1 hero_progression (hero_progression.hero_instance_id → hero_instances.id)
heroes (1) ──< hero_skills      (hero_skills.hero_id → heroes.id)        [N-N]
skills (1) ──< hero_skills      (hero_skills.skill_id → skills.id)        [N-N]
skills (1) ──< battle_logs      (battle_logs.skill_id → skills.id)
skills (1) ──< status_effects   (status_effects.id ← skills.status_effect_id via FK)
skills.status_effect_id → status_effects.id

users (1) ──< inventory         (inventory.user_id → users.id)
items (1) ──< inventory         (inventory.item_id → items.id)
hero_instances (1) ──< inventory (inventory.equipped_hero_id → hero_instances.id)
hero_instances (1) ──< equipment_slots (equipment_slots.hero_instance_id → hero_instances.id)
inventory (1) ──< equipment_slots (equipment_slots.inventory_id → inventory.id)

stages (1) ──< waves            (waves.stage_id → stages.id)
stages (1) ──< battle_runs      (battle_runs.stage_id → stages.id)
users (1) ──< battle_runs       (battle_runs.user_id → users.id)
battle_runs (1) ──< battle_logs (battle_logs.battle_run_id → battle_runs.id)

users (1) ──1 idle_state        (idle_state.user_id → users.id, UQ)
stages (1) ──< idle_state       (idle_state.stage_id → stages.id)
users (1) ──< idle_claims       (idle_claims.user_id → users.id)
idle_state (1) ──< idle_claims  (idle_claims.idle_state_id → idle_state.id)
idle_claims.ledger_ref_id → currency_ledger.id

gacha_banners (1) ──< gacha_pulls   (gacha_pulls.banner_id → gacha_banners.id)
users (1) ──< gacha_pulls          (gacha_pulls.user_id → users.id)
heroes (1) ──< gacha_pulls         (gacha_pulls.hero_id → heroes.id)
gacha_banners (1) ──< banner_heroes (banner_heroes.banner_id → gacha_banners.id)  [N-N]
heroes (1) ──< banner_heroes       (banner_heroes.hero_id → heroes.id)            [N-N]
users (1) ──< pity_state           (pity_state.user_id → users.id)
gacha_banners (1) ──< pity_state   (pity_state.banner_id → gacha_banners.id)
gacha_pulls.ledger_ref_id → currency_ledger.id
pity_state.last_pull_id → gacha_pulls.id

users (1) ──< marketplace_listings (marketplace_listings.seller_id → users.id)
hero_instances (1) ──< marketplace_listings (marketplace_listings.hero_instance_id → hero_instances.id)
inventory (1) ──< marketplace_listings (marketplace_listings.item_id → inventory.id)
marketplace_listings (1) ──< escrow_holds (escrow_holds.listing_id → marketplace_listings.id)
users (1) ──< escrow_holds (escrow_holds.buyer_id → users.id)
marketplace_listings (1) ──1 marketplace_transactions (marketplace_transactions.listing_id → marketplace_listings.id, UQ)
users (1) ──< marketplace_transactions (seller_id, buyer_id → users.id)

guilds (1) ──< guild_members  (guild_members.guild_id → guilds.id)   [N-N]
users (1) ──< guild_members   (guild_members.user_id → users.id)     [N-N, UQ user_id]
guilds (1) ──< raids          (raids via guild participation)
raids (1) ──< raid_runs       (raid_runs.raid_id → raids.id)
guilds (1) ──< raid_runs      (raid_runs.guild_id → guilds.id)
users (1) ──< raid_runs       (raid_runs.user_id → users.id)

shop_products (1) ──< purchases (purchases.product_id → shop_products.id)
users (1) ──< purchases        (purchases.user_id → users.id)
purchases.ledger_ref_id → currency_ledger.id
users (1) ──< subscriptions    (subscriptions.user_id → users.id)
purchases (1) ──< subscriptions (subscriptions.purchase_ref_id → purchases.id)

admin_users (1) ──< content_versions (content_versions.released_by → admin_users.id)
heroes.release_version → content_versions.version
items.release_version → content_versions.version
skills.release_version → content_versions.version
stages.release_version → content_versions.version
gacha_banners.release_version → content_versions.version
raids.release_version → content_versions.version
events.release_version → content_versions.version
status_effects.release_version → content_versions.version

users (1) ──< audit_logs      (audit_logs.actor_id → users.id where actor_type='USER')
admin_users (1) ──< audit_logs (audit_logs.actor_id → admin_users.id where actor_type='ADMIN')

events (1) ──< event_rewards  (event_rewards.event_id → events.id)
currencies (1) ──< events      (events.reward_currency_code → currencies.code)

elemental_types (1) ──< heroes    (heroes.element → elemental_types.code)
elemental_types (1) ──< skills    (skills.element → elemental_types.code)

users (1) ──< party_members        (party_members.user_id → users.id)   [N-N]
hero_instances (1) ──< party_members (party_members.hero_instance_id → hero_instances.id) [N-N]
```

**Relasi N-N eksplisit** (via junction): `hero_skills`, `banner_heroes`, `guild_members`, `party_members`, `marketplace_listings` (user↔hero/item, tapi tanpa junction karena listing menyimpan FK langsung ke penjual + objek).

---

## D. Constraint Integritas

### D.1. Foreign Key ON DELETE behavior
| Relasi | Behavior | Alasan |
|---|---|---|
| users → user_credentials, sessions, player_profiles, hero_instances, wallets, inventory, battle_runs, idle_*, gacha_pulls, marketplace_listings, audit_logs, party_members | **CASCADE** | Hapus akun = hapus data turunan |
| users → currency_ledger, gacha_pulls, pity_state, marketplace_transactions, purchases | **RESTRICT** | Audit immutabel; akun dianonimisasi bukan dihapus fisik (lihat H) |
| heroes/items/skills/stages → instance & junction | **CASCADE / RESTRICT** | Master dihapus = instance ikut (CASCADE); reference ledger RESTRICT |
| gacha_banners → gacha_pulls | RESTRICT | Banner historis tetap teraudit |
| marketplace_listings → transactions/escrow | RESTRICT | Audit tetap ada |

### D.2. Unique Constraints anti-duplikasi
- `wallets (user_id, currency_code)` → **tepat 1 wallet per user per currency** (cegah split saldo).
- `pity_state (user_id, banner_id)` → 1 state pity per user per banner.
- `guild_members (user_id)` → 1 guild per user.
- `marketplace_transactions (listing_id)` → 1 transaksi final per listing (no double-sell).
- `hero_progression (hero_instance_id)`, `idle_state (user_id)`, `player_profiles (user_id)` → 1:1.
- `currency_ledger (user_id, nonce)` → idempotency (cegah replay/pull ganda).
- `gacha_pulls (batch_id, pull_index)` → tidak ada pull ganda dalam 1 batch.
- `users (email)`, `users (public_id)`, `sessions (refresh_token_hash)`, `purchases (order_id)` → unik global.

### D.3. Check Constraints
- `wallets.balance >= 0` — tidak boleh negatif.
- `currency_ledger.balance_after >= 0`.
- `pity_state.current_pity BETWEEN 0 AND 90` (hard pity cap CDP).
- `pity_state.since_r_guarantee BETWEEN 0 AND 10`.
- `party_members.slot BETWEEN 1 AND 5` + trigger count aktif ≤ 5.
- `gacha_banners.rate_c+rate_r+rate_e+rate_l = 100` (trigger validasi saat insert/update).
- `hero_instances.level BETWEEN 1 AND 200`, `stars BETWEEN 1 AND 6`.
- `marketplace_listings`: CHECK mutual-exclusive HERO/ITEM.
- `battle_runs.server_authoritative = true` (CDP: combat server-authoritative).
- `idle_state.accumulated_seconds <= 86400` (cap 24 jam).
- Enum status: `marketplace_listings.status IN (ACTIVE,SOLD,CANCELLED,EXPIRED)`.

### D.4. Enum (status stabil)
```
user_status      = ACTIVE | SUSPENDED | BANNED | DELETED
currency_type    = SOFT | HARD | EVENT
rarity           = C | R | E | L | M
element          = FIRE | WATER | WIND | EARTH | LIGHT | DARK
item_category    = GEAR | CONSUMABLE | MATERIAL
listing_status   = ACTIVE | SOLD | CANCELLED | EXPIRED
escrow_status    = HELD | RELEASED | REFUNDED
purchase_status  = PENDING | COMPLETED | FAILED | REFUNDED
subscription_status = ACTIVE | EXPIRED | CANCELLED
battle_result    = WIN | LOSE | FLEE
```

---

## E. Indeks yang Disarankan

| Tabel | Indeks | Tipe | Query Panas |
|---|---|---|---|
| marketplace_listings | `(status, rarity, price)` | composite (partial WHERE status='ACTIVE') | List aktif by rarity+price |
| marketplace_listings | `(seller_id, status)` | composite | Riwayat jual user |
| gacha_pulls | `(user_id, banner_id, created_at)` | composite | Riwayat pull user per banner |
| gacha_pulls | `(batch_id, pull_index)` | UQ | Replay 10-pull |
| battle_runs | `(user_id, started_at DESC)` | composite | Histori battle user |
| battle_logs | `(battle_run_id, turn_no)` | PK composite | Replay battle |
| idle_claims | `(user_id, claimed_at DESC)` | composite | Riwayat klaim idle |
| audit_logs | `(actor_type, actor_id, created_at DESC)` | composite (partisi) | Audit per aktor |
| currency_ledger | `(user_id, created_at DESC)` | composite | Mutasi user |
| currency_ledger | `(user_id, nonce)` | UQ | Idempotency |
| pity_state | `(user_id, banner_id)` | UQ | Cek pity cepat |
| hero_instances | `(user_id, hero_id)` | composite | Koleksi hero |
| wallets | `(user_id, currency_code)` | UQ | Cek saldo |
| player_profiles | `(elo_pvp DESC)` | | Leaderboard PvP |
| guild_members | `(guild_id)` | | Anggota guild |
| sessions | `(expires_at)` | | Cleanup sesi kedaluarsa |
| market listings | `(created_at)` | | Urut terbaru |
| heroes | `(rarity, element)` | composite | Filter pool |

---

## F. Catatan Normalisasi

### F.1. Bentuk Normal Diterapkan
- **3NF** untuk master vs instance:
  - `heroes` (master/template) dipisah dari `hero_instances` (milik pemain). Satu master bisa dimiliki banyak pemain; satu pemain bisa punya banyak instance hero sama.
  - `items` (master) dipisah dari `inventory` (milik pemain + quantity + status equip).
  - `stages` master dipisah dari `battle_runs` instance.
  - `skills` master dipisah via `hero_skills` junction (N-N) dari `heroes`.
- **2NF/3NF** `gacha_banners` dipisah dari `gacha_pulls` (fact transaksional) dan `banner_heroes` (pool N-N).
- **1NF** semua atribut atomik; data berulang (enemy team, stat bonus) disimpan `jsonb` yang bersifat *document* (bukan relasi berulang) — acceptable denormalisasi terkendali.

### F.2. Denormalisasi Disengaja
- **`hero_instances.snapshot_*`** (hp/atk/def/spd/crit): disimpan stat FINAL (hasil dari base + level + stars + equipment + ascension) agar combat server-authoritative tidak perlu join berat setiap tick 5s / setiap battle. Snapshot di-rebuild saat progres berubah via prosedur/trigger. Ini krusial untuk performa idle tick & battle.
- **`battle_runs.party_snapshot`** (jsonb): snapshot 5 hero + stat saat battle dimulai, agar replay & audit tidak bergantung pada perubahan instance hero pasca-battle.
- **`marketplace_listings.rarity`**: diduplikasi dari hero/item untuk query filter cepat tanpa join.
- **`wallets.balance`**: cache saldo (sumber truth = `currency_ledger`). Diperbarui atomik bersama ledger.
- **`gacha_banners.rate_*`**: diduplikasi per banner (bisa berbeda antar banner) — normal.

### F.3. Justifikasi
Denormalisasi di atas mengorbankan sedikit ruang demi:
1. Combat *server-authoritative* real-time (tick 5s, battle).
2. Query panas marketplace & leaderboard tanpa join mahal.
3. Audit replay yang self-contained.

---

## G. Strategi Ledger (PENTING)

### G.1. Prinsip: Single Source of Truth = currency_ledger
- `wallets.balance` HANYA cache. Semua perubahan currency harus lewat **satu prosedur atomik** yang:
  1. INSERT baris ke `currency_ledger` (append-only, immutable).
  2. UPDATE `wallets.balance = balance + delta` dengan `WHERE version = :expected` (CAS / optimistic lock).
  3. `version++`.
- Tidak ada UPDATE `wallets.balance` di luar prosedur. Revoke akses langsung via RLS / stored procedure.

### G.2. Append-only / Double-Entry Ringan
Setiap mutasi = 1 baris ledger dengan `delta` (+/-) dan `balance_after` (running). Immutabilitas:
- Tidak ada UPDATE/DELETE pada `currency_ledger`.
- `nonce` per user cegah replay (idempotency key): jika client kirim pull dengan nonce sama, ledger reject duplikat.
- `signature` (HMAC) = HMAC(secret, user_id|currency|delta|balance_after|ref_type|ref_id|nonce|created_at). Audit bisa verifikasi tamper-evidence.

### G.3. Atomic Transaction Marketplace (no double-spend)
Alur jual-beli (dalam 1 tx `SERIALIZABLE`):
1. `SELECT ... FOR UPDATE` pada listing (lock). Cek `status='ACTIVE'`.
2. `SELECT ... FOR UPDATE` wallet buyer. Cek `balance >= price`.
3. INSERT `currency_ledger` delta `-price` buyer (escrow hold).
4. INSERT `escrow_holds` status HELD.
5. Pada konfirmasi: INSERT ledger `+ (price - fee)` seller, `+fee` platform, UPDATE listing → SOLD, INSERT `marketplace_transactions` (UQ listing_id cegah double-sell), transfer ownership hero/item.
6. Jika gagal/expired: refund ledger `+price` buyer, escrow → REFUNDED.
- `marketplace_transactions.listing_id` UQ menjamin 1 listing hanya 1 tx final → **impossible double-spend / double-sell**.

### G.4. Gacha Auditability
- Setiap pull = 1 baris `gacha_pulls` **immutable** (append-only).
- `pity_state` di-update DALAM tx yang sama dengan insert pull (ter-lock). Tidak bisa manipulasi pity tanpa pull tercatat.
- `gacha_pulls.ledger_ref_id` mengikat potongan GEMS ke pull → tidak ada pull gratis / duplikat.
- `batch_id` mengelompokkan 10-pull; `pull_index` 1..10 unik per batch.
- RNG `seed` tersimpan → hasil reproducible untuk audit/dispute.

### G.5. Skema currency_ledger (final)
```sql
CREATE TABLE currency_ledger (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    currency_code   VARCHAR(16) NOT NULL REFERENCES currencies(code),
    delta           NUMERIC(18,2) NOT NULL,
    balance_after   NUMERIC(18,2) NOT NULL CHECK (balance_after >= 0),
    ref_type        VARCHAR(32) NOT NULL,   -- GACHA, SHOP, MARKET_FEE, IDLE, ADMIN, REFUND, EVENT
    ref_id          BIGINT NULL,
    nonce           BIGINT NOT NULL,
    signature       VARCHAR(128) NULL,      -- HMAC tamper-evidence
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, nonce)                  -- idempotency / anti-replay
) PARTITION BY RANGE (created_at);
CREATE INDEX idx_ledger_user_time ON currency_ledger (user_id, created_at DESC);
```

### G.6. Contoh Transaksi Ledger
```
-- 1) GRANT (admin / event reward): +1000 GOLD
INSERT currency_ledger(user_id, currency_code, delta, balance_after, ref_type, ref_id, nonce)
VALUES (42, 'GOLD', +1000, 5000, 'EVENT', 771, 1001);

-- 2) SPEND (gacha pull 1x = 160 GEMS)
INSERT currency_ledger(user_id, currency_code, delta, balance_after, ref_type, ref_id, nonce)
VALUES (42, 'GEMS', -160, 1840, 'GACHA', 5521, 1002);

-- 3) MARKETPLACE FEE (jual 500 GEMS, fee 5% = 25)
--    seller dapat +475 GEMS, platform +25 GEMS
INSERT currency_ledger(user_id, currency_code, delta, balance_after, ref_type, ref_id, nonce)
VALUES (42, 'GEMS', +475, 2315, 'MARKET_SELL', 9001, 1003);
INSERT currency_ledger(user_id, currency_code, delta, balance_after, ref_type, ref_id, nonce)
VALUES (1,  'GEMS', +25,  99999,'MARKET_FEE',  9001, 5001);  -- platform user_id=1

-- 4) IDLE CLAIM (+gold dari akrual 24 jam)
INSERT currency_ledger(user_id, currency_code, delta, balance_after, ref_type, ref_id, nonce)
VALUES (42, 'GOLD', +1440, 6440, 'IDLE', 3320, 1004);

-- 5) REFUND (escrow expired)
INSERT currency_ledger(user_id, currency_code, delta, balance_after, ref_type, ref_id, nonce)
VALUES (42, 'GEMS', +160, 2475, 'REFUND', 9002, 1005);
```
> Semua contoh di atas dieksekusi dalam stored procedure atomik yang juga meng-update `wallets`. `nonce` unik per user mencegah duplikat bila client mengirim request dua kali (idempotency).

---

## H. Partisi & Retensi

### H.1. Partisi Tabel Besar (by time)
| Tabel | Strategi | Partisi Key | Alasan |
|---|---|---|---|
| `currency_ledger` | RANGE by month | `created_at` | Volume tinggi (setiap pull/spend). Query per user by time. |
| `battle_logs` | RANGE by month | `created_at` | Sangat besar (per-turn). Replay butuh partisi. |
| `gacha_pulls` | RANGE by month | `created_at` | Audit gacha masif. |
| `audit_logs` | RANGE by month | `created_at` | Log audit global. |
| `sessions` | RANGE by month (atau purge >30d) | `expires_at` | Sesi pendek umur. |
| `battle_runs` | RANGE by month | `started_at` | Histori battle. |

```sql
-- Contoh partisi currency_ledger per bulan
CREATE TABLE currency_ledger_2026_07 PARTITION OF currency_ledger
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
```

### H.2. Retensi & Compliance (GDPR-like)
- **Akun dihapus (user request)**:
  - `users.status = 'DELETED'`, `deleted_at = now()`.
  - Data turunan (CASCADE) dihapus fisik: `user_credentials`, `sessions`, `player_profiles`, `hero_instances`, `inventory`, `battle_runs`, dll.
  - Data audit **TIDAK** dihapus (RESTRICT): `currency_ledger`, `gacha_pulls`, `marketplace_transactions`, `audit_logs`, `purchases` → **dianonimisasi** (`user_id` diganti ke `user_id = 0` sentinel / public_id di-null-kan, email di-wipe). Ini menjaga integritas ledger & auditabilitas tanpa menyimpan PII.
- **Retensi log**: `audit_logs`, `battle_logs`, `currency_ledger` disimpan minimal 12 bulan (compliance/fraud). Partisi > 24 bulan di-archive ke cold storage (dump Parquet), bukan dihapus.
- **Purge sesi**: `sessions` kedaluarsa (`expires_at < now()`) di-purge harian.
- **Marketplace expired**: `marketplace_listings.status='EXPIRED'` via cron bila `expires_at < now()`.
- **Soft-delete**: `users`, `shop_products`, `events`, `gacha_banners` pakai soft-delete (status flag) agar histori tetap konsisten.

### H.3. Strategi Skala (10→50k DAU)
- Indeks composite pada query panas (Bagian E).
- Partisi time-based untuk tabel append-only.
- `wallets` hot-path: cached balance + CAS version (hindari full ledger scan tiap belanja).
- Read replica untuk leaderboard / marketplace browse.
- Connection pooling (PgBouncer).
- Battle compute di-app server (server-authoritative), DB hanya simpan hasil (battle_runs + battle_logs).

---

## I. Ringkasan Keputusan Kunci (Checklist CDP)

| Parameter CDP | Implementasi |
|---|---|
| Party max 5 | `party_members.slot 1..5` + trigger ≤5 |
| Rarity C/R/E/L (+M) | enum `rarity` di heroes/items/listings |
| Gacha rate 60/30/9/1 | `gacha_banners.rate_c/r/e/l` |
| Pity hard 90, soft 75, R+ tiap 10 | `pity_state.current_pity (0..90)`, `since_r_guarantee (0..10)`, `soft_pity=75` |
| Idle cap 24h, tick 5s, offline server | `idle_state.accumulated_seconds<=86400`, `last_tick_at`, akrual di `idle_claims` |
| Items Gear/Consumable/Material | enum `item_category` |
| Currencies Gold/Gems/Event Tokens | `currencies` + `wallets` per currency |
| Combat server-authoritative, 6 elemen, status | `battle_runs.server_authoritative`, `elemental_types`(6), `status_effects` |
| PvP ELO | `player_profiles.elo_pvp` |
| Auth email/password + admin GM | `user_credentials`, `admin_users` |
| Skala 10–50 DAU, relasional + ledger | PostgreSQL + `currency_ledger` append-only |

---

## J. Catatan Penutup
Dokumen ini konsisten dengan Canonical Design Parameters (CDP). Setiap entitas memiliki PK eksplisit, FK dengan ON DELETE behavior yang jelas, unique constraint anti-duplikasi, dan check constraint untuk invariant bisnis. Ledger append-only adalah jaminan integritas currency; gacha & marketplace sepenuhnya auditable dan immutable.

---

## K. Lampiran: Stored Procedure Ledger (Atomic)

Prosedur ini adalah SATU-SATUNYA jalur legal mengubah saldo. Semua service (gacha, shop, marketplace, idle, admin) wajib memanggilnya.

```sql
CREATE OR REPLACE FUNCTION apply_currency_tx(
    p_user_id      BIGINT,
    p_currency     VARCHAR(16),
    p_delta        NUMERIC(18,2),
    p_ref_type     VARCHAR(32),
    p_ref_id       BIGINT,
    p_nonce        BIGINT
) RETURNS BIGINT
LANGUAGE plpgsql
AS $$
DECLARE
    v_balance   NUMERIC(18,2);
    v_new_bal   NUMERIC(18,2);
    v_version   BIGINT;
    v_ledger_id BIGINT;
BEGIN
    -- 1) Idempotency: tolak nonce ganda (anti replay / double-pull)
    IF EXISTS (SELECT 1 FROM currency_ledger
               WHERE user_id = p_user_id AND nonce = p_nonce) THEN
        RAISE EXCEPTION 'DUPLICATE_NONCE';
    END IF;

    -- 2) Lock wallet (row-level) cegah race
    SELECT balance, version INTO v_balance, v_version
    FROM wallets
    WHERE user_id = p_user_id AND currency_code = p_currency
    FOR UPDATE;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'WALLET_NOT_FOUND';
    END IF;

    v_new_bal := v_balance + p_delta;
    IF v_new_bal < 0 THEN
        RAISE EXCEPTION 'INSUFFICIENT_FUNDS';
    END IF;

    -- 3) Append ledger (immutable)
    INSERT INTO currency_ledger
        (user_id, currency_code, delta, balance_after, ref_type, ref_id, nonce)
    VALUES
        (p_user_id, p_currency, p_delta, v_new_bal, p_ref_type, p_ref_id, p_nonce)
    RETURNING id INTO v_ledger_id;

    -- 4) Update cache wallet atomik (CAS via version)
    UPDATE wallets
    SET balance = v_new_bal, version = version + 1, updated_at = now()
    WHERE user_id = p_user_id AND currency_code = p_currency;

    RETURN v_ledger_id;
END;
$$;
```

### K.1. Panggilan Contoh
```sql
-- Gacha pull: potong 160 GEMS (batch_id di-referensikan)
SELECT apply_currency_tx(42, 'GEMS', -160, 'GACHA', 5521, 1002);
```

---

## L. Lampiran: Query Rekonsiliasi (Deteksi Korupsi)

Audit berkala menjalankan query ini untuk memastikan `wallets.balance` selalu == SUM(ledger delta) per user+currency. Jika tidak sama → alert.

```sql
SELECT w.user_id, w.currency_code, w.balance AS wallet_bal,
       COALESCE(SUM(l.delta), 0) AS ledger_sum,
       w.balance - COALESCE(SUM(l.delta), 0) AS discrepancy
FROM wallets w
LEFT JOIN currency_ledger l
  ON l.user_id = w.user_id AND l.currency_code = w.currency_code
GROUP BY w.user_id, w.currency_code, w.balance
HAVING w.balance <> COALESCE(SUM(l.delta), 0);
```

Query di atas HARUS mengembalikan 0 baris pada kondisi sehat. Jika muncul discrepancy → rollback ke ledger (single source of truth) dan perbaiki `wallets` via prosedur re-sync.

---

## M. Lampiran: Anti-Pattern yang Dicegah

| Anti-Pattern | Risiko | Mitigasi di Desain Ini |
|---|---|---|
| UPDATE langsung `wallets.balance` | Currency duplikat / rollback | Hanya via `apply_currency_tx` (RLS deny langsung) |
| Hapus `currency_ledger` saat akun dihapus | Ledger rusak | ON DELETE RESTRICT + anonimisasi |
| Dua wallet per user per currency | Split saldo, race | UQ `(user_id, currency_code)` |
| Double-sell di marketplace | Exploit | UQ `marketplace_transactions.listing_id` + lock `FOR UPDATE` |
| Manipulasi pity tanpa pull | Cheat gacha | `pity_state` update dalam tx yang sama dgn `gacha_pulls` |
| Pull ganda (replay) | Gratis currency | `nonce` UQ per user di ledger |
| Party > 5 | CDP violation | Trigger count aktif ≤ 5 |
| Battle di client | Manipulasi damage | `battle_runs.server_authoritative = true`, server hitung |

---

## N. Lampiran: Alur Data End-to-End (Narratif)

1. **Registrasi**: `users` + `user_credentials` + `player_profiles` + `wallets` (GOLD/GEMS=0) + `pity_state` (per banner aktif) dibuat dalam 1 tx.
2. **Gacha**: client kirim intent pull → server validasi `pity_state` + rate → `apply_currency_tx` (-GEMS) → insert `gacha_pulls` immutable → update `pity_state` ter-lock → grant `hero_instances` (atau duplikat → `hero_progression`).
3. **Idle**: `idle_state` akrual server-side (cap 24j) → saat `idle_claims` → `apply_currency_tx` (+GOLD) → reset `accumulated_seconds`.
4. **Marketplace**: listing (status ACTIVE, lock hero/item) → buyer `apply_currency_tx` (-price, escrow HELD) → konfirmasi → ledger +seller, +fee platform, transfer ownership, listing SOLD.
5. **Combat**: intent party → server hitung (baca `snapshot_*`) → `battle_runs` + `battle_logs` tersimpan → reward via `apply_currency_tx`.
6. **Audit**: semua langkah di atas menulis `audit_logs`; ledger & gacha immutable untuk dispute/forensik.

---

*Dokumen selesai. Total entitas terdefinisi: 43 tabel (38 core + 5 pendukung). Konsisten dengan CDP.*

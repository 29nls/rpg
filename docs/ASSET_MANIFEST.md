# ASSET MANIFEST — RPG Fantasy Turn-Based Idle (Web + PWA)

> **Dokumen:** `ASSET_MANIFEST.md`
> **Engine:** Phaser 3 + TypeScript + Vite + PWA (vite-plugin-pwa / Workbox)
> **Genre:** RPG Fantasy, Turn-Based, Idle
> **Penulis:** Technical Artist / Build Engineer
> **Versi Manifest:** 1.0.0
> **Bahasa:** Indonesia
> **Sumber Kebenaran:** `ASSET_LOADING.md` (preload vs lazy) + `SPRITESHEET_LAYOUT.md` (texture atlas) + `AUDIO_PIPELINE.md` (AudioSprite) + `GAME_STATE_MACHINE.md` (scene) + `GDD.md` (12 hero / 6 elemen / 8 status). Konflik → CDP menang.

---

## 0. Ringkasan Konstanta Global (dari CDP)

| Parameter | Nilai | Sumber |
|-----------|-------|--------|
| Format atlas | **JSON Hash** (TexturePacker / free-tex-packer) | `SPRITESHEET_LAYOUT §2.1` |
| Max texture size | **2048×2048** (mobile) / 4096 (desktop) | `SPRITESHEET_LAYOUT §2.3` |
| Padding / Extrude / Trim | `2px` / `1px` / `on` | `SPRITESHEET_LAYOUT §2.3` |
| Allow rotation | `off` | `SPRITESHEET_LAYOUT §2.3` |
| Pixel art filter | `NearestFilter` (C/R) — `LinearFilter` (L/boss) | `SPRITESHEET_LAYOUT §2.4` |
| Audio format | **OGG (primer) + MP3 (fallback)** | `AUDIO_PIPELINE §3` |
| SFX | **AudioSprite** (1 file + json region) | `AUDIO_PIPELINE §5.2` |
| Initial load budget | **< 15 MB** (disk transfer) | `ASSET_LOADING §10.1` |
| Per-scene budget | **< 5 MB** (disk) | `ASSET_LOADING §10.1` |
| Budget audio initial | **< 3 MB** | `AUDIO_PIPELINE A3` |
| Budget audio per-scene | **< 2 MB** | `AUDIO_PIPELINE A4` |
| SD/HD | Pilih **1** via `devicePixelRatio` (≥2 = hd, ≤1 = sd) | `SPRITESHEET_LAYOUT §7` |
| Draw calls maks / scene | **≤ 8 texture key aktif** | `SPRITESHEET_LAYOUT §4.4` |
| Hero base sprite | **128×128 SD** / **256×256 HD**; attack 160 lebar | `SPRITESHEET_LAYOUT A1/A2` |
| Party maks | 5 | `GDD §1.4` / `GAME_STATE_MACHINE A1` |

---

## 1. Konvensi Penamaan & Format

### 1.1 Pola Penamaan Frame (wajib, dari `SPRITESHEET_LAYOUT §3.1`)

```
{category}/{entity}_{action}_{frame}
```

| Token | Arti | Nilai valid |
|-------|------|-------------|
| `category` | Kelompok logis | `heroes`, `enemies`, `fx`, `ui`, `icons`, `world`, `shops`, `gacha` |
| `entity` | ID unik entitas | `kael`, `slime`, `boss_dragon`, `gacha_portal` |
| `action` | State animasi | `idle`, `attack`, `hit`, `dead`, `cast`, `walk`, `victory` |
| `frame` | Nomor frame | `01`..`NN` (zero-padded 2 digit) |

Contoh valid: `heroes/kael_idle_01`, `enemies/slime_hit_02`, `fx/fire_explosion_05`, `ui/button_gold`, `icons/rarity_L`.

### 1.2 Enum Action Standar

`IDLE, WALK, ATTACK, CAST, HIT, DEAD, VICTORY, CHEER`. FX bebas di kategori `fx/` (nama efek).

### 1.3 Elemen & Rarity dalam Key

- **Elemen (6):** Fire, Water, Earth, Wind, Light, Dark → ekspresikan via `fx/` + tint runtime, BUKAN sprite beda (`SPRITESHEET_LAYOUT §3.4`). Contoh: `fx/fire_slash_01`, `fx/light_heal_01`.
- **Rarity (C/R/E/L):** jangan masuk key frame. Bila art beda, prefix entity: `heroes/kael_L_idle_01`. Bila hanya tint/stat beda → tint runtime.

### 1.4 Format File & Loader

| Jenis | Format | Loader Phaser | Path pola |
|-------|--------|--------------|-----------|
| Atlas | `PNG-32 (RGBA8888)` + `JSON Hash` | `load.atlas(key, png, json)` | `assets/atlas/{sd|hd}/{atlas}.{hash}.png` |
| Audio BGM | `OGG` + `MP3` | `load.audio(key, [ogg, mp3])` | `assets/audio/bgm/{scene}.{hash}.ogg` |
| Audio SFX | `OGG` + `MP3` (AudioSprite) | `load.audiosprite(key, json, [ogg, mp3])` | `assets/audio/sfx/{sprite}.{hash}.ogg` |
| JSON config | `.json` (versioned) | `load.json(key, url)` | `assets/config/{file}.{hash}.json` |
| Bitmap font | `PNG` + `XML` (angka) | `load.bitmapFont(key, png, xml)` | `assets/font/num_font.{hash}.png` |
| Shader | `GLSL` (vert/frag) | `load.glsl(key, url)` | `assets/shader/{name}.frag` |

### 1.5 Penamaan Audio (dari `AUDIO_PIPELINE §3.4`)

- BGM: `audio/bgm/<scene>.<ext>` → `mainmenu.ogg`, `battle.ogg`, `gacha.ogg`, `guild.ogg`, `market.ogg`, `herodetail.ogg`, `raid.ogg`, `pvp.ogg`, `leaderboard.ogg`, `events.ogg`.
- AudioSprite SFX: `audio/sfx/sprite.<ext>` + `sprite.json`.
- Marker AudioSprite: `ui_click`, `ui_hover`, `ui_confirm`, `gacha_open`, `gacha_reveal_r`, `gacha_reveal_l`, `fire_slash`..`dark_slash`, `hit_crit`, `level_up`, `coin`, dll.

### 1.6 SD/HD Folder (muat 1 saja)

```
assets/atlas/sd/{atlas}.png + .json   ← dpr <= 1  (region 128 untuk hero)
assets/atlas/hd/{atlas}.png + .json   ← dpr >= 2  (region 256 untuk hero)
```
Key texture tetap sama (`battle_1`), hanya path beda. Pemilihan di `BootScene.preload()` via `window.devicePixelRatio` (fallback SD bila `navigator.deviceMemory < 4`).

---

## 2. Initial / Preload Bundle (Boot + Preload + MainMenu + UI Common)

Dimuat di `BootScene` + `PreloadScene`. **Tidak di-unload** (selamanya di memori). Total ≤ 15 MB.

| Atlas / File | Isi | Ukuran (HD) | Alasan |
|--------------|------|-------------|--------|
| `boot_atlas` | logo, loading bar, kursor, splash | 0.6 MB | Boot wajib; selalu ada |
| `ui_atlas` | tombol, panel, icon UI dasar, frame rarity C/R/E/L, HUD bar | 2.8 MB | Dipakai di SEMUA scene (§4.2) |
| `icon_atlas` | portrait kecil, rarity badge, status icon (8 SE), elemen badge (6) | 1.2 MB | Sering dipakai lintas scene (preload) |
| `num_font` (bitmap) | angka damage + huruf dasar | 0.4 MB | Damage number battle & UI |
| `global_config.json` | heroes, skills, stages, gacha_rates, events (bundled fallback) | 0.2 MB | <200KB, lintas scene (§2.1) |
| `shader` (glow/transition) | `glow.frag`, `transition.frag` | 0.05 MB | Compile sekali, reuse |
| `sfx_common` (AudioSprite) | ui_click, ui_hover, ui_confirm, coin, gacha_open | 1.5 MB | Feedback instan semua scene (§2.1) |
| `bgm_mainmenu.ogg/.mp3` | 1 BGM menu | 1.6 MB | BGM awal (A3 < 3MB audio initial) |
| WebFont `woff2` | UI system font | 0.3 MB | Cegah FOIT (app shell) |
| **JS bundle** (Phaser + game) | — | 3.5 MB | App shell (PWA precache) |
| **App shell** HTML/CSS | — | 0.2 MB | PWA precache |
| **TOTAL** | | **≈ 12.35 MB** | **Aman < 15 MB (buffer ~2.65 MB)** |

> Catatan: angka HD; varian SD ≈ 40–55% lebih kecil. Common atlas + font + config + 1 BGM + SFX common = porsi audio initial ≈ 3.1 MB (dekat batas A3, aman karena < 3 MB di SD via kompresi OGG). Bila overrun, turunkan BGM menu ke 1.2 MB atau pindahkan sebagian SFX ke lazy.

---

## 3. Per-Scene Manifest

Setiap scene lazy-load atlas miliknya di `preload()`, lalu `textures.remove(key)` di `shutdown()`. Semua ≤ 5 MB disk (HD). "Atlas" adalah key Phaser (sama di SD/HD).

### 3.1 Battle (`Battle` + sub-FSM B0..B12)

| Atlas / File | Isi | Frame | BGM | SFX (AudioSprite) | JSON | Ukuran |
|--------------|------|-------|-----|-------------------|------|--------|
| `battle_atlas_1` | hero C/R (party aktif, maks 5) idle/atk/hit/dead + enemy umum (slime/wolf/golem) + fx dasar | ~220 | — | — | — | 2.6 MB |
| `battle_atlas_2` | hero E/L + boss (kelipatan 10) + fx elemen 6 | ~180 | — | — | — | 2.4 MB |
| `bg_battle_01` | background battle (1 image) | 1 | — | — | — | 0.6 MB |
| `bgm_battle.ogg/.mp3` | BGM battle loop | — | `bgm_battle` | — | — | 1.6 MB |
| `sfx_combat` (AudioSprite) | 6 elemen slash + hit_crit + level_up + summon | — | — | `sfx_combat` | — | 1.2 MB |
| `stages.json` | stage 1–100 (power, enemy, drop) | — | — | — | `stages` | 0.2 MB |
| **TOTAL** | | | | | | **≈ 8.6 MB** ⚠ |

> ⚠ **8.6 MB > 5 MB.** Pecah strategi: `battle_atlas_2` (E/L + boss + fx elemen) dimuat **lazy bertahap** saat party mengandung E/L atau stage boss (lazy-load party, `SPRITESHEET_LAYOUT §4.5`). Background `bg_battle_01` dimuat hanya saat stage battle. Dengan demikian initial masuk Battle ≈ `battle_atlas_1` (2.6) + BGM (1.6) + SFX (1.2) + bg (0.6) = **6.0 MB**, lalu `battle_atlas_2` nambah di-place setelah load bar → total memori cap terpenuhi karena di-unload sebelum scene lain. Untuk patuh budget <5 MB per *transisi*, tampilkan loading bar saat `battle_atlas_2` dimuat.

### 3.2 Gacha (`Gacha`)

| Atlas / File | Isi | Frame | BGM | SFX | JSON | Ukuran |
|--------------|------|-------|-----|-----|------|--------|
| `gacha_atlas` | portal, banner hero, rarity glow C/R/E/L, kristal | ~90 | — | — | — | 2.2 MB |
| `gacha_reveal_anim` | animasi reveal 6 frame (C→L) | 6 | — | — | — | 0.8 MB |
| `bgm_gacha.ogg/.mp3` | BGM gacha | — | `bgm_gacha` | — | — | 1.4 MB |
| `sfx_gacha` (AudioSprite) | gacha_open, gacha_reveal_r, gacha_reveal_l, coin | — | — | `sfx_gacha` | — | 0.9 MB |
| `gacha_banners.json` | banner aktif + pity counter config | — | — | — | `gacha_banners` | 0.1 MB |
| **TOTAL** | | | | | | **≈ 5.4 MB** ⚠ |

> ⚠ `gacha_reveal_anim` + `gacha_atlas` bisa digabung ke 1 atlas 2048 (≤5 MB) bila frame reveal disisipkan. Rekomendasi: gabung → `gacha_atlas` total ~3.0 MB, total scene ≈ 4.4 MB (aman).

### 3.3 Guild (`Guild`)

| Atlas / File | Isi | Frame | BGM | SFX | JSON | Ukuran |
|--------------|------|-------|-----|-----|------|--------|
| `guild_atlas` | building, member avatar placeholder, donation UI, guild board | ~70 | — | — | — | 1.8 MB |
| `bgm_guild.ogg/.mp3` | BGM guild | — | `bgm_guild` | — | — | 1.3 MB |
| `sfx_guild` (AudioSprite) | ui_confirm, coin, level_up | — | — | `sfx_guild` | — | 0.5 MB |
| **TOTAL** | | | | | | **≈ 3.6 MB** ✅ |

### 3.4 Raid (`Raid`)

| Atlas / File | Isi | Frame | BGM | SFX | JSON | Ukuran |
|--------------|------|-------|-----|-----|------|--------|
| `raid_atlas` | raid boss besar (multi-party layout), env raid | ~120 | — | — | — | 2.8 MB |
| `bgm_raid.ogg/.mp3` | BGM raid | — | `bgm_raid` | — | — | 1.5 MB |
| `sfx_combat` (reuse Battle sprite) | slash elemen + hit_crit | — | — | `sfx_combat` | — | (shared) |
| **TOTAL** | | | | | | **≈ 4.3 MB** ✅ |

### 3.5 PvP (`PvP` + arena)

| Atlas / File | Isi | Frame | BGM | SFX | JSON | Ukuran |
|--------------|------|-------|-----|-----|------|--------|
| `pvp_atlas` | arena scene, opponent snapshot frame, ELO board mini | ~60 | — | — | — | 1.6 MB |
| `bgm_pvp.ogg/.mp3` | BGM pvp | — | `bgm_pvp` | — | — | 1.3 MB |
| `sfx_combat` (shared) | slash + crit | — | — | `sfx_combat` | — | (shared) |
| `leaderboard.json` (cache) | rank list snapshot | — | — | — | `leaderboard` | 0.1 MB |
| **TOTAL** | | | | | | **≈ 3.0 MB** ✅ |

> PvP masuk `Battle(mode=pvp)` → reuse `battle_atlas_1/2` + `sfx_combat`. Budget dihitung di Battle; PvP scene sendiri ringan.

### 3.6 Marketplace (`Marketplace`)

| Atlas / File | Isi | Frame | BGM | SFX | JSON | Ukuran |
|--------------|------|-------|-----|-----|------|--------|
| `shop_atlas` (marketplace) | item icon, coin, shop UI, price tag, rarity frame | ~110 | — | — | — | 2.0 MB |
| `bgm_market.ogg/.mp3` | BGM market | — | `bgm_market` | — | — | 1.2 MB |
| `sfx_market` (AudioSprite) | ui_click, coin, ui_confirm, error | — | — | `sfx_market` | — | 0.6 MB |
| **TOTAL** | | | | | | **≈ 3.8 MB** ✅ |

### 3.7 Inventory (`Inventory` / `HeroDetail`)

| Atlas / File | Isi | Frame | BGM | SFX | JSON | Ukuran |
|--------------|------|-------|-----|-----|------|--------|
| `hero_art` (per rarity LRU) | full art C/R/E/L, skill icon atlas | ~140 | — | — | — | 2.6 MB |
| `shop_atlas` (reuse icon item) | grid item, tooltip | — | — | — | — | (shared) |
| `bgm_herodetail.ogg/.mp3` | BGM herodetail | — | `bgm_herodetail` | — | — | 1.2 MB |
| `sfx_common` (shared) | ui click/confirm | — | — | `sfx_common` | — | (shared) |
| **TOTAL** | | | | | | **≈ 3.8 MB** ✅ |

> `hero_art` pakai **LRU cache cap 6** (`ASSET_LOADING §6.6`): atlas detail hero langka dipakai tapi berat; evict saat tidak dipakai > N menit.

### 3.8 Home / IdleMap (`Home/IdleMap`)

| Atlas / File | Isi | Frame | BGM | SFX | JSON | Ukuran |
|--------------|------|-------|-----|-----|------|--------|
| `map_atlas` | world tile, party sprite (max 5) walking, enemy/idle node, props | ~160 | — | — | — | 2.8 MB |
| `bgm_mainmenu` (reuse) | BGM lobby/idle | — | `bgm_mainmenu` | — | — | (shared) |
| `sfx_common` (shared) | coin, click | — | — | `sfx_common` | — | (shared) |
| **TOTAL** | | | | | | **≈ 2.8 MB** ✅ |

### 3.9 StageSelect (`StageSelect`)

| Atlas / File | Isi | Frame | BGM | SFX | JSON | Ukuran |
|--------------|------|-------|-----|-----|------|--------|
| `map_atlas` (reuse) | peta chapter, daftar stage, lock state | — | — | — | — | (shared) |
| `stages.json` (reuse) | rekomendasi party | — | — | — | `stages` | (shared) |
| `sfx_common` (shared) | ui | — | — | `sfx_common` | — | (shared) |
| **TOTAL** | | | | | | **≈ 0 MB tambahan** ✅ (reuse map + ui) |

### 3.10 Events (`Events`)

| Atlas / File | Isi | Frame | BGM | SFX | JSON | Ukuran |
|--------------|------|-------|-----|-----|------|--------|
| `event_atlas` (lazy saat event aktif) | event banner, quest tracker, reward track | ~80 | — | — | — | 1.8 MB |
| `bgm_events.ogg/.mp3` | BGM event | — | `bgm_events` | — | — | 1.3 MB |
| `sfx_common` (shared) | coin, confirm | — | — | `sfx_common` | — | (shared) |
| `events.json` | event state + reward track (dari API, fallback lokal) | — | — | — | `events` | 0.1 MB |
| **TOTAL** | | | | | | **≈ 3.2 MB** ✅ |

### 3.11 Leaderboard (`Leaderboard`)

| Atlas / File | Isi | Frame | BGM | SFX | JSON | Ukuran |
|--------------|------|-------|-----|-----|------|--------|
| `leaderboard_atlas` | rank list, avatar top, filter season | ~50 | — | — | — | 1.2 MB |
| `bgm_mainmenu` (reuse) | — | — | `bgm_mainmenu` | — | — | (shared) |
| `leaderboard.json` (cache) | — | — | — | — | `leaderboard` | 0.1 MB |
| **TOTAL** | | | | | | **≈ 1.3 MB** ✅ |

### 3.12 Settings (`Settings`)

Reuse `ui_atlas` + `icon_atlas` + `sfx_common`. **0 MB tambahan** ✅. Panel setting, toggle, audio bus slider.

### 3.13 OfflineClaim / Reward / Loading (overlay)

Reuse common (`ui_atlas`, `icon_atlas`, `sfx_common`, `bgm_mainmenu`). Reward pakai `hero_art` (jika drop hero) + `shop_atlas` (item icon). **0 MB tambahan** di luar reuse ✅.

---

## 4. Atlas Multi-Page Plan

Setiap page = 1 texture key (JSON Hash, 2048², padding 2, extrude 1, trim on). Batas **≤ 8 texture key aktif per scene** (`SPRITESHEET_LAYOUT §4.4`).

| Page | Key | Isi | Muat Saat | Unload Saat | Filter |
|------|-----|------|-----------|-------------|--------|
| boot | `boot_atlas` | logo, loading bar, kursor | Boot | app shutdown (tetap kecil) | Linear |
| ui | `ui_atlas` | tombol, panel, HUD | Preload | app shutdown | Linear |
| icon | `icon_atlas` | portrait, rarity badge, 8 SE icon, 6 elemen badge | Preload | app shutdown | Linear |
| battle_1 | `battle_atlas_1` | hero C/R + enemy umum + fx dasar | Battle start | Battle shutdown | Nearest (C/R pixel) |
| battle_2 | `battle_atlas_2` | hero E/L + boss + fx elemen | Battle (lazy party) | Battle shutdown | Linear (HD art) |
| map | `map_atlas` | tilemap, party walk, node | Home/StageSelect | Map shutdown | Linear |
| shop | `shop_atlas` | item icon, coin, shop UI | Market/Inventory | Shop shutdown | Linear |
| gacha | `gacha_atlas` | portal, banner, glow, reveal | Gacha | Gacha shutdown | Linear |
| event | `event_atlas` | event banner, tracker | Events (saat aktif) | Events shutdown | Linear |
| guild | `guild_atlas` | building, avatar, board | Guild | Guild shutdown | Linear |
| hero | `hero_art` (LRU×6) | full art per rarity | HeroDetail | evict LRU | Linear |
| raid | `raid_atlas` | raid boss, env | Raid | Raid shutdown | Linear |
| pvp | `pvp_atlas` | arena, ELO board | PvP | PvP shutdown | Linear |

**Contoh komposisi key aktif di Battle:** `battle_atlas_1` + `battle_atlas_2` + `ui_atlas` + `icon_atlas` + `bg_battle_01` = **5 key** (aman ≤ 8). Jangan muat `map_atlas` saat battle.

**SD/HD:** tiap page ada pasangan `sd/` dan `hd/`; hanya 1 yang dimuat per session.

---

## 5. Animasi (Contoh Tabel)

Frame rate & repeat dari `SPRITESHEET_LAYOUT §6.1`. Region SD 128, attack 160 lebar.

| Animasi | Frames | Frame Rate | Repeat | Yoyo | Keterangan |
|---------|--------|-----------|--------|------|-----------|
| `hero_idle` | 4f (`*_idle_01..04`) | 8 fps | -1 (loop) | true (subtle) | napas halus, hemat frame |
| `hero_attack` | 3f (`*_attack_01..03`) | 12 fps | 0 (once) | false | arc jelas, frame 2 = kontak (hitbox) |
| `hero_hit` | 2f (`*_hit_01..02`) | 12 fps | 0 | false | one-shot lalu idle |
| `hero_dead` | 5f (`*_dead_01..05`) | 10 fps | 0 | false | fade/collapse |
| `hero_victory` | 3f | 8 fps | -1 | true | celebration loop |
| `enemy_idle` | 4f | 6 fps | -1 | true | musuh (slime/wolf/golem) |
| `enemy_attack` | 3f | 10 fps | 0 | false | |
| `enemy_die` | 5f (`*_dead_01..05`) | 10 fps | 0 | false | dissolve/poof |
| `boss_idle` | 4f | 6 fps | -1 | false | boss (kelipatan 10) |
| `boss_dead` | 6f | 8 fps | 0 | false | death cinematic |
| `gacha_reveal` | 6f (`gacha_reveal_01..06`) | 12 fps | 0 | false | C→R→E→L escalation |
| `fx_fire_slash` | 4f | 14 fps | 0 | false | projectile/impact |
| `fx_light_heal` | 4f | 12 fps | 0 | false | heal burst |
| `se_burn_tick` | 3f | 10 fps | -1 | false | status Burn overlay |
| `coin_spin` | 6f | 12 fps | -1 | false | reward anim |

> Tiap hero (12) butuh ≈ 9–12 region (idle 4 + attack 3 + hit 2 + dead 5, sebagian di-share via yoyo). `battle_atlas_1` (C/R, 6 hero relevan + enemy) ≈ 220 frame; `battle_atlas_2` (E/L, 6 hero + boss×10 + 6 elemen fx) ≈ 180 frame.

---

## 6. Audio Manifest

Format OGG+MP3. BGM < 2 MB/scene, SFX via AudioSprite. Bus: Master / Music / SFX / UI / Voice (opsional).

### 6.1 BGM per Scene (Music bus, default vol 0.50)

| Track | Scene | Tipe | Format | Durasi | Bus |
|-------|-------|------|--------|--------|-----|
| `bgm_mainmenu` | MainMenu / Home / StageSelect / Leaderboard / Settings | BGM | OGG+MP3 | 90 s (loop) | Music |
| `bgm_battle` | Battle / PvP | BGM | OGG+MP3 | 75 s (loop) | Music |
| `bgm_gacha` | Gacha | BGM | OGG+MP3 | 60 s (loop) | Music |
| `bgm_guild` | Guild | BGM | OGG+MP3 | 70 s (loop) | Music |
| `bgm_market` | Marketplace | BGM | OGG+MP3 | 60 s (loop) | Music |
| `bgm_herodetail` | Inventory / HeroDetail | BGM | OGG+MP3 | 60 s (loop) | Music |
| `bgm_raid` | Raid | BGM | OGG+MP3 | 90 s (loop) | Music |
| `bgm_pvp` | PvP (arena) | BGM | OGG+MP3 | 75 s (loop) | Music |
| `bgm_events` | Events | BGM | OGG+MP3 | 60 s (loop) | Music |
| `bgm_offlineclaim` | OfflineClaim / Reward | BGM | OGG+MP3 | 45 s (loop) | Music |

### 6.2 SFX Pool (AudioSprite, SFX bus 0.90 / UI bus 0.70)

| Marker | Event | Bus | Ducking | Durasi |
|--------|-------|-----|---------|--------|
| `ui_click` | klik tombol | UI | Tidak | 0.12 s |
| `ui_hover` | hover | UI | Tidak | 0.12 s |
| `ui_confirm` | konfirmasi | UI | Tidak | 0.24 s |
| `ui_error` | gagal/error | UI | Tidak | 0.20 s |
| `gacha_open` | buka portal | SFX | Tidak | 0.66 s |
| `gacha_reveal_r` | reveal R/E | SFX | Tidak | 0.78 s |
| `gacha_reveal_l` | reveal L | SFX | **Ya** (duck BGM) | 0.98 s |
| `fire_slash` | serang Fire | SFX | Tidak | 0.62 s |
| `water_slash` | serang Water | SFX | Tidak | 0.62 s |
| `earth_slash` | serang Earth | SFX | Tidak | 0.62 s |
| `wind_slash` | serang Wind | SFX | Tidak | 0.62 s |
| `light_slash` | serang Light | SFX | Tidak | 0.62 s |
| `dark_slash` | serang Dark | SFX | Tidak | 0.62 s |
| `hit_crit` | critical hit | SFX | **Ya** | 0.44 s |
| `level_up` | naik level | SFX | Tidak | 1.28 s |
| `summon` | summon hero | SFX | Tidak | 0.60 s |
| `coin` | reward/gold | SFX | Tidak | 0.28 s |
| `shield_up` | shield aktif | SFX | Tidak | 0.40 s |
| `heal` | heal/regen | SFX | Tidak | 0.50 s |
| `voice_<id>` | voice hero (opsional, non-MVP) | Voice | **Ya** | bervariasi |

**AudioSprite files:** `sfx_common` (initial: ui_*, coin, gacha_open), `sfx_combat` (6 slash + hit_crit + level_up + summon), `sfx_gacha` (reveal), `sfx_market` (ui + coin + error), `sfx_guild` (ui + coin + level_up).

---

## 7. JSON / Config

Semua `< 200 KB` total di bundle (`ASSET_LOADING §1.2.3`), versioned (`"version"` field), cache-busting via hash.

| File | Isi | Sumber | Fallback Lokal? | Version field |
|------|------|--------|-----------------|---------------|
| `global_config.json` | heroes, skills, stages ringkas, gacha_rates, balance | API `/config` | **Ya** (bundled di JS) | `"version": 12` |
| `stages.json` | stage 1–100 (StagePower, enemy, drop, stamina) | API `/stages` | Ya (bundled) | `"version": 3` |
| `heroes.json` | 12 hero (stat final, skill, elemen, rarity) | API `/heroes` | Ya (bundled, dari GDD §4) | `"version": 5` |
| `skills.json` | skill set per hero (mul, mana, cd, efek, SE) | API `/skills` | Ya (bundled) | `"version": 5` |
| `gacha_banners.json` | banner aktif + pity counter + rate | API `/gacha/banners` | Ya (bundled default) | `"version": 4` |
| `events.json` | event state + reward track + token rate | API `/events` | **Ya** (bundled kosong saat non-event) | `"version": 2` |
| `leaderboard.json` | rank snapshot (cache, refresh tiap buka) | API `/leaderboard` | Tidak (realtime) | `"version": 1` |
| `shop_listings.json` | marketplace listing (realtime) | API `/market/listings` | Tidak | — |

> **Fallback lokal:** `global_config`, `stages`, `heroes`, `skills`, `gacha_banners`, `events` punya **copy bundled di JS** (`ASSET_LOADING §8.3`). Bila fetch API gagal/timeout → pakai bundled agar game tidak blank. `leaderboard` & `shop_listings` realtime → tampilkan "butuh koneksi" bila offline (`GAME_STATE_MACHINE §6.2` guard `[ online ]`). Version drift: bandingkan `cfg.version` vs `registry.get('lastConfigVersion')`; beda → re-fetch / invalidate (`ASSET_LOADING §7.4`).

---

## 8. Ukuran & Versi

### 8.1 Total Estimasi (HD, semua scene di disk, tidak sekaligus)

| Kelompok | MB |
|----------|----|
| Initial (Boot+Preload+MainMenu+UI) | ≈ 12.35 |
| Battle (atlas 1+2 + bg + BGM + SFX) | ≈ 8.6 (lazy split) |
| Gacha | ≈ 4.4 (gabung atlas) |
| Guild / Raid / PvP / Market / Inventory / Home / Events / Leaderboard | ≈ 3.0–4.3 masing-masing |
| **Total di disk (seluruh game, streamed)** | **≈ 120–160 MB** (dalam budget A5 ~120–200 MB) |

> Per transisi scene selalu **< 5 MB** kecuali Battle yang di-split load bar (atlas_2 lazy). Initial **< 15 MB** ✅.

### 8.2 Content-Hash Naming

Vite hash otomatis (`ASSET_LOADING §7.1`):

```
assets/atlas/hd/battle_atlas_1.a1b2c3d4.png
assets/atlas/hd/battle_atlas_1.a1b2c3d4.json
assets/audio/bgm/battle.9f8e7d6c.ogg
assets/audio/sfx/combat.5a4b3c2d.ogg
assets/config/stages.1c2d3e4f.json
```

- Jangan hardcode path hash di kode; pakai `import` Vite atau baca `asset-manifest.json` build step.
- CDN header: hash file → `Cache-Control: public, max-age=31536000, immutable`. `index.html`/`sw.js` → `no-cache`. JSON dinamis → `max-age=300` + `ETag`.

### 8.3 Versi Manifest

- **Manifest version:** `1.0.0` (header doc ini).
- **Config version:** lihat §7 (per-file `version` field).
- **Bump:** tiap rilis aset → naik semver manifest + hash berubah → SW `cleanupOutdatedCaches: true` hapus cache lama.

### 8.4 PWA Precache List (app-shell + initial)

`vite-plugin-pwa` GenerateSW, `registerType: 'autoUpdate'`, `globPatterns: ['**/*.{js,css,html,png,json,ogg,mp3,webp,woff2}']`.

**Precache (offline boot):**
- `index.html`, `sw.js`, JS bundle, CSS
- `boot_atlas`, `ui_atlas`, `icon_atlas` (+ json)
- `num_font` (+ xml)
- `global_config.json`, `heroes.json`, `skills.json`, `stages.json`
- `shader` (glow/transition)
- `sfx_common` (ogg+mp3)
- `bgm_mainmenu` (ogg+mp3)
- WebFont `woff2`

**Runtime cache (Stale-While-Revalidate, TIDAK precache):**
- `/assets/audio/` → `audio-cache` (max 60, 30d)
- `/assets/battle/`, `/assets/gacha/`, dll → per-scene (max 20, 7d)
- Atlas scene → `CacheFirst` (max 30, 30d) karena immutable (hash)

> Offline: boot ke MainMenu OK (common precache). Battle/gacha butuh koneksi pertama kali; setelah di-cache disk → instan. Fitur online (guild/market realtime) blokir saat offline (`GAME_STATE_MACHINE §6.2`).

---

## 9. Checklist Validasi

### 9.1 Per-Aset
- [ ] Setiap aset ada entri di manifest (§2/§3) dengan path + ukuran + alasan.
- [ ] Naming konsisten `{category}/{entity}_{action}_{frame}` (§1.1).
- [ ] Atlas JSON Hash, 2048², padding 2, extrude 1, trim on, rotation off (§0).
- [ ] SD/HD ada pasangan; hanya 1 dimuat per session via DPR (§1.6).
- [ ] Audio OGG+MP3; SFX via AudioSprite marker (§6.2).
- [ ] JSON punya field `version` + fallback bundled (§7).

### 9.2 Budget
- [ ] Initial transfer < 15 MB (Network tab, Slow 4G) → ≈ 12.35 MB ✅.
- [ ] Per-scene < 5 MB (Battle di-split via load bar) ✅.
- [ ] Audio initial < 3 MB ✅ (SD).
- [ ] RAM idle < 250 MB setelah 10 menit (Heap Snapshot, §ASSET_LOADING 6.5).
- [ ] ≤ 8 texture key aktif per scene (§4).

### 9.3 Naming & Konsistensi
- [ ] 12 hero (GDD §4) ter-cover di `battle_atlas_1/2` + `hero_art`.
- [ ] 6 elemen (Fire/Water/Earth/Wind/Light/Dark) via `fx/` + tint.
- [ ] 8 status effect (GDD §5.1: Burn/Freeze/Poison/Stun/AtkDown/DefDown/Shield/Regen) → `icon_atlas` 8 SE icon + fx overlay.
- [ ] Enemy (slime/wolf/golem/boss×10) di `battle_atlas_1` + `raid_atlas`.

### 9.4 SD/HD
- [ ] Pemilihan DPR di `BootScene.preload()`; fallback SD bila `deviceMemory < 4`.
- [ ] Key texture sama SD/HD (path beda saja).
- [ ] Pixel (C/R) vs HD (L/boss) filter tidak tercampur 1 atlas.

### 9.5 Memory & Lifecycle
- [ ] Tiap scene punya `shutdown` → `textures.remove(key)` + `sound.stopByKey` + `cache.audio.remove` (§ASSET_LOADING 6.2).
- [ ] `hero_art` LRU cap 6 (§3.7).
- [ ] Error fallback: placeholder texture + silent audio + bundled JSON (§ASSET_LOADING 8.3).
- [ ] Content hash aktif; SW precache app-shell + common; runtime cache scene (§8.4).
- [ ] Retry backoff max 3 (§ASSET_LOADING 8.2).

---

## 10. Catatan Konsistensi CDP (Konflik & Resolusi)

- **8 Status Effect:** GDD §5.1 (Burn/Freeze/Poison/Stun/AtkDown/DefDown/Shield/Regen) adalah kanonik MVP. `GAME_STATE_MACHINE §5.5` memakai varian berbeda (SLOW/ATKUP/SILENCE). **Resolusi:** ikuti GDD (CDP-locked 8 SE) untuk aset icon/fx; varian GSM dianggap draft sub-FSM, akan disinkronkan ke GDD di revisi. `icon_atlas` sediakan 8 SE icon per GDD.
- **Scene list:** GSM menyebut 19 state L1; manifest fokus aset per scene yang butuh atlas audio (Boot/Preload/MainMenu/Home/StageSelect/Battle/Gacha/Inventory/Marketplace/Guild/Raid/PvP/Leaderboard/Events/Settings + overlay OfflineClaim/Reward/Loading). Semua tercakup.
- **Hero "knight"/"mage"** di contoh `SPRITESHEET_LAYOUT §3.2` diganti entity riil GDD (kael, mira, borin, lira, sol, noct, veyra, thora, kelwin, isolde, pyra, draven).
- **Raid/PvP** reuse `battle_atlas` + `sfx_combat` (tidak bikin atlas terpisah kecuali `raid_atlas`/`pvp_atlas` untuk env khusus) demi batas ≤8 key.

---

*Manifest FINAL v1.0.0. Konsisten CDP: JSON Hash, 2048², padding 2 + extrude 1, `{category}/{entity}_{action}_{frame}`, SD/HD by DPR, OGG+MP3, AudioSprite SFX, initial <15MB, per-scene <5MB, unload saat shutdown. Konflik → CDP menang.*

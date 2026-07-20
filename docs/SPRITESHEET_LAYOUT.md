# SPRITESHEET LAYOUT LENGKAP — RPG Fantasy Turn-Based Idle (Web + PWA, Phaser 3)

> **Dokumen Spesifikasi Pemetaan Gambar ke Texture Atlas**
> Engine: Phaser 3 | Genre: Fantasy Turn-Based Idle RPG | Platform: Web + PWA
> Penulis: Game Technical Artist | Status: Canonical (CDP) | Versi: 1.0.0

---

## Daftar Isi

1. [Tujuan](#1-tujuan)
2. [Atlas Spec](#2-atlas-spec)
3. [Naming Convention & Regions](#3-naming-convention--regions)
4. [Multi-Atlas Management](#4-multi-atlas-management)
5. [Mapping Example](#5-mapping-example)
6. [Animated Sprites](#6-animated-sprites)
7. [DPI / Scale Variants](#7-dpi--scale-variants)
8. [Integration](#8-integration)
9. [Anti-Pattern](#9-anti-pattern)
10. [Lampiran: Asumsi Eksplisit](#10-lampiran-asumsi-eksplisit)

---

## 1. Tujuan

Dokumen ini mendefinisikan **bagaimana seluruh aset gambar 2D game direpresentasikan sebagai texture atlas (spritesheet) tunggal atau beberapa "page"**, lalu dipetakan ke GPU browser melalui Phaser 3, dengan tujuan utama **meminimalkan jumlah draw calls** dan menjaga performa 60 FPS pada perangkat web/PWA kelas menengah.

### 1.1 Mengapa Texture Atlas (Bukan Banyak File Individual)

Setiap gambar individual (`knight_idle_01.png`, `knight_idle_02.png`, ...) yang dimuat ke GPU browser menjadi satu **WebGL texture** tersendiri. Tiap texture yang berbeda memicu **texture swap** dan **draw call** baru saat dirender. Pada game RPG idle dengan puluhan hero (rarity C/R/E/L), 6 elemen, animasi idle/attack/hit, jumlah texture individual bisa mencapai ratusan hingga ribuan.

Konsekuensi tanpa atlas:

- **Draw calls meledak.** Browser (WebGL) membatasi efisiensi batching; setiap texture beda = panggilan `drawElements`/`drawArrays` terpisah.
- **HTTP requests banyak.** Tiap file = 1 request. PWA/web lambat di jaringan buruk, membebani service worker cache.
- **Texture swap mahal.** GPU harus mengikat (bind) texture berbeda bergantian tiap sprite → stall pipeline, turun FPS.
- **Overhead memori GPU.** Metadata, padding internal, dan alignment tiap texture individual memboroskan VRAM.

Dengan texture atlas:

- **Satu texture besar** berisi ratusan frame → **satu bind texture** untuk seluruh batch sprite di scene yang sama.
- **Satu HTTP request** per atlas (plus 1 JSON metadata).
- **Batching maksimal.** Phaser WebGL renderer dapat menggabungkan ratusan quad sprite ke dalam 1 draw call selama texture sama dan state render kompatibel.
- **Cache PWA efisien.** 1 file besar lebih mudah di-cache service worker daripada ribuan file kecil.

### 1.2 Batasan yang Disadari

- Atlas raksasa ≠ selalu lebih baik (lihat [Anti-Pattern §9](#9-anti-pattern)). Ada batas `MAX_TEXTURE_SIZE` GPU (sering 4096 atau 8192, tapi aman di 2048–4096).
- Batching di Phaser butuh **texture key sama** dan **tidak ada perubahan blend mode / tint / alpha berbeda secara tidak berurutan**. Atlas membantu karena semua frame berbagi texture.
- Atlas TIDAK mengurangi jumlah triangle; ia mengurangi **jumlah state change GPU** (bind, uniform). Itu penyumbang terbesar CPU overhead di render loop.

### 1.3 Metrik Sukses (KPI)

| Metrik | Target | Catatan |
|---|---|---|
| Draw calls per scene battle | ≤ 8 | 1 atlas battle + UI + efek |
| HTTP requests aset gambar awal | ≤ 6 file | preload kritis |
| FPS stabil | 60 (target), 30 (minimum) | perangkat mid-low |
| Waktu load pertama (PWA) | ≤ 3 s (4G) | atlas terkompresi |
| VRAM total (SD) | ≤ 64 MB | desktop boleh lebih |

---

## 2. Atlas Spec

Spesifikasi teknis pembuatan atlas. Semua nilai di bawah adalah **canonical** dan wajib diikuti oleh artist & build pipeline.

### 2.1 Format Atlas

Phaser 3 mendukung dua format metadata atlas dari TexturePacker:

- **JSON Hash** (direkomendasikan) — frame diakses via string key. Cocok untuk penamaan `{category}/{entity}_{action}_{frame}`.
- **JSON Array** — frame diakses via indeks/urutan. Lebih ringkas tapi rawan reorder saat mengubah atlas.

**Keputusan:** Gunakan **JSON Hash** untuk semua atlas game. Key string lebih aman saat atlas di-repack dan memudahkan lookup animasi berbasis nama.

```jsonc
// Contoh header JSON Hash (disingkat)
{
  "frames": {
    "heroes/knight_idle_01": { "frame": {...}, "rotated": false, "trimmed": true, ... },
    "heroes/knight_idle_02": { "frame": {...}, "rotated": false, "trimmed": true, ... }
  },
  "meta": { "app": "https://www.codeandweb.com/texturepacker", "image": "battle_atlas_1.png", "format": "RGBA8888", "size": {"w":2048,"h":2048} }
}
```

### 2.2 Tools Pembuat Atlas

| Tool | Lisensi | Catatan |
|---|---|---|
| **TexturePacker** | Komersial (ada free tier) | Standar industri, ekspor JSON Hash/Array native Phaser 3 |
| **free-tex-packer** | Gratis/OSS | CLI & web, mendukung Phaser 3, bisa di-automasi di build |
| **Shoebox** | Gratis | Berbasis Adobe AIR, GUI sederhana, kurang maintenance |
| **AssetKit / custom script** | OSS | Bila butuh kontrol penuh (packing deterministik) |

**Rekomendasi pipeline:** `free-tex-packer` (CLI) dijalankan via npm script pada fase build, agar reproducible dan terintegrasi CI. TexturePacker boleh dipakai untuk iterasi artist cepat lalu diekspor ke format Phaser.

### 2.3 Aturan Packing (Canonical)

| Parameter | Nilai | Alasan |
|---|---|---|
| `maxSize` | **2048 × 2048** (mobile/WebGL aman) | Kompatibilitas maksimal. Desktop boleh 4096. |
| `maxSizeDesktop` | **4096 × 4096** | GPU desktop modern support; naikkan hanya bila perlu. |
| `powerOfTwo` | **true bila perlu** | WebGL modern toleran NPOT, tapi POT aman untuk mipmap & kompatibilitas lama. Atlas kami POT (2048/4096). |
| `padding` | **2 px** | Jarak antar region cegah sampling overlap. |
| `extrude` | **1 px** | Duplikasi piksel tepi ke luar region → cegah **bleeding** saat scaling/rotation (creates seam). |
| `trim` | **enable** | Pangkas piksel transparan → hemat ruang atlas. Simpan `sourceSize` & `spriteSourceSize` agar posisi render tetap benar. |
| `allowRotation` | **hati-hati (default OFF)** | Phaser 3 support frame rotated, tapi menambah kompleksitas. ON hanya bila densitas atlas kritis; pastikan `rotated:true` dikonsumsi Phaser benar. |
| `shapePadding` | 2 px | Sama dengan padding untuk region tak beraturan. |
| `algorithm` | MaxRects (Best) | Packing paling padat & deterministik. |
| `textureFormat` | PNG-32 (RGBA8888) | Presisi warna; kompresi lossless. WebP bila pipeline HTTP support. |
| `premultiplyAlpha` | **true** | Phaser WebGL default; hindari halo hitam pada sprite semi-transparan. (Pastikan exporter & loader cocok.) |

> **Catatan bleeding:** Tanpa `extrude`/`padding`, saat GPU melakukan bilinear sampling di tepi frame, ia bisa men-sample piksel frame tetangga → garis sampah (seam). Padding+extrude wajib untuk animasi yang di-scale.

### 2.4 Pixel Art vs HD Art

- **Pixel art (C/R rarity sebagian):** gunakan `filter: NearestFilter` (disable smoothing) → `pixelArt: true` di config Phaser. Atlas tetap PNG, tapi jangan di-scale bilinear.
- **HD art (L rarity / boss):** gunakan `LinearFilter` (smoothing) → `pixelArt: false`. Butuh variant HD (lihat [§7](#7-dpi--scale-variants)).

Kedua gaya **jangan dicampur dalam satu atlas** bila filter berbeda (lihat anti-pattern §9.1).

### 2.5 Warna & Format

- Format: **RGBA8888** default (alpha penuh).
- Bila atlas solid tanpa transisi alpha kompleks (mis. tilemap latar), bisa **RGB565/RGBA4444** untuk hemat VRAM — tapi tidak default.
- Hindari format lossy (JPG) untuk sprite ber-alpha.

---

## 3. Naming Convention & Regions

Penamaan frame konsisten = kunci animasi & lookup andal antar atlas.

### 3.1 Pola Penamaan (Canonical)

```
{category}/{entity}_{action}_{frame}
```

| Token | Arti | Contoh nilai |
|---|---|---|
| `category` | Kelompok logis aset | `heroes`, `enemies`, `ui`, `fx`, `icons`, `world` |
| `entity` | ID unik entitas | `knight`, `mage_fire`, `slime`, `boss_dragon` |
| `action` | State animasi | `idle`, `attack`, `hit`, `dead`, `cast`, `walk` |
| `frame` | Nomor frame | `01`..`NN` (zero-padded 2 digit) |

Contoh valid:
- `heroes/knight_idle_01`
- `heroes/knight_attack_03`
- `enemies/slime_hit_02`
- `fx/fire_explosion_05`
- `ui/button_gold`
- `icons/rarity_L`

### 3.2 Tabel Contoh Region (Atlas Battle)

| Frame Key | Region (x,y,w,h) | Trimmed | Rotated | Keterangan |
|---|---|---|---|---|
| `heroes/knight_idle_01` | 0,0,128,128 | ya | tidak | frame idle ke-1 |
| `heroes/knight_idle_02` | 130,0,128,128 | ya | tidak | frame idle ke-2 |
| `heroes/knight_idle_03` | 260,0,128,128 | ya | tidak | frame idle ke-3 |
| `heroes/knight_idle_04` | 390,0,128,128 | ya | tidak | frame idle ke-4 |
| `heroes/knight_attack_01` | 520,0,160,128 | ya | tidak | pose ayun ke-1 |
| `heroes/knight_attack_02` | 682,0,160,128 | ya | tidak | pose ayun ke-2 |
| `heroes/knight_attack_03` | 844,0,160,128 | ya | tidak | pose ayun ke-3 |
| `heroes/mage_idle_01` | 0,130,128,128 | ya | tidak | — |
| `enemies/slime_idle_01` | 130,130,96,96 | ya | tidak | — |
| `fx/slash_01` | 228,130,128,128 | ya | tidak | efek serangan |

> Asumsi: hero base sprite 128×128 (SD). Attack lebih lebar (160) karena ayunan senjata. Lihat §10 untuk asumsi eksplisit.

### 3.3 Enum Action Standar

Gunakan action terbatas agar atlas & animasi konsisten:

```
IDLE, WALK, ATTACK, CAST, HIT, DEAD, VICTORY, CHEER
```

FX terpisah di kategori `fx/` dengan action bebas (nama efek).

### 3.4 Rarity & Elemen dalam Key?

- **Rarity (C/R/E/L)** → jangan masuk key frame; gunakan prefix entity bila art beda: `heroes/knight_L_idle_01` (bila ada varian art rarity). Bila hanya beda tint/stat, pakai tint runtime, bukan atlas beda.
- **Elemen (6 elemen: Fire, Water, Earth, Wind, Light, Dark)** → masuk kategori `fx/` atau suffix entity: `fx/fire_explosion_01`, `fx/water_heal_01`. Sprite hero netral; elemen diekspresikan via efek & tint.

---

## 4. Multi-Atlas Management

Satu atlas 2048×2048 tidak cukup untuk seluruh game (ratusan hero × animasi). Strategi: pecah jadi beberapa **pages** atlas, dikelola per kebutuhan scene.

### 4.1 Kapan Memecah Atlas

- Total region melebihi `maxSize` (2048/4096) → pecah otomatis oleh packer (multi-page).
- Aset beda scene jarang dipakai bersamaan → pisahkan atlas per scene (battle vs map vs shop).
- Aset beda filter (pixel vs smooth) → pisahkan.
- Aset beda DPI variant → pisahkan (lihat §7).

### 4.2 Aturan Pembagian (Canonical)

| Atlas Page | Isi | Dimuat Saat | Dibongkar Saat |
|---|---|---|---|
| `boot_atlas` | logo, loading bar, kursor | Boot scene | tetap (kecil) |
| `ui_atlas` | tombol, panel, ikon umum | Boot/preload | saat app shutdown |
| `battle_atlas_1` | hero raritas C/R + enemy umum + fx dasar | Battle scene start | Battle scene shutdown |
| `battle_atlas_2` | hero raritas E/L + boss + fx elemen | Battle scene start (atau lazy saat party punya) | Battle scene shutdown |
| `map_atlas` | tilemap, NPC, prop dunia | Map scene start | Map shutdown |
| `shop_atlas` | item icon, frame rarity | Shop scene start | Shop shutdown |
| `icon_atlas` | portrait kecil, rarity badge | preload (sering dipakai) | app shutdown |

### 4.3 Penomoran Page

Gunakan suffix `_1`, `_2` ... Bila packer generate multi-page, konsisten beri nama `battle_atlas_{n}.png` + `.json`. Di Phaser, tiap page = **texture key berbeda** namun bisa dibagi animasi lintas page bila perlu (jarang; hindari).

### 4.4 Constraint Draw Calls per Scene

Target: dalam 1 scene, gunakan **maksimal 4–8 texture key aktif** secara bersamaan.

- Battle: `battle_atlas_1` + `battle_atlas_2` + `ui_atlas` + `fx_atlas` = 4 key. Aman.
- Jangan muat `map_atlas` saat battle (bongkar dulu).

### 4.5 Strategi Lazy Load Party

Hero di party pemain (maks 5) → pastikan di `battle_atlas_1`. Hero cadangan (inventory) → boleh di `battle_atlas_2` (lazy load saat dibutuhkan, mis. ganti party di pre-battle).

---

## 5. Mapping Example

Contoh nyata JSON Hash atlas + cara load & animasi di Phaser 3.

### 5.1 File Atlas (hero_knight.json)

```json
{
  "frames": {
    "heroes/knight_idle_01": {
      "frame": { "x": 0, "y": 0, "w": 128, "h": 128 },
      "rotated": false,
      "trimmed": true,
      "spriteSourceSize": { "x": 0, "y": 0, "w": 128, "h": 128 },
      "sourceSize": { "x": 0, "y": 0, "w": 128, "h": 128 },
      "pivot": { "x": 0.5, "y": 0.5 }
    },
    "heroes/knight_idle_02": {
      "frame": { "x": 130, "y": 0, "w": 128, "h": 128 },
      "rotated": false,
      "trimmed": true,
      "spriteSourceSize": { "x": 0, "y": 0, "w": 128, "h": 128 },
      "sourceSize": { "x": 0, "y": 0, "w": 128, "h": 128 },
      "pivot": { "x": 0.5, "y": 0.5 }
    },
    "heroes/knight_idle_03": {
      "frame": { "x": 260, "y": 0, "w": 128, "h": 128 },
      "rotated": false,
      "trimmed": true,
      "spriteSourceSize": { "x": 0, "y": 0, "w": 128, "h": 128 },
      "sourceSize": { "x": 0, "y": 0, "w": 128, "h": 128 },
      "pivot": { "x": 0.5, "y": 0.5 }
    },
    "heroes/knight_idle_04": {
      "frame": { "x": 390, "y": 0, "w": 128, "h": 128 },
      "rotated": false,
      "trimmed": true,
      "spriteSourceSize": { "x": 0, "y": 0, "w": 128, "h": 128 },
      "sourceSize": { "x": 0, "y": 0, "w": 128, "h": 128 },
      "pivot": { "x": 0.5, "y": 0.5 }
    },
    "heroes/knight_attack_01": {
      "frame": { "x": 520, "y": 0, "w": 160, "h": 128 },
      "rotated": false,
      "trimmed": true,
      "spriteSourceSize": { "x": 0, "y": 0, "w": 160, "h": 128 },
      "sourceSize": { "x": 0, "y": 0, "w": 160, "h": 128 },
      "pivot": { "x": 0.5, "y": 0.5 }
    },
    "heroes/knight_attack_02": {
      "frame": { "x": 682, "y": 0, "w": 160, "h": 128 },
      "rotated": false,
      "trimmed": true,
      "spriteSourceSize": { "x": 0, "y": 0, "w": 160, "h": 128 },
      "sourceSize": { "x": 0, "y": 0, "w": 160, "h": 128 },
      "pivot": { "x": 0.5, "y": 0.5 }
    },
    "heroes/knight_attack_03": {
      "frame": { "x": 844, "y": 0, "w": 160, "h": 128 },
      "rotated": false,
      "trimmed": true,
      "spriteSourceSize": { "x": 0, "y": 0, "w": 160, "h": 128 },
      "sourceSize": { "x": 0, "y": 0, "w": 160, "h": 128 },

      "pivot": { "x": 0.5, "y": 0.5 }
    }
  },
  "meta": {
    "app": "https://www.codeandweb.com/texturepacker",
    "version": "1.0",
    "image": "hero_knight.png",
    "format": "RGBA8888",
    "size": { "w": 2048, "h": 2048 },
    "scale": "1",
    "smartupdate": "$TexturePacker:SmartUpdate:knight$"
  },
  "animations": {
    "knight_idle": ["heroes/knight_idle_01", "heroes/knight_idle_02", "heroes/knight_idle_03", "heroes/knight_idle_04"],
    "knight_attack": ["heroes/knight_attack_01", "heroes/knight_attack_02", "heroes/knight_attack_03"]
  }
}
```

> **Catatan:** Blok `"animations"` di atas adalah ekstensi TexturePacker; Phaser 3.60+ dapat langsung membaca `this.anims.createFromAseprite`/atlas animasi. Bila tidak didukung, buat animasi manual (§5.3).

### 5.2 Load di Phaser

```javascript
// di dalam Scene.preload()
preload() {
  // format: load.atlas(keyTexture, pathGambar, pathJSON)
  this.load.atlas(
    'hero_knight',                 // texture key (dipakai di game)
    'assets/atlas/hero_knight.png',
    'assets/atlas/hero_knight.json'
  );
}
```

Penjelasan:
- Argumen 1 = texture key unik (`hero_knight`). Semua frame di atlas ini diakses via key ini.
- Argumen 2 = file PNG atlas.
- Argumen 3 = file JSON metadata (Hash). Phaser parse `frames` otomatis.

Bila atlas multi-page (TexturePacker pecah jadi 2 PNG):

```javascript
preload() {
  this.load.atlas('battle_1', 'assets/atlas/battle_atlas_1.png', 'assets/atlas/battle_atlas_1.json');
  this.load.atlas('battle_2', 'assets/atlas/battle_atlas_2.png', 'assets/atlas/battle_atlas_2.json');
}
```

### 5.3 Buat Animasi Manual

```javascript
// di dalam Scene.create() (setelah preload selesai)
create() {
  // Animasi IDLE: 4 frame, 8 fps, loop
  this.anims.create({
    key: 'knight_idle',
    frames: [
      { key: 'hero_knight', frame: 'heroes/knight_idle_01' },
      { key: 'hero_knight', frame: 'heroes/knight_idle_02' },
      { key: 'hero_knight', frame: 'heroes/knight_idle_03' },
      { key: 'hero_knight', frame: 'heroes/knight_idle_04' }
    ],
    frameRate: 8,
    repeat: -1   // -1 = infinite loop
  });

  // Animasi ATTACK: 3 frame, 12 fps, play once
  this.anims.create({
    key: 'knight_attack',
    frames: [
      { key: 'hero_knight', frame: 'heroes/knight_attack_01' },
      { key: 'hero_knight', frame: 'heroes/knight_attack_02' },
      { key: 'hero_knight', frame: 'heroes/knight_attack_03' }
    ],
    frameRate: 12,
    repeat: 0
  });

  // Buat sprite & mainkan
  const knight = this.add.sprite(400, 300, 'hero_knight', 'heroes/knight_idle_01');
  knight.play('knight_idle');
}
```

### 5.4 Buat Animasi dari Blok `animations` (Phaser 3.60+)

```javascript
create() {
  // jika JSON punya blok "animations", gunakan:
  this.anims.createFromAseprite('hero_knight'); // untuk Aseprite
  // atau parse manual blok animations:
  const json = this.cache.json.get('hero_knight_json');
  if (json && json.animations) {
    for (const [animKey, frames] of Object.entries(json.animations)) {
      this.anims.create({
        key: animKey,
        frames: frames.map(f => ({ key: 'hero_knight', frame: f })),
        frameRate: 10,
        repeat: animKey.endsWith('idle') ? -1 : 0
      });
    }
  }
}
```

### 5.5 Akses Region Tanpa Animasi

```javascript
// mendapatkan ukuran region
const frame = this.textures.get('hero_knight').get('heroes/knight_idle_01');
console.log(frame.width, frame.height); // 128, 128 (atau sourceSize bila trimmed)
```

---

## 6. Animated Sprites

Konfigurasi frame, fps, repeat, yoyo, dan hubungannya dengan game loop.

### 6.1 Konfigurasi Animasi (CDP)

| Field | Nilai Umum | Arti |
|---|---|---|
| `frameRate` | idle 6–10, attack 10–14, hit 12 | fps animasi |
| `repeat` | `-1` idle/loop, `0` one-shot | jumlah pengulangan |
| `repeatDelay` | 0–200 ms | jeda antar loop |
| `yoyo` | `false` (idle), `true` (idle subtle) | main maju-mundur (hemat frame) |
| `showOnStart`/`hideOnComplete` | sesuaikan | sembunyikan fx setelah selesai |
| `delay` | 0 | tunda mulai |

### 6.2 Yoyo untuk Menghemat Frame

Animasi idle bisa pakai **yoyo** agar 4 frame terasa 8 tanpa menambah region:

```javascript
this.anims.create({
  key: 'knight_idle',
  frames: [
    { key: 'hero_knight', frame: 'heroes/knight_idle_01' },
    { key: 'hero_knight', frame: 'heroes/knight_idle_02' },
    { key: 'hero_knight', frame: 'heroes/knight_idle_03' },
    { key: 'hero_knight', frame: 'heroes/knight_idle_04' }
  ],
  frameRate: 6,
  repeat: -1,
  yoyo: true   // 1-2-3-4-3-2-1-2-... (efek napas)
});
```

> Yoyo menghemat atlas (frame lebih sedikit) tapi bisa terasa repetitif. Gunakan untuk idle halus; hindari untuk attack (butuh arc jelas).

### 6.3 Hubungan dengan Game Loop (Interpolasi)

Phaser render loop (`game.loop`) menjalankan:

1. **Update** (logika: `scene.update(time, delta)`).
2. **Pre-update animasi** — `AnimationManager` advance frame berdasar `delta` & `frameRate`.
3. **Render** — WebGL batch quad dari frame aktif tiap sprite.

Interpolasi: animasi maju berbasis **waktu (delta ms)**, bukan per-frame tetap. Artinya di 60 FPS vs 30 FPS, durasi animasi sama (mis. attack 0.25 s), hanya jumlah frame tampil beda. Ini penting agar gameplay turn-based tetap sinkron walau FPS berbeda.

```javascript
// delta-based: Phaser otomatis. Tidak perlu manual kecuali custom tween.
update(time, delta) {
  // jangan ubah anim frame manual tiap update; biarkan AnimationManager.
  // gunakan this.tweens untuk pergerakan (interpolasi posisi):
}
```

### 6.4 State Machine Sprite

Idle RPG butuh transisi antar state (idle ↔ attack ↔ hit ↔ dead). Pola sederhana:

```javascript
class HeroSprite extends Phaser.GameObjects.Sprite {
  constructor(scene, x, y, texture) {
    super(scene, x, y, texture, 'heroes/knight_idle_01');
    this.state = 'idle';
    this.play('knight_idle');
  }
  playAttack() {
    this.stop();
    this.play('knight_attack');
    this.once('animationcomplete', () => this.playIdle());
  }
  playIdle() {
    this.state = 'idle';
    this.play('knight_idle');
  }
  playHit() {
    this.stop();
    this.play('knight_hit'); // one-shot
    this.once('animationcomplete', () => this.playIdle());
  }
}
```

> Gunakan event `animationcomplete` (bukan timer manual) agar transisi presisi mengikuti fps aktual.

### 6.5 Frame Events (Hitbox Timing)

Untuk serangan, saat frame ke-2 (kontak), picu damage:

```javascript
this.anims.create({
  key: 'knight_attack',
  frames: [
    { key: 'hero_knight', frame: 'heroes/knight_attack_01' },
    { key: 'hero_knight', frame: 'heroes/knight_attack_02', duration: 80 }, // kontak
    { key: 'hero_knight', frame: 'heroes/knight_attack_03' }
  ],
  frameRate: 12
});

sprite.on('animationupdate', (anim, frame) => {
  if (frame.frame.name === 'heroes/knight_attack_02') {
    this.dealDamage();
  }
});
```

---

## 7. DPI / Scale Variants

Mendukung layar retina (devicePixelRatio tinggi) tanpa memuat dua variant sekaligus.

### 7.1 Strategi Dua Variant: SD & HD

| Variant | Resolusi region | Dipakai untuk |
|---|---|---|
| **SD** (1x) | 128×128 | `devicePixelRatio <= 1` atau memori rendah |
| **HD** (2x) | 256×256 | `devicePixelRatio >= 2` (retina) |

**Aturan emas:** Jangan muat SD & HD bersamaan. Pilih satu berdasar `window.devicePixelRatio()` saat boot, lalu muat hanya variant itu.

### 7.2 Pemilihan Variant di Boot

```javascript
// BootScene.preload()
preload() {
  const dpr = window.devicePixelRatio || 1;
  const variant = dpr >= 2 ? 'hd' : 'sd';
  this.registry.set('assetVariant', variant);

  // muat hanya satu variant
  this.load.atlas(
    'battle_1',
    `assets/atlas/${variant}/battle_atlas_1.png`,
    `assets/atlas/${variant}/battle_atlas_1.json`
  );
}
```

### 7.3 Penamaan Folder Variant

```
assets/atlas/sd/battle_atlas_1.png
assets/atlas/sd/battle_atlas_1.json
assets/atlas/hd/battle_atlas_1.png
assets/atlas/hd/battle_atlas_1.json
```

Key texture tetap `battle_1` (sama di SD/HD) → kode game tidak peduli variant. Hanya path file beda.

### 7.4 Skala Renderer vs Atlas

Phaser punya `scale.mode` & `resolution`. Pendekatan kami:

- Gunakan **atlas HD** (region 2x) dan biarkan Phaser scale ke layar via `Scale.FIT`/`RESIZE`.
- Atau gunakan **atlas SD** dan naikkan `game.config.resolution = 2` (Phaser legacy) — tapi ini membebani fill-rate. Lebih baik atlas HD + scale down.

> **Rekomendasi:** Atlas HD sebagai sumber, render scale menyesuaikan. SD hanya untuk perangkat sangat lemah (fallback memori).

### 7.5 Deteksi Memori & Fallback

```javascript
// bila VRAM ketat, force SD
const mem = navigator.deviceMemory || 4; // GB (Chrome)
const variant = (dpr >= 2 && mem >= 4) ? 'hd' : 'sd';
```

---

## 8. Integration

Integrasi atlas ke alur asset loading, game loop (batching), dan manajemen memori.

### 8.1 Preload vs Lazy per Scene

**Preload (Boot/Global):** aset selalu perlu (UI, ikon, boot).
**Lazy (per Scene):** aset spesifik scene (battle atlas saat masuk battle).

```javascript
// BootScene: preload kritis
preload() {
  this.load.atlas('ui', 'assets/atlas/sd/ui_atlas.png', 'assets/atlas/sd/ui_atlas.json');
  this.load.atlas('icon', 'assets/atlas/sd/icon_atlas.png', 'assets/atlas/sd/icon_atlas.json');
}

// BattleScene: lazy load atlas battle
preload() {
  const v = this.registry.get('assetVariant');
  this.load.atlas('battle_1', `assets/atlas/${v}/battle_atlas_1.png`, `assets/atlas/${v}/battle_atlas_1.json`);
  this.load.atlas('battle_2', `assets/atlas/${v}/battle_atlas_2.png`, `assets/atlas/${v}/battle_atlas_2.json`);
}
```

### 8.2 Progress Bar

Gunakan `this.load.on('progress', ...)` untuk loading bar (UI atlas).

```javascript
preload() {
  const bar = this.add.graphics();
  this.load.on('progress', (value) => {
    bar.clear();
    bar.fillStyle(0x00ff00);
    bar.fillRect(100, 300, 600 * value, 20); // x,y,w,h
  });
  // ... load atlas
}
```

### 8.3 Batching di Game Loop

Phaser WebGL `WebGLRenderer` secara otomatis **batch** sprite yang:
- Texture key **sama**.
- Tidak ada perubahan **blend mode** di antaranya.
- Tidak ada **tint/custom pipeline** memecah batch.

Agar batch maksimal di battle:
- Urutkan render berdasar atlas: render semua sprite `battle_1`, lalu `battle_2`, lalu `ui`. Phaser urutkan via depth & display list; pastikan sprite se-atlas berdekatan di display list.
- Hindari `setTint` acak di tengah batch (tint menambah pipeline state). Kumpulkan sprite bertint di akhir.
- Gunakan **satu** atlas untuk seluruh unit battle bila muat (prioritas batch).

```javascript
// atur depth agar se-atlas berurutan
battleUnits.forEach((u, i) => u.setDepth(10 + i));
uiElements.forEach((e, i) => e.setDepth(100 + i));
```

### 8.4 Memory: Unload saat Scene Shutdown

Penting: atlas besar harus **dibongkar** saat scene tidak lagi butuh, cegah kebocoran VRAM.

```javascript
// BattleScene.shutdown()
shutdown() {
  this.textures.remove('battle_1'); // hapus texture & bebaskan VRAM
  this.textures.remove('battle_2');
  this.anims.remove('knight_idle');
  this.anims.remove('knight_attack');
  // UI & icon tetap (global) -> jangan remove
}
```

> **Perhatian:** `this.textures.remove(key)` menghapus seluruh atlas. Pastikan tak ada sprite masih mereferensi key tersebut (destroy sprite dulu).

### 8.5 PWA Caching (Service Worker)

Atlas (PNG + JSON) termasuk aset statis → cache via service worker (Workbox).

```javascript
// sw.js (Workbox)
workbox.routing.registerRoute(
  /\/assets\/atlas\/.*\.(png|json)$/,
  new workbox.strategies.CacheFirst({
    cacheName: 'atlas-cache',
    plugins: [
      new workbox.expiration.ExpirationPlugin({ maxEntries: 30, maxAgeSeconds: 60*60*24*30 })
    ]
  })
);
```

> CacheFirst cocok karena atlas immutable (versi via hash nama file: `battle_atlas_1.a1b2c3.png`).

### 8.6 Versioning Atlas

Ganti nama file tiap build (`battle_atlas_1.{hash}.png`) agar service worker & browser cache invalid otomatis. Jangan pakai query string (?v=2) — kurang andal di some SW.

### 8.7 Build Pipeline (npm script)

```jsonc
// package.json
{
  "scripts": {
    "atlas:build": "free-tex-packer --config ./tools/atlas.config.json",
    "atlas:watch": "free-tex-packer --config ./tools/atlas.config.json --watch",
    "build": "npm run atlas:build && vite build"
  }
}
```

```jsonc
// tools/atlas.config.json (free-tex-packer)
{
  "textures": [
    {
      "name": "battle_atlas_1",
      "path": "src/assets/raw/battle_1",
      "maxSize": { "w": 2048, "h": 2048 },
      "padding": 2,
      "extrude": 1,
      "allowRotation": false,
      "trim": true,
      "outputFormat": "json-hash",
      "nameFormat": "{category}/{name}_{frame}"
    }
  ]
}
```

---

## 9. Anti-Pattern

Daftar kesalahan umum yang merusak performa draw calls / VRAM.

### 9.1 Jangan Campur Ukuran Ekstrem dalam 1 Atlas

❌ Satu atlas berisi ikon 16×16 DAN boss 1024×1024.
✅ Pisahkan `icon_atlas` (kecil) dan `boss_atlas` (besar). Campuran memboroskan packing & batching (region kecil tersebar, region besar menguasai).

### 9.2 Jangan Buat Atlas yang Melanggar Batas GPU

❌ Atlas 8192×8192 di perangkat mobile (MAX_TEXTURE_SIZE sering 4096; beberapa 2048).
✅ Batasi 2048 (mobile) / 4096 (desktop). Pecah ke multi-page bila lewat.

```javascript
// deteksi batas GPU
const maxTex = this.renderer.gl.getParameter(this.renderer.gl.MAX_TEXTURE_SIZE);
console.log('Max texture size:', maxTex); // 2048/4096/8192
```

### 9.3 Jangan Buat Terlalu Banyak Atlas Kecil

❌ 50 atlas masing-masing 256×256 (tiap atlas = 1 texture bind = draw call terpisah).
✅ Gabungkan aset terkait ke atlas 2048 yang padat. Terlalu banyak atlas justru **menaikkan** draw calls (bind swap antar atlas).

Aturan: target **≤ 8 texture key aktif per scene** (lihat §4.4).

### 9.4 Jangan Muat SD & HD Bersamaan

❌ Load `battle_sd` dan `battle_hd` lalu pilih runtime.
✅ Pilih 1 variant saat boot (§7.2), muat hanya itu. Menghemat VRAM 2x.

### 9.5 Jangan Lupa Padding / Extrude (Bleeding)

❌ Atlas tanpa padding → seam hitam di tepi sprite saat di-scale.
✅ `padding:2, extrude:1` wajib (§2.3).

### 9.6 Jangan Rotate Frame Tanpa Konfirmasi Phaser

❌ `allowRotation:true` lalu Phaser salah menggambar (frame miring).
✅ Default OFF. Bila ON, pastikan exporter tulis `rotated:true` dan Phaser baca benar (Phaser support sejak lama, tapi uji).

### 9.7 Jangan Biarkan Atlas Tidak Dibongkar (Memory Leak)

❌ Pindah scene tapi atlas tetap di memory → VRAM habis setelah beberapa battle.
✅ `this.textures.remove(key)` di `shutdown()` (§8.4).

### 9.8 Jangan Campur Filter Pixel & Smooth dalam 1 Atlas

❌ Pixel art (nearest) & HD art (linear) di atlas sama → satu filter global, salah satu blur/jagged.
✅ Pisahkan atlas `pixel_atlas` vs `hd_atlas`, atur `texture.setFilter()` sesuai.

### 9.9 Jangan Pakai JPG untuk Sprite Ber-Alpha

❌ JPG tidak support alpha → halo hitam.
✅ PNG-32 (RGBA8888) atau WebP (dengan alpha).

### 9.10 Jangan Over-Trim Hingga Hilangkan Pivot

❌ Trim ekstrem tanpa simpan `spriteSourceSize` → posisi animasi meleset (senjata "loncat").
✅ Aktifkan `trim:true` TAPI simpan metadata source size (TexturePacker default). Phaser restore otomatis via `spriteSourceSize`.

---

## 10. Lampiran: Asumsi Eksplisit

Dokumen ini mengasumsikan hal berikut (ubah bila proyek berbeda):

| # | Asumsi | Dampak |
|---|---|---|
| A1 | Hero base sprite SD = **128×128 px**; HD = 256×256 | Ukuran region §3.2, §7 |
| A2 | Attack pose lebih lebar (**160 px**) dari idle (128) | Region attack beda lebar |
| A3 | 4 frame idle, 3 frame attack, 2 frame hit standar per hero | Jumlah region per hero ≈ 9–12 |
| A4 | Maks 5 hero di party pemain; raritas C/R di `battle_atlas_1`, E/L di `battle_atlas_2` | Pembagian §4.2 |
| A5 | 6 elemen: Fire, Water, Earth, Wind, Light, Dark — diekspresikan via `fx/` + tint, bukan sprite beda | §3.4 |
| A6 | Target 60 FPS; minimal 30 FPS di mid-low device | KPI §1.3 |
| A7 | `devicePixelRatio` ≤1 → SD, ≥2 → HD; fallback SD bila `deviceMemory` <4 GB | §7.2, §7.5 |
| A8 | Phaser 3.60+ (dukungan `animations` block & API terkini) | §5.4 |
| A9 | PNG-32 default; WebP diizinkan bila server HTTP support `image/webp` | §2.3, §2.5 |
| A10 | `premultipliedAlpha: true` (default WebGL Phaser) | §2.3 |
| A11 | Build tool: Vite + free-tex-packer CLI; service worker: Workbox | §8.5, §8.7 |
| A12 | Atlas di-version via hash filename (`battle_atlas_1.{hash}.png`) | §8.6 |

---

## 11. Ringkasan Cepat (Cheat Sheet)

```text
FORMAT      : JSON Hash (TexturePacker / free-tex-packer)
MAX SIZE    : 2048 (mobile) / 4096 (desktop)
PADDING     : 2px   EXTRUDE: 1px   TRIM: on   ROTATE: off
NAMING      : {category}/{entity}_{action}_{frame}
PAGES       : boot / ui / icon / battle_1 / battle_2 / map / shop
LOAD        : this.load.atlas('key', 'x.png', 'x.json')
ANIM        : this.anims.create({ key, frames:[{key,frame}], frameRate, repeat })
DPI         : pilih SD/HD saat boot via devicePixelRatio, jangan dua-dua
UNLOAD      : this.textures.remove(key) di scene.shutdown()
ANTI        : jangan campur ukuran ekstrem / jangan atlas >GPU limit /
              jangan terlalu banyak atlas kecil / jangan lupa padding
```

---

## 12. Referensi

- Phaser 3 Docs — Texture Atlas: https://docs.phaser.io/api-documentation/class/loader-loaderplugin#atlas
- Phaser 3 Docs — AnimationManager: https://docs.phaser.io/api-documentation/class/animations-animationmanager
- TexturePacker — Phaser 3 export: https://www.codeandweb.com/texturepacker
- free-tex-packer: https://github.com/code-and-web/free-tex-packer
- WebGL `MAX_TEXTURE_SIZE`: https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API/Constants
- Workbox (PWA caching): https://developer.chrome.com/docs/workbox

---

*Dokumen selesai. Sesuai Canonical Design Parameters (CDP). Engine Phaser 3, format JSON Hash, max 2048/4096, padding 2 + extrude 1, penamaan {category}/{entity}_{action}_{frame}, multi-page per scene, SD/HD by DPR, unload saat shutdown.*

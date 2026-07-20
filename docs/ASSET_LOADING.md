# ASSET LOADING STRATEGY — RPG Fantasy Turn-Based Idle (Web + PWA)

> **Dokumen:** `ASSET_LOADING.md`
> **Engine:** Phaser 3 + TypeScript + Vite + PWA (vite-plugin-pwa / Workbox)
> **Genre:** RPG Fantasy, Turn-Based, Idle
> **Penulis:** Game Technical Artist / Engine Programmer
> **Versi:** 1.0
> **Bahasa:** Indonesia

---

## CATATAN EDITOR & ASUMSI EKSPLISIT

Dokumen ini mengikat seluruh parameter desain ke **Canonical Design Parameters (CDP)** yang disepakati tim. Untuk menjaga dokumen tetap komprehensif tanpa menunggu klarifikasi, asumsi berikut **ditetapkan secara eksplisit** dan wajib dipakai sampai ada revisi tertulis:

| # | Asumsi | Nilai / Keputusan |
|---|--------|-------------------|
| A1 | Target platform | Mobile (Android/iOS Safari) + Desktop Chrome/Edge. Layar utama portrait 720×1280 (logical), desktop skala ulang. |
| A2 | Koneksi rata-rata user | 4G (dengan fallback 3G lambat). Strategi harus tahan 3G. |
| A3 | Budget awal (initial load) | **< 15 MB** (app shell + font + UI dasar + common atlas). |
| A4 | Budget per-scene tambahan | **< 5 MB** dimuat lazy saat masuk scene. |
| A5 | Total aset di disk (seluruh game) | ~120–200 MB (tidak semua dimuat sekaligus; di-stream lazily). |
| A6 | Sistem elemen | 6 elemen: **Fire, Water, Earth, Wind, Light, Dark**. |
| A7 | Rarity hero | **C (Common), R (Rare), E (Epic), L (Legendary)**. |
| A8 | Scene utama | Boot, Preload, MainMenu, Battle, Gacha, Guild, Market, HeroDetail, Settings. |
| A9 | Renderer | WebGL (Phaser.AUTO, fallback Canvas). Memory leak paling kritis di WebGL texture. |
| A10 | Build tool | Vite + vite-plugin-pwa (Workbox GenerateSW mode). |
| A11 | Audio format | OGG (prioritas) + MP3 (fallback Safari/iOS). |
| A12 | Bahasa UI | Indonesia + Inggris (string di JSON, bukan di texture). |
| A13 | Offline | PWA harus bisa boot offline setelah first visit (precache app shell). |

---

## DAFTAR ISI

1. [Tujuan & Kategori Aset](#1-tujuan--kategori-aset)
2. [Preload (Critical/Boot) vs Lazy/On-Demand](#2-preload-criticalboot-vs-lazyon-demand)
3. [Phaser Loader](#3-phaser-loader)
4. [Texture Atlas & AudioSprite](#4-texture-atlas--audiosprite)
5. [Object Pooling & Reuse](#5-object-pooling--reuse)
6. [Memory Management (PENTING)](#6-memory-management-penting)
7. [Cache Busting & Versioning](#7-cache-busting--versioning)
8. [Error Handling & Retry](#8-error-handling--retry)
9. [PWA Caching Layer](#9-pwa-caching-layer)
10. [Budget & Metric](#10-budget--metric)
11. [Hubungan Dokumen Lain](#11-hubungan-dokumen-lain)
12. [Lampiran: Checklist & Glosarium](#12-lampiran-checklist--glosarium)

---

## 1. Tujuan & Kategori Aset

### 1.1 Tujuan Strategi

Tujuan utama dokumen ini:

- **Boot cepat & stabil**: user melihat app shell < 3 detik di 4G, < 8 detik di 3G.
- **Tidak ada crash karena OOM**: browser (terutama iOS Safari) mematikan tab jika memori WebGL/JS heap meledak. Idle game berjalan lama → leak fatal.
- **Streaming bertahap**: hanya aset scene aktif yang di memori. Pindah scene = unload aset lama.
- **Offline-capable**: lewat PWA, setelah install/first visit game bisa dimainkan tanpa jaringan.
- **Resilience**: asset gagal dimuat harus retry dengan backoff + fallback placeholder, tidak crash.

### 1.2 Kategori Aset & Strategi per Kategori

Setiap kategori punya karakteristik memori & I/O berbeda. Strategi dibedakan:

| Kategori | Format | Loader Phaser | Strategi Muat | Risiko Utama |
|----------|--------|---------------|---------------|--------------|
| Image / Texture Atlas | PNG (webp opsional), atlas JSON | `load.image`, `load.atlas`, `load.spritesheet` | Atlas dikompres + di-batch. Boot = common atlas; scene = dedicated atlas. | WebGL texture leak; fragmentasi VRAM. |
| Audio | OGG + MP3 | `load.audio` (array URL) | Preload SFX kecil; music streaming/runtime cache. AudioSprite untuk SFX. | Dekode audio block main thread; banyak node = memory. |
| JSON / Config | `.json` (stage, skill, banner, hero) | `load.json` | Semua config kecil (< 200 KB total) → masuk Boot. Data dinamis lewat API. | Parsing besar block; version drift. |
| Font | WebFont (woff2) + Bitmap Font | `WebFont`/CSS + `load.bitmapFont` | UI font woff2 di app shell; in-game text pakai Bitmap Font (atlas). | FOIT/FOUT; texture font leak. |
| Shader | `.frag`/`.vert` (GLSL) | `load.glsl` / `Phaser.Display.Shader` | Shader global sedikit (hanya efek elemen). Dimuat sekali di Boot. | Recompile shader; context loss. |
| Lain (video iklan, lottie) | mp4 / json | `load.video` (jarang) | Opsional, lazy saat event. | Berat; hindari di idle game. |

#### 1.2.1 Image / Texture Atlas

- **Format utama:** PNG-8/PNG-32 tergantung alpha. Gunakan **WebP** untuk foto/banner besar (lossy) bila target browser mendukung (iOS 14+). Phaser `load.image` bisa muat webp; fallback PNG lewat `load.image` dengan deteksi.
- **Atlas:** selalu kemas sprite ke atlas (TexturePacker / `phaser-pack` / `free-tex-packer`). Satu atlas = satu draw call group + satu WebGL texture. Hindari ribuan gambar lepas.
- **Resolusi:** gunakan **2 skala** — `@1x` (1280 lebar max) dan `@2x` (retina). Pilih via `window.devicePixelRatio` + ukuran layar. Default load `@2x` untuk mobile; desktop bisa `@1x` bila layar besar tapi DPR=1.
- **Max texture size:** batasi atlas ke **2048×2048** (aman di iOS). Jika perlu 4096, cek `gl.getParameter(gl.MAX_TEXTURE_SIZE)`.
- **Pow2:** Phaser 3 tidak wajib pow2, tapi atlas 2048 (pow2) lebih aman untuk mipmap/filtering.

#### 1.2.2 Audio

- **Format ganda:** `load.audio('sfx_hit', ['sfx_hit.ogg', 'sfx_hit.mp3'])`. Phaser pilih yang didukung.
- **AudioSprite:** semua SFX pendek (< 2 detik) digabung jadi satu file + json region → 1 decode, banyak playback. Lihat [§4](#4-texture-atlas--audiosprite) & Audio Pipeline doc.
- **Music:** file panjang (1–3 MB) di-stream lewat runtime cache; jangan precache semua music.
- **Decode:** Phaser decode async; jangan mainkan sebelum `complete`. Gunakan `sound.once('decoded')`.

#### 1.2.3 JSON / Config

- **Stage config, skill config, hero config, banner config, gacha rates** → JSON kecil. Total < 200 KB → muat di Boot, simpan di registry/global state terkelola (bukan di Phaser cache global tanpa batas).
- **Versioning:** setiap JSON punya field `"version"` untuk deteksi stale (lihat [§7](#7-cache-busting--versioning)).

#### 1.2.4 Font

- **UI/system font:** woff2 via CSS `@font-face`, dimuat di app shell (precache PWA). Hindari FOIT dengan `font-display: swap`.
- **In-game Bitmap Font:** teks damage number, nama skill → Bitmap Font (angka + huruf) sebagai atlas texture. Satu atlas font kecil, dimuat di Boot (common).
- **Jangan** pakai DOM text overlay untuk teks beranimasi banyak → mahal. Pakai BitmapText Phaser.

#### 1.2.5 Shader (GLSL)

- Elemen effect (Fire/Wind particle), glow legendary, screen transition.
- Jumlah shader **kecil & tetap** → compile sekali di Boot, reuse via `Phaser.Display.Shader` / pipeline.
- Simpan referensi di manager, jangan bikin baru tiap frame.

---

## 2. Preload (Critical/Boot) vs Lazy/On-Demand

Prinsip inti: **Boot = apa yang wajib supaya game bisa diinteraksi. Scene = apa yang khusus untuk scene itu.**

### 2.1 Tier 1 — Boot / Critical (App Shell)

Dimuat di `BootScene` + `PreloadScene`. Tanpa ini, game tidak bisa jalan.

- Phaser engine bundle (sudah di JS).
- App shell HTML/CSS/JS (PWA precache).
- WebFont woff2 (UI).
- Common atlas: tombol, panel, icon UI, loading bar, cursor.
- Bitmap font (angka + huruf dasar).
- Global JSON config (hero, skill, stage, banner, gacha rates) — kecil.
- Shader global (glow, transition).
- 1 SFX kecil untuk klik UI (via AudioSprite common).

### 2.2 Tier 2 — Lazy / On-Demand (Per-Scene)

Dimuat saat `scene.start()` / `scene.launch()`. Di-unload saat `shutdown`.

- **Battle:** battle atlas (hero sprite per elemen, enemy, FX projectile, background battle), battle SFX, battle BGM (runtime cache).
- **Gacha:** gacha atlas (portal, banner hero, rarity glow C/R/E/L), gacha SFX, summon animation shader.
- **Guild:** guild atlas (building, member avatar placeholder), guild BGM.
- **Market:** market atlas (item icon, coin, shop UI), market SFX.
- **HeroDetail:** hero full-art atlas per rarity, skill icon atlas.
- **Settings:** minimal, reuse common.

### 2.3 Tabel Keputusan Muat

| Aset | Kapan Dimuat | Alasan |
|------|--------------|--------|
| Phaser bundle + app shell | Boot (precache PWA) | Wajib agar game boot offline & cepat. |
| WebFont woff2 | Boot | UI terlihat konsisten, cegah FOIT. |
| Common UI atlas | Boot | Dipakai di semua scene (tombol, panel). |
| Bitmap font (angka) | Boot | Damage number di battle & seluruh UI. |
| Global JSON config | Boot | Kecil (<200KB), dipakai lintas scene. |
| Global shader (glow/transition) | Boot | Efek lintas scene, compile sekali. |
| UI click SFX (AudioSprite common) | Boot | Feedback instan di semua scene. |
| Battle hero/enemy atlas | Lazy (Battle) | Berat, hanya saat battle. |
| Battle background | Lazy (Battle) | Besar, scene-specific. |
| Battle BGM | Runtime cache (Battle) | Streaming, tidak precache. |
| Gacha portal/banner atlas | Lazy (Gacha) | Khusus gacha. |
| Gacha summon shader | Lazy (Gacha) | Animasi summon. |
| Guild building atlas | Lazy (Guild) | Khusus guild. |
| Market item/coin atlas | Lazy (Market) | Khusus market. |
| Hero full-art per rarity | Lazy (HeroDetail) | Berat, hanya saat lihat detail. |
| Event/banner limited atlas | Lazy (saat event aktif) | Musiman, tidak selalu ada. |

### 2.4 Aturan Transisi Scene

```
Masuk Scene B:
  1. (di Scene A shutdown) -> unload semua aset Tier-2 milik A
  2. (di Scene B create)   -> load aset Tier-2 milik B (dengan loading bar bila > 1MB)
  3. Common atlas TETAP di memori (tidak di-unload)
```

Common atlas = "selamanya di memori setelah Boot" (aman, kecil). Tier-2 = "pinjam, kembalikan".

---

## 3. Phaser Loader

### 3.1 Dasar `this.load`

Di dalam `preload()` scene, gunakan `this.load.*`. Semua loader antri & dijalankan Phaser setelah `preload()` selesai.

```typescript
// BattleScene.preload()
preload(): void {
  // Image tunggal (jarang; prefer atlas)
  this.load.image('bg_battle_01', 'assets/battle/bg_battle_01.webp');

  // Texture atlas (png + json)
  this.load.atlas(
    'battle_atlas',
    'assets/battle/battle_atlas.png',
    'assets/battle/battle_atlas.json'
  );

  // Spritesheet (frame grid)
  this.load.spritesheet('hero_fire', 'assets/hero/hero_fire.png', {
    frameWidth: 128,
    frameHeight: 128,
    margin: 1,
    spacing: 1,
  });

  // Audio ganda format
  this.load.audio('bgm_battle', [
    'assets/audio/bgm_battle.ogg',
    'assets/audio/bgm_battle.mp3',
  ]);

  // JSON config
  this.load.json('stage_config', 'assets/config/stage_config.json');

  // Bitmap font
  this.load.bitmapFont(
    'num_font',
    'assets/font/num_font.png',
    'assets/font/num_font.xml'
  );

  // GLSL shader
  this.load.glsl('glow_frag', 'assets/shader/glow.frag');
}
```

### 3.2 Event `progress` & `complete`

Gunakan untuk loading bar UI.

```typescript
preload(): void {
  const bar = this.add.graphics();
  const pct = this.add.text(/* ... */);

  this.load.on('progress', (value: number) => {
    // value: 0..1
    bar.clear();
    bar.fillStyle(0xffcc00, 1);
    bar.fillRect(40, 300, 640 * value, 24);
    pct.setText(`${Math.floor(value * 100)}%`);
  });

  this.load.on('complete', () => {
    bar.destroy();
    pct.destroy();
  });

  // ... loader calls ...
}
```

Catatan:
- `progress` memancar per-file, bukan per-byte. Untuk smoothing, interpasi UI.
- `complete` artinya semua loader selesai (sukses ATAU gagal setelah retry, lihat [§8](#8-error-handling--retry)).
- Jangan `scene.start()` manual di dalam `complete` jika sudah pakai flow normal; gunakan `create()`.

### 3.3 Parallel vs Sequential

- **Default Phaser = parallel** (semua `load.*` di-antri & di-fetch bersamaan, dibatasi oleh browser connection pool ~6).
- **Sequential** hanya bila perlu urutan (mis. load shader dulu sebelum efek). Gunakan `this.load.queue` manual atau bagi jadi dua scene berurutan.
- **Rekomendasi:** parallel untuk atlas/audio/json sekaligus (cepat). Untuk first-load kritis, batasi jumlah file agar progress bar halus (bundle ke atlas besar daripada 50 file kecil).

### 3.4 Loader untuk Boot Scene

```typescript
export class BootScene extends Phaser.Scene {
  constructor() { super('Boot'); }

  preload(): void {
    // Hanya aset kritis (Tier-1)
    this.load.atlas('common_ui', 'assets/common/ui.png', 'assets/common/ui.json');
    this.load.bitmapFont('num_font', 'assets/font/num_font.png', 'assets/font/num_font.xml');
    this.load.json('global_config', 'assets/config/global_config.json');
    this.load.audioSprite('sfx_common', 'assets/audio/sfx_common.json', [
      'assets/audio/sfx_common.ogg', 'assets/audio/sfx_common.mp3',
    ]);
  }

  create(): void {
    // Simpan config ke registry terkelola (bukan global lepas)
    const cfg = this.cache.json.get('global_config');
    this.registry.set('config', cfg);
    this.scene.start('Preload'); // atau langsung MainMenu bila cukup
  }
}
```

### 3.5 Multi-File Loading Bar di PreloadScene

Untuk initial load, gabungkan progress semua loader kritis + Tampilkan perkiraan MB.

```typescript
export class PreloadScene extends Phaser.Scene {
  private loadedMB = 0;
  constructor() { super('Preload'); }

  preload(): void {
    const total = 14.5; // MB estimasi initial (lihat Budget §10)
    this.load.on('progress', (v: number) => {
      this.loadedMB = v * total;
      this.updateBar(v, this.loadedMB, total);
    });
    // ... load Tier-1 ...
  }

  private updateBar(v: number, mb: number, total: number): void {
    // render bar + teks `${mb.toFixed(1)} / ${total} MB`
  }
}
```

---

## 4. Texture Atlas & AudioSprite

Bagian ini **hanya integrasi**; detail pembuatan atlas & region audio ada di dokumen terpisah:
- **Spritesheet Layout** (`SPRITESHEET_LAYOUT.md`) — grid frame, margin/spacing, padding, packing.
- **Audio Pipeline** (`AUDIO_PIPELINE.md`) — encode OGG/MP3, AudioSprite region, normalisasi.

### 4.1 Integrasi Atlas

- Atlas dibuat di TexturePacker/FreeTexPacker → `*.png` + `*.json` (format Phaser/JSON Hash).
- Di game: `this.load.atlas(key, png, json)`.
- Akses frame: `this.textures.get(key).get('frame_name')` atau langsung di sprite `this.add.sprite(x,y,'atlas','hero_idle_1')`.
- **Naming convention (CDP):** `elemen_rarity_aksi_index` → `fire_l_summon_01`. Konsisten agar loader & animasi tidak salah frame.

### 4.2 Integrasi AudioSprite

- AudioSprite = 1 file audio + json berisi region `{ sfx_hit: { start: 0, end: 1.2 }, ... }`.
- Loader: `this.load.audioSprite('sfx_common', jsonUrl, [ogg, mp3])`.
- Mainkan: `this.sound.playAudioSprite('sfx_common', 'sfx_hit')`.
- Manfaat: 1 decode untuk puluhan SFX → hemat memori & startup.

### 4.3 Aturan Atlas per Scene

| Atlas | Key | Isi | Unload saat |
|-------|-----|-----|-------------|
| `common_ui` | Boot | Tombol, panel, icon | Tidak (selamanya) |
| `battle_atlas` | Battle | Hero/enemy/FX/bg | Battle shutdown |
| `gacha_atlas` | Gacha | Portal/banner/glow | Gacha shutdown |
| `guild_atlas` | Guild | Building/avatar | Guild shutdown |
| `market_atlas` | Market | Item/coin/shop | Market shutdown |
| `hero_art` (per rarity) | HeroDetail | Full art C/R/E/L | HeroDetail shutdown |

---

## 5. Object Pooling & Reuse

Idle game = banyak sprite bergerak (projectile, particle, damage number, floating text). **Jangan `new`/`destroy` tiap frame** → GC pressure + fragmentasi.

### 5.1 Konsep

Pool: array objek siap-pakai. `spawn()` ambil dari pool (atau buat baru bila kosong). `recycle()` kembalikan ke pool (set `active=false`, `visible=false`), bukan `destroy()`.

### 5.2 Pool Sederhana ( Generic )

```typescript
export class ObjectPool<T> {
  private free: T[] = [];
  private readonly factory: () => T;
  private readonly reset: (obj: T) => void;

  constructor(factory: () => T, reset: (obj: T) => void, preload = 16) {
    this.factory = factory;
    this.reset = reset;
    for (let i = 0; i < preload; i++) this.free.push(factory());
  }

  obtain(): T {
    const obj = this.free.pop() ?? this.factory();
    return obj;
  }

  release(obj: T): void {
    this.reset(obj);
    this.free.push(obj);
  }

  get size(): number { return this.free.length; }
}
```

### 5.3 Pool Damage Number (BitmapText)

```typescript
export class DamageNumberPool {
  private pool: Phaser.GameObjects.BitmapText[] = [];
  constructor(
    private scene: Phaser.Scene,
    private fontKey = 'num_font',
  ) {}

  spawn(x: number, y: number, value: number, color: string): Phaser.GameObjects.BitmapText {
    let t = this.pool.pop();
    if (!t) {
      t = this.scene.add.bitmapText(x, y, this.fontKey, '', 32);
    }
    t.setActive(true).setVisible(true).setPosition(x, y).setText(`${value}`).setTint(Phaser.Display.Color.HexStringToColor(color).color);
    this.scene.tweens.add({
      targets: t, y: y - 40, alpha: 0, duration: 600,
      onComplete: () => this.recycle(t!),
    });
    return t;
  }

  private recycle(t: Phaser.GameObjects.BitmapText): void {
    t.setActive(false).setVisible(false).setAlpha(1);
    this.pool.push(t);
  }
}
```

### 5.4 Pool Projectile (Battle)

```typescript
export class ProjectilePool {
  private free: Phaser.GameObjects.Image[] = [];
  constructor(private scene: Phaser.Scene, private texture: string) {}

  fire(x: number, y: number, tx: number, ty: number, onHit: () => void): void {
    const p = this.free.pop() ?? this.scene.add.image(0, 0, this.texture);
    p.setActive(true).setVisible(true).setPosition(x, y);
    this.scene.physics.moveTo(p, tx, ty, 400);
    this.scene.time.delayedCall(1200, () => {
      onHit();
      this.recycle(p);
    });
  }

  private recycle(p: Phaser.GameObjects.Image): void {
    p.setActive(false).setVisible(false);
    this.free.push(p);
  }
}
```

### 5.5 Aturan Pooling

- **Wajib pool:** projectile, particle, damage number, floating combo text, hit spark.
- **Boleh `destroy`:** objek scene-lifetime (background, panel) — hanya saat scene shutdown.
- **Pre-grow:** inisialisasi pool dengan N objek di `create()` agar tidak allocate saat pertempuran sengit.
- **Cap:** bila `free` terlalu besar (mis. > 200), bisa `destroy` sebagian untuk bebaskan memori saat idle.
- Integrasi dengan **Game Loop doc** (lihat [§11](#11-hubungan-dokumen-lain)): pooling dipanggil dari `update()`.

---

## 6. Memory Management (PENTING)

Ini bagian paling kritis untuk idle game. Tab browser (terutama iOS Safari) **dibunuh** bila memori naik terus. Idle game jalan berjam-jam → leak = crash pasti.

### 6.1 Aturan Emas

> **Setiap `load` harus punya `unload`.**
> Setiap `add` yang kepunyaan scene harus `destroy` di `shutdown`.
> Setiap listener harus `off` di `shutdown`.

### 6.2 Scene Lifecycle Hook

Phaser panggil `scene.events.on('shutdown', ...)` saat scene berhenti (sebelum `destroy`). Gunakan untuk unload aset Tier-2.

```typescript
export class BattleScene extends Phaser.Scene {
  private loadedKeys: string[] = []; // aset milik scene ini

  preload(): void {
    this.load.atlas('battle_atlas', 'assets/battle/battle_atlas.png', 'assets/battle/battle_atlas.json');
    this.loadedKeys.push('battle_atlas');
    this.load.audio('bgm_battle', ['assets/audio/bgm_battle.ogg', 'assets/audio/bgm_battle.mp3']);
    this.loadedKeys.push('bgm_battle');
  }

  create(): void {
    this.events.once('shutdown', this.onShutdown, this);
    this.events.once('destroy', this.onDestroy, this);
  }

  private onShutdown(): void {
    // 1. Hentikan & unload audio
    this.sound.stopByKey('bgm_battle');
    // 2. Unload texture atlas milik scene
    this.loadedKeys.forEach((key) => {
      if (this.textures.exists(key)) {
        this.textures.remove(key); // bebaskan WebGL texture
      }
      if (this.cache.audio.exists(key)) {
        this.cache.audio.remove(key);
      }
    });
    this.loadedKeys = [];
  }

  private onDestroy(): void {
    // Bersihkan reference eksplisit
    this.projectilePool = undefined as any;
  }
}
```

### 6.3 Jangan Simpan Texture di Global Tanpa Manajemen

- **Salah:** `window.globalTextures = this.textures` lalu lupa hapus → texture menumpuk tiap masuk scene.
- **Benar:** gunakan `this.textures` (milik game, tapi track key), dan hapus via `textures.remove(key)` saat tidak dipakai.
- Simpan hanya **referensi** (key string), bukan objek texture, di state global.

### 6.4 Bersihkan Reference & Listener

```typescript
create(): void {
  this.scale.on('resize', this.onResize, this);
  this.input.on('pointerdown', this.onTap, this);
}
// di shutdown:
private onShutdown(): void {
  this.scale.off('resize', this.onResize, this);
  this.input.off('pointerdown', this.onTap, this);
  this.tweens.killAll();      // hentikan tween gantung
  this.time.removeAllEvents(); // hapus delayedCall
}
```

- `this.tweens.killAll()` penting: tween yang menahan reference ke game object cegah GC.
- `this.time.removeAllEvents()` cegah callback api setelah scene mati.

### 6.5 Pantau WebGL Textures & JS Heap

**Chrome DevTools — Memory tab:**
- **Heap Snapshot:** ambil 2 snapshot (sebelum & sesudah bolak-balik scene 5×). Bandingkan; cari objek `WebGLTexture` / `Phaser.Textures.Texture` yang jumlahnya naik terus → leak.
- **Allocation instrumentation:** rekam saat loop scene; lihat objek tidak dibebaskan.
- **Detached DOM / Phaser object:** cari `Phaser.GameObjects.Image` terlepas tapi masih di heap.

**Chrome DevTools — Performance tab:**
- Rekam 10 detik saat idle + battle. Lihat GC spike (garbage collection sering = allocate tiap frame → perbaiki pooling).
- Lihat "JS heap" line naik terus tanpa turun = leak.

**WebGL info:**
```typescript
// Cek jumlah texture & total VRAM (perkiraan)
const tm = this.game.renderer.textureManager as any;
console.log('Textures in manager:', Object.keys(tm.textures).length);
```

**iOS Safari:** tidak ada DevTools lengkap; gunakam **Safari Web Inspector** (tethari iPhone ke Mac). Pantau "Memory" graph; bila merah → OOM imminent.

### 6.6 Strategi LRU Cache untuk Atlas Jarang Dipakai

Untuk atlas yang jarang tapi berat (event banner, hero art legendary), pakai **LRU (Least Recently Used)** dengan cap.

```typescript
export class AtlasLRU {
  private cache = new Map<string, number>(); // key -> lastUsedTick
  private readonly cap: number;
  constructor(cap = 6) { this.cap = cap; }

  touch(key: string): void {
    this.cache.delete(key);
    this.cache.set(key, performance.now());
    if (this.cache.size > this.cap) {
      const oldestKey = this.cache.keys().next().value!;
      this.cache.delete(oldestKey);
      // panggil callback unload
      this.onEvict?.(oldestKey);
    }
  }
  onEvict?: (key: string) => void;
}
```

- Atlas event yang tidak dipakai > N menit → `textures.remove(key)`.
- Cap cegah akumulasi (mis. user buka 20 hero detail berurutan).
- Common atlas **tidak** masuk LRU (selalu ada).

### 6.7 Context Loss Handling

WebGL context bisa hilang (tab background lama, GPU reset). Tangani:

```typescript
this.game.events.on(Phaser.Core.Events.CONTEXT_LOST, () => {
  console.warn('WebGL context lost — pause game');
  this.scene.pause();
});
this.game.events.on(Phaser.Core.Events.CONTEXT_RESTORED, () => {
  console.warn('WebGL context restored — reload textures');
  // Phaser otomatis restore sebagian; validate atlas ada
});
```

### 6.8 Checklist Memory per Scene

- [ ] `shutdown` hook terdaftar.
- [ ] Semua `loadedKeys` di-`textures.remove`.
- [ ] Semua audio di-`stopByKey` + `cache.audio.remove`.
- [ ] Listener `scale`/`input` di-`off`.
- [ ] `tweens.killAll()` + `time.removeAllEvents()`.
- [ ] Reference ke pool / object di-null-kan.
- [ ] Tidak ada `setInterval` global tanpa `clearInterval`.

---

## 7. Cache Busting & Versioning

Asset di-cache browser + PWA → bila file diperbarui tapi nama sama, user dapat versi lama (stale). Solusi: **content hash filename**.

### 7.1 Content Hash (Vite)

Vite otomatis hash asset build:

```
assets/battle_atlas-a1b2c3d4.png
assets/battle_atlas-a1b2c3d4.json
```

`load.atlas` memakai path hasil build (via `import` atau manifest). Di kode, jangan hardcode path hash; gunakan manifest:

```typescript
// vite menyuntikkan URL hashed lewat import
import battleAtlasPng from './assets/battle/battle_atlas.png';
import battleAtlasJson from './assets/battle/battle_atlas.json?url';

this.load.atlas('battle_atlas', battleAtlasPng, battleAtlasJson);
```

Atau generate `asset-manifest.json` di build step lalu baca di runtime.

### 7.2 CDN Cache Headers

- File hash → header `Cache-Control: public, max-age=31536000, immutable` (1 tahun, karena hash berubah bila konten berubah).
- `index.html` / `sw.js` → `Cache-Control: no-cache` (selalu validasi).
- JSON config dinamis → `max-age=300` + `ETag`.

### 7.3 PWA Precache Manifest Tetap Valid

- `vite-plugin-pwa` GenerateSW otomatis masukkan asset hashed ke precache manifest.
- **Masalah:** bila kita pakai `import` + lazy `import()` (dynamic), Vite masukkan ke chunk terpisah & di-precache otomatis bila di `includeAssets`.
- Pastikan `globPatterns` sw mencakup `**/*.{png,json,ogg,mp3,webp,woff2}`.
- `navigateFallback: 'index.html'` untuk SPA routing.

### 7.4 Version Field di JSON

```json
{ "version": 12, "stages": [ ... ] }
```

Di `BootScene`, bandingkan `cfg.version` dengan `registry.get('lastConfigVersion')`. Bila beda → fetch ulang / invalidate cache (lihat [§9](#9-pwa-caching-layer)).

---

## 8. Error Handling & Retry

Jaringan buruk / 404 / timeout harus **tidak crash game**.

### 8.1 Phaser Load Error Event

```typescript
this.load.on('loaderror', (file: Phaser.Loader.File) => {
  console.error(`Gagal muat: ${file.key} (${file.url})`);
  this.handleAssetError(file);
});
```

### 8.2 Retry dengan Exponential Backoff

```typescript
private retries = new Map<string, number>();
private readonly MAX_RETRY = 3;

private handleAssetError(file: Phaser.Loader.File): void {
  const n = (this.retries.get(file.key) ?? 0) + 1;
  if (n > this.MAX_RETRY) {
    this.useFallback(file.key);
    return;
  }
  this.retries.set(file.key, n);
  const delay = Math.min(1000 * 2 ** n, 8000); // 2s, 4s, 8s
  this.time.delayedCall(delay, () => {
    this.load.atlas(file.key, file.url, /* json url */);
    this.load.start(); // ulangi load
  });
}
```

### 8.3 Fallback Placeholder

- Texture gagal → pakai `common_ui` placeholder (kotak abu + "?").
- Audio gagal → silent (jangan crash). Game tetap jalan tanpa SFX.
- JSON gagal → pakai config bundled di JS (default) agar game tidak blank.

```typescript
private useFallback(key: string): void {
  if (this.textures.exists(key)) return;
  // buat texture 1x1 abu sebagai placeholder
  const g = this.make.graphics({ x: 0, y: 0 }, false);
  g.fillStyle(0x444444).fillRect(0, 0, 64, 64);
  g.generateTexture(key, 64, 64);
  g.destroy();
}
```

### 8.4 Timeout Global

Bila `load` tidak selesai dalam X detik (jaringan gantung), batalkan & tampilkan "Coba Lagi".

```typescript
private loadWithTimeout(timeoutMs = 15000): void {
  const timer = this.time.delayedCall(timeoutMs, () => {
    this.load.stop();
    this.showRetryButton();
  });
  this.load.once('complete', () => timer.remove());
}
```

### 8.5 Aturan Error

- [ ] Setiap kategori punya fallback.
- [ ] Retry max 3 + backoff.
- [ ] Tidak ada `throw` di loader path (cegah white screen).
- [ ] User bisa "Coba Lagi" manual.

---

## 9. PWA Caching Layer

`vite-plugin-pwa` (Workbox) menyediakan Service Worker. Interaksi: **SW cache aset, Phaser cache di memori.**

### 9.1 Precache (App Shell + Common Atlas)

`vite-plugin-pwa` `registerType: 'autoUpdate'` + `workbox.globPatterns`:

```typescript
// vite.config.ts
import { VitePWA } from 'vite-plugin-pwa';
export default defineConfig({
  plugins: [VitePWA({
    registerType: 'autoUpdate',
    workbox: {
      globPatterns: ['**/*.{js,css,html,png,json,ogg,mp3,webp,woff2}'],
      cleanupOutdatedCaches: true,
      navigateFallback: 'index.html',
      runtimeCaching: [
        {
          urlPattern: ({ url }) => url.pathname.startsWith('/assets/audio/'),
          handler: 'StaleWhileRevalidate',
          options: { cacheName: 'audio-cache', expiration: { maxEntries: 60, maxAgeSeconds: 60 * 60 * 24 * 30 } },
        },
        {
          urlPattern: ({ url }) => url.pathname.startsWith('/assets/battle/'),
          handler: 'StaleWhileRevalidate',
          options: { cacheName: 'scene-battle', expiration: { maxEntries: 20, maxAgeSeconds: 60 * 60 * 24 * 7 } },
        },
      ],
    },
  })],
});
```

### 9.2 Runtime Cache (Stale-While-Revalidate)

- **Strategi:** tampilkan aset dari cache (cepat), lalu update di background. Cocok untuk atlas scene & audio (tidak berubah tiap detik).
- **Precache:** hanya app shell + common atlas + global JSON (agar boot offline).
- **Jangan precache:** semua battle/gacha atlas (terlalu besar, > budget). Biarkan runtime cache setelah pertama kali dimuat.

### 9.3 Interplay dengan Memory Management

- SW cache = disk (tidak makan RAM). Phaser `textures` = RAM/VRAM.
- Saat scene shutdown, kita `textures.remove` (bebaskan RAM) TAPI SW tetap simpan di disk → next visit instan.
- Jadi: **unload di RAM, keep di disk.** Keduanya wajib.

### 9.4 Version Drift

- SW update otomatis (`autoUpdate`). Bila `global_config.version` naik, kita bisa `caches.delete()` spesifik lalu re-fetch (lihat [§7.4](#7-cache-busting--versioning)).
- `cleanupOutdatedCaches: true` hapus SW lama saat update.

### 9.5 Offline Boot Flow

```
1. User buka URL pertama kali -> SW install, precache app shell + common.
2. User tutup tab.
3. User buka lagi (offline) -> SW sajikan app shell dari precache.
4. Phaser boot, load common atlas (sudah di cache disk) -> main menu muncul.
5. User klik Battle -> atlas battle belum di disk (first visit offline) -> gagal.
   => Tampilkan "Butuh koneksi untuk mode ini" atau fallback ke cached scenes.
```

Idle game inti (auto-battle, collect reward) harus bisa offline; fitur online (guild, market real-time) butuh jaringan.

---

## 10. Budget & Metric

### 10.1 Target Budget

| Item | Target | Catatan |
|------|--------|---------|
| Initial load (disk transfer) | **< 15 MB** | App shell + font + common atlas + global JSON. |
| Initial load (RAM/VRAM after boot) | **< 60 MB** | Phaser + texture common. |
| Per-scene tambahan (disk) | **< 5 MB** | Atlas battle/gacha/guild/market. |
| Per-scene RAM tambahan | **< 30 MB** | Setelah load, sebelum unload scene lama. |
| Total RAM cap (aman iOS) | **< 250 MB** | Di atas ini Safari bunuh tab. |
| First Contentful Paint | **< 3 s (4G)** | App shell visible. |
| Boot to MainMenu | **< 5 s (4G), < 10 s (3G)** | |
| Scene transition (load+lazy) | **< 1.5 s** | Pakai loading bar bila > 1 MB. |

### 10.2 Breakdown Initial (Contoh)

| Aset | MB |
|------|----|
| JS bundle (Phaser + game) | 3.5 |
| App shell HTML/CSS | 0.2 |
| WebFont woff2 | 0.3 |
| Common UI atlas (@2x 2048²) | 2.8 |
| Bitmap font atlas | 0.4 |
| Global JSON config | 0.2 |
| Global shader | 0.05 |
| Common SFX AudioSprite | 1.5 |
| **Total** | **~9.0 MB** (aman < 15) |

Sisa ~6 MB buffer untuk runtime cache app shell audio/texture tambahan.

### 10.3 Cara Mengukur

**Transfer size (network):**
- Chrome DevTools → Network tab → "Disable cache" off, reload.
- Lihat "Transferred" di bawah; pastikan < 15 MB initial.
- Gunakan throttling "Slow 4G" / "3G" untuk simulasi.

**RAM/VRAM:**
- Chrome DevTools → Memory → "Performance monitor" → JS heap size, DOM nodes.
- `chrome://gpu` untuk VRAM estimates (tidak akurat, perkiraan).
- Phaser: `this.game.renderer.textures.getTextureKeys().length` tiap transisi.

**FPS / jank:**
- Performance tab → rekam → lihat frame > 16ms (target 60fps).
- GC pauses terlihat sebagai "long task".

**Automated budget check (CI):**
```typescript
// script ci-check-budget.ts
import { readFileSync } from 'fs';
// hitung total dist/assets *.js+*.png+*.json+audio < 15MB
```

### 10.4 Alert & Guardrail

- Bila initial > 15 MB → pecah common atlas atau turunkan resolusi.
- Bila RAM idle naik > 200 MB setelah 10 menit → cek leak (§6).
- Bila scene transition > 2 s → kurangi atlas atau tambah loading bar + compress.

---

## 11. Hubungan Dokumen Lain

Dokumen ini terhubung dengan:

| Dokumen | Hubungan |
|---------|----------|
| **GAME_LOOP.md** | Pooling (§5) dipanggil dari `update()`. Game loop jangan allocate tiap frame; gunakan pool. Memory (§6) dicek saat loop bolak-balik scene. |
| **SPRITESHEET_LAYOUT.md** | Atlas dibuat di sini (grid, margin, packing). Asset Loading konsumsi hasilnya via `load.atlas`/`load.spritesheet` (§3, §4). |
| **AUDIO_PIPELINE.md** | AudioSprite & encode OGG/MP3 di sini. Asset Loading muat via `load.audio`/`load.audioSprite` (§3.1, §4.2). |
| **SAVE_DATA.md** | JSON config versioning (§7.4) sinkron dengan save schema version. Cache busting config butuh koordinasi dengan migrasi save. |
| **PERFORMANCE.md** | Budget & metric (§10) → target FPS. Memory leak (§6) → jank. |
| **PWA_DEPLOY.md** | vite-plugin-pwa config (§9) di-deploy di sini. SW update flow. |

**Alur lintas dokumen:**

```
SPRITESHEET_LAYOUT ─┐
AUDIO_PIPELINE ─────┼─> ASSET_LOADING (load, pool, memory, cache) ─> GAME_LOOP (pakai)
SAVE_DATA ──────────┘                                          └─> PERFORMANCE (ukur)
                                                  PWA_DEPLOY <─ ASSET_LOADING (SW cache)
```

---

## 12. Lampiran: Checklist & Glosarium

### 12.1 Pre-Flight Checklist (setiap rilis)

- [ ] Initial transfer < 15 MB (Network tab, Slow 4G).
- [ ] Per-scene < 5 MB.
- [ ] Semua scene punya `shutdown` unload (§6.2).
- [ ] Tidak ada leak setelah 10× bolak-balik scene (Heap Snapshot, §6.5).
- [ ] Audio di-`stopByKey` + `cache.audio.remove` tiap shutdown.
- [ ] Error fallback ada untuk tiap kategori (§8.3).
- [ ] Retry backoff terpasang (§8.2).
- [ ] Content hash aktif (Vite build, §7.1).
- [ ] SW precache app shell + common; runtime cache scene (§9).
- [ ] Offline boot ke main menu berhasil (§9.5).
- [ ] Pooling aktif untuk projectile/particle/damage number (§5).
- [ ] Budget CI check lewat (§10.3).

### 12.2 Glosarium

| Istilah | Arti |
|---------|------|
| Atlas | Kumpulan sprite dalam 1 gambar besar + json frame. |
| AudioSprite | 1 file audio berisi banyak SFX + region json. |
| Precache | SW simpan aset di disk saat install. |
| Runtime cache | SW simpan aset saat pertama dimuat (SWR). |
| SWR | Stale-While-Revalidate: serve cache, update background. |
| LRU | Least Recently Used: cache buang item paling lama tak dipakai. |
| OOM | Out Of Memory: browser bunuh tab. |
| Context loss | WebGL context hilang (GPU reset / background lama). |
| Content hash | Nama file ber-hash konten untuk cache busting. |
| FOIT/FOUT | Flash Of Invisible/Unstyled Text (font loading). |
| VRAM | Video RAM tempat WebGL texture. |
| Pool | Kumpulan objek reusable, hindari allocate/destroy tiap frame. |

### 12.3 Keputusan Desain (CDP-locked)

- 6 elemen: Fire, Water, Earth, Wind, Light, Dark.
- Rarity: C, R, E, L.
- Scene: Boot, Preload, MainMenu, Battle, Gacha, Guild, Market, HeroDetail, Settings.
- Engine: Phaser 3 + TS + Vite + PWA.
- Audio: OGG + MP3.
- Budget: < 15 MB initial, < 5 MB/scene.
- Renderer: WebGL (fallback Canvas).

---

*Dokumen selesai. Semua keputikan mengikat CDP. Revisi via PR + bump versi di header.*

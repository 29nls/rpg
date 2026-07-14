# AUDIO PIPELINE — RPG Fantasy Turn-Based Idle (Web + PWA)

> **Dokumen:** `AUDIO_PIPELINE.md`
> **Engine:** Phaser 3 + TypeScript + Vite + PWA (vite-plugin-pwa / Workbox)
> **Genre:** RPG Fantasy, Turn-Based, Idle
> **Penulis:** Audio Programmer / Game Engineer
> **Versi:** 1.0
> **Bahasa:** Indonesia

---

## CATATAN EDITOR & ASUMSI EKSPLISIT

Dokumen ini mengikat seluruh parameter desain ke **Canonical Design Parameters (CDP)** yang disepakati tim. Agar dokumen tetap komprehensif tanpa menunggu klarifikasi, asumsi berikut **ditetapkan secara eksplisit** dan wajib dipakai sampai ada revisi tertulis:

| # | Asumsi | Nilai / Keputusan |
|---|--------|-------------------|
| A1 | Target platform | Mobile (Android/iOS Safari) + Desktop Chrome/Edge. Layar utama portrait 720×1280 (logical), desktop skala ulang. |
| A2 | Koneksi rata-rata | 4G (fallback 3G lambat). Pipeline audio harus tahan di 3G. |
| A3 | Budget audio awal (initial load) | **< 3 MB** (master bus + 1 BGM menu + SFX kritis). |
| A4 | Budget per-scene tambahan | **< 2 MB** BGM/lazy audio per scene. |
| A5 | Sistem elemen | 6 elemen: **Fire, Water, Earth, Wind, Light, Dark**. |
| A6 | Rarity hero | **C (Common), R (Rare), E (Epic), L (Legendary)**. |
| A7 | Scene utama | Boot, Preload, MainMenu, Battle, Gacha, Guild, Market, HeroDetail, Settings. |
| A8 | Renderer | WebGL (Phaser.AUTO, fallback Canvas). |
| A9 | Build tool | Vite + vite-plugin-pwa (Workbox GenerateSW mode). |
| A10 | Format audio | **OGG (prioritas)** + **MP3 (fallback)** Safari/iOS lama. |
| A11 | Web Audio API | Dipakai langsung lewat Phaser Sound Manager (`sound: { type: Phaser.WEB_AUDIO }`). |
| A12 | Voice (character voice) | Opsional; **TIDAK** masuk MVP. Diwujudkan sebagai bus terpisah bila diaktifkan. |
| A13 | Voice chat (multiplayer) | **TIDAK** ada di MVP maupun post-MVP v1. Dihindari sepenuhnya. |
| A14 | Toleransi delay SFX | Target **< 20 ms** antara intent suara dan terdengarnya suara (perceptual "instant"). |
| A15 | Autoplay policy | Browser memulai `AudioContext` dalam state `suspended`; butuh **user gesture** untuk `resume()`. |
| A16 | Bahasa UI | Indonesia + Inggris (string di JSON). |
| A17 | Offline | PWA boot offline setelah first visit (precache app shell + audio kritis). |
| A18 | Kanal output | Stereo (2.0). Tanpa spatial/HRTF di MVP. |

> Konstanta kunci (CDP): 6 elemen, rarity C/R/E/L, scene Boot→Preload→MainMenu→Battle/Gacha/Guild/Market/HeroDetail/Settings, format OGG+MP3, BGM per scene, SFX battle/gacha/ui, Voice opsional.

---

## DAFTAR ISI

1. [Tujuan & Arsitektur](#1-tujuan--arsitektur)
2. [Web Audio API Basics](#2-web-audio-api-basics)
3. [Format & Fallback](#3-format--fallback)
4. [Audio Unlock](#4-audio-unlock)
5. [Pre-decode & Preload](#5-pre-decode--preload)
6. [Buses & Volume Matrix](#6-buses--volume-matrix)
7. [Pooling & Concurrency](#7-pooling--concurrency)
8. [Latency Config](#8-latency-config)
9. [Settings & Persistence](#9-settings--persistence)
10. [Error Handling & Edge Case](#10-error-handling--edge-case)
11. [Hubungan Dokumen Lain](#11-hubungan-dokumen-lain)
12. [Lampiran: Kode Lengkap & Glosarium](#12-lampiran-kode-lengkap--glosarium)

---

## 1. Tujuan & Arsitektur

### 1.1 Tujuan Pipeline Audio

Pipeline audio bertanggung jawab memastikan **tidak ada delay terasa** (target < 20 ms, lihat A14) saat aset suara dimainkan, baik BGM, SFX, maupun Voice opsional. Tujuan spesifik:

- **Zero-perceived-latency SFX**: suara aksi battle/gacha/ui terdengar tepat saat event terjadi, tidak "nge-blank" 100–400 ms akibat decode on-the-fly.
- **Stabilitas autoplay**: audio tidak gagal diam-diam karena kebijakan autoplay browser (A15).
- **Skalabilitas**: 6 elemen, puluhan event suara (serangan tiap elemen, gacha reveal, UI click, level-up) tanpa meledakkan memory/HTTP.
- **Kontrol volume granular**: master + 4 bus (Music/SFX/UI/Voice) dengan mute & ducking.
- **Hemat resource**: suspend context di background, recycle instance SFX, kompresor anti-clipping.

### 1.2 Lapisan Arsitektur (Top-down)

Pipeline dibagi menjadi 5 lapisan. Dari atas ke bawah:

```
┌──────────────────────────────────────────────────────────────┐
│  GAME CODE / SCENES (Phaser 3)                                │
│  BattleScene · GachaScene · UIScene · MainMenuScene ...        │
│  ↳ memanggil AudioManager.play('sfx_fire_slash')              │
└───────────────────────────┬──────────────────────────────────┘
                            │  API tinggi (play/stop/setVolume)
                            ▼
┌──────────────────────────────────────────────────────────────┐
│  AUDIO MANAGER (singleton TS, pembungkus Phaser)               │
│  - registri kunci audio, pool, unlock, ducking, persist       │
└───────────────────────────┬──────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────┐
│  PHASER SOUND MANAGER  (this.sound)  — pembungkus Web Audio    │
│  - load audio / audiosprite, add(key), play, polygons         │
└───────────────────────────┬──────────────────────────────────┘
                            │  GainGraph (bus)
                            ▼
┌──────────────────────────────────────────────────────────────┐
│  WEB AUDIO API (AudioContext + node graph)                     │
│                                                                │
│   [decoded AudioBuffer]                                        │
│        │  AudioBufferSourceNode                                │
│        ▼                                                       │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐        │
│   │ Music   │   │  SFX    │   │   UI    │   │  Voice  │  (bus)  │
│   │ GainNode│   │ GainNode│   │ GainNode│   │ GainNode│        │
│   └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘        │
│        └─────────────┴─────────────┴─────────────┘            │
│                          ▼                                     │
│                   ┌──────────────┐   (Master Gain + Compressor) │
│                   │  MASTER BUS  │                              │
│                   └──────┬───────┘                              │
│                          ▼                                     │
│                  AudioContext.destination ──▶ speaker/headphone│
└──────────────────────────────────────────────────────────────┘
                            ▲
                            │
┌──────────────────────────────────────────────────────────────┐
│  DECODER / PRE-DECODE CACHE + AUDIO POOL                       │
│  - decodeAudioData saat preload → buffer siap                 │
│  - pool instance SFX (maks 16) untuk recycle                   │
└──────────────────────────────────────────────────────────────┘
```

**Penjelasan aliran:**
1. Game code memanggil API tinggi (`AudioManager.play`) — tidak menyentuh Web Audio langsung.
2. `AudioManager` mengecek unlock, ambil instance dari pool, atur ducking bila perlu.
3. Phaser Sound Manager menerjemahkan ke Web Audio (membuat `AudioBufferSourceNode` dari buffer yang sudah didecode).
4. Sinyal mengalir `source → busGain → masterGain(+compressor) → destination`.
5. Decoder & pool berada di bawah agar `play()` tidak memicu decode baru (sumber delay utama).

### 1.3 Komponen Inti

| Komponen | Tanggung jawab | Catatan CDP |
|----------|----------------|-------------|
| `AudioManager` | Singleton pembungkus. Registri, unlock, bus, pool, ducking, persist. | `src/audio/AudioManager.ts` |
| `Phaser.SoundManager` | Wrapper Web Audio bawaan Phaser. `this.sound`. | `type: Phaser.WEB_AUDIO` |
| `AudioContext` | Engine audio browser. `state`, `resume()`, `decodeAudioData`. | 1 per game |
| `BusGain` | `GainNode` per bus (Music/SFX/UI/Voice) + master. | 5 node |
| `DecodeCache` | `Map<string, AudioBuffer>` hasil pre-decode. | di memori |
| `SfxPool` | Pool instance SFX (maks 16) untuk recycle. | cegah clipping |

---

## 2. Web Audio API Basics

### 2.1 Apa itu AudioContext

`AudioContext` adalah "otak" Web Audio API. Ia memegang:

- `sampleRate` — biasanya 44100 atau 48000 Hz.
- `currentTime` — jam waktu audio (detik, monoton, presisi tinggi). Dasar scheduling.
- `state` — `'suspended'` | `'running'` | `'closed'`.
- `destination` — node output akhir (speaker).
- `baseLatency` / `outputLatency` — estimasi latency (lihat §8).
- Method `createGain()`, `createBufferSource()`, `createDynamicsCompressor()`, `decodeAudioData()`, `resume()`, `suspend()`.
- Opsi konstruktor `latencyHint` — petunjuk ke browser seberapa rendah latency yang diinginkan.

```ts
// Konstruksi dasar (Phaser sudah membuat ini; ini ilustrasi level rendah)
const ctx = new (window.AudioContext || (window as any).webkitAudioContext)({
  latencyHint: 'interactive', // 'interactive' | 'balanced' | 'playback'
});
console.log(ctx.state);       // 'suspended' (belum di-unlock)
console.log(ctx.sampleRate);  // 48000 mis.
```

### 2.2 Node Graph: source → gain → destination

Web Audio berbasis **node graph** (graf berarah). Sinyal mengalir dari node sumber ke node tujuan melalui node pemrosesan.

```ts
// Pola kanonik satu suara
const buffer: AudioBuffer = ...;            // sudah didecode
const src = ctx.createBufferSource();        // node sumber
src.buffer = buffer;

const gain = ctx.createGain();               // node penguat (volume)
gain.gain.value = 0.9;

src.connect(gain);                           // source -> gain
gain.connect(ctx.destination);               // gain -> output

src.start(0);                                // mainkan seketika (pada currentTime)
```

Untuk pipeline kita, `gain` diganti dengan **bus gain** (§6), lalu bus gain menyambung ke **master gain + compressor**, baru ke `destination`.

### 2.3 Mengapa Pre-decode Penting (Sumber Delay Utama)

`decodeAudioData(arrayBuffer)` adalah operasi **asinkron & berat CPU**. Jika kita decode saat pertama kali memainkan suara:

```
t=0  event terjadi (tombol ditekan)
t=0  mulai decodeAudioData(ogg)  ← blocking-ish, main-thread, 50–300 ms
t=+X selesai decode → baru bisa start()
t=+X  suara terdengar  (terlambat, "blank")
```

Dengan **pre-decode saat preload**, buffer sudah ada di memori:

```
t=0  event terjadi
t=0  src.start() langsung → suara terdengar < 20 ms
```

Ini alasan utama pipeline kita melakukan decode di scene `Preload` (§5), bukan di `play()`.

### 2.4 Phaser Sound Manager sebagai Pembungkus

Phaser menyediakan `this.sound` (instance `Phaser.Sound.WebAudioSoundManager`) yang membungkus `AudioContext`. Konfigurasi wajib di `game config`:

```ts
// src/main.ts
import Phaser from 'phaser';

const config: Phaser.Types.Core.GameConfig = {
  type: Phaser.AUTO,
  audio: {
    disableWebAudio: false,       // WAJIB false → pakai Web Audio
    type: Phaser.WEB_AUDIO,       // eksplisit pakai Web Audio API
    noAudio: false,
  },
  // ... scene, scale, dll
};

new Phaser.Game(config);
```

> **Peringatan:** jangan pakai `Phaser.HTML5_AUDIO` untuk SFX. HTML5 `<audio>` streaming menambah latency besar dan tidak bisa dipool dengan baik. Web Audio (`WEB_AUDIO`) wajib untuk zero-latency SFX.

---

## 3. Format & Fallback

### 3.1 OGG sebagai Primer, MP3 sebagai Fallback

| Format | Keunggulan | Kelemahan | Dukungan |
|--------|-----------|-----------|----------|
| **OGG (Vorbis)** | File paling kecil (30–50% dari MP3), decode cepat, kualitas baik | Tidak didukung Safari/iOS lama (<14.5 untuk OGG umum) | Chrome, Edge, Firefox, Android WebView ✅ |
| **MP3** | Dukungan universal | File lebih besar, decode sedikit lebih lambat | Semua browser termasuk Safari/iOS ✅ |

**Keputusan (A10):** OGG dipakai dulu; Phaser otomatis fallback ke MP3 bila browser tidak bisa memainkan OGG (lewat `canPlay`).

### 3.2 Cara Phaser Memilih Format

Phaser `loader.audio` menerima **array** berisi file dengan ekstensi berbeda. Phaser akan memilih file pertama yang bisa diputar browser (`canPlay`):

```ts
// src/scenes/PreloadScene.ts
preload(): void {
  // Phaser pilih OGG bila didukung, else MP3
  this.load.audio('bgm_battle', [
    'audio/bgm/battle.ogg',
    'audio/bgm/battle.mp3',
  ]);

  this.load.audio('sfx_ui_click', [
    'audio/sfx/ui_click.ogg',
    'audio/sfx/ui_click.mp3',
  ]);
}
```

Untuk **AudioSprite** (§5), polanya sama tapi lewat `loader.audiosprite`:

```ts
this.load.audiosprite('sfx_sprite', [
  'audio/sfx/sprite.ogg',
  'audio/sfx/sprite.mp3',
], null, 'audio/sfx/sprite.json');
```

### 3.3 Deteksi `canPlay`

Phaser Sound Manager menyediakan deteksi capability:

```ts
// Mengecek dukungan codec (berguna untuk log/telemetri)
const sm = this.sound as Phaser.Sound.WebAudioSoundManager;
console.log('OGG :', sm.canPlayAudio('ogg'));   // true di Chrome
console.log('MP3 :', sm.canPlayAudio('mp3'));   // true di semua
console.log('OPUS:', sm.canPlayAudio('opus'));  // opsional
```

Codec support matrix (ringkas):

| Browser | OGG | MP3 | Catatan |
|---------|-----|-----|---------|
| Chrome/Edge (Android/Desktop) | ✅ | ✅ | OGG dipakai |
| Firefox | ✅ | ✅ | OGG dipakai |
| Safari 14.5+ (iOS/Mac) | ✅ | ✅ | OGG aman; MP3 fallback |
| Safari < 14.5 (lama) | ❌ | ✅ | MP3 fallback otomatis |
| Android WebView | ✅ | ✅ | OGG dipakai |

### 3.4 Konvensi Penamaan File

- BGM: `audio/bgm/<scene>.<ext>` — mis. `mainmenu.ogg`, `battle.ogg`, `gacha.ogg`.
- SFX tunggal (jarang): `audio/sfx/<nama>.<ext>`.
- AudioSprite SFX: `audio/sfx/sprite.<ext>` + `sprite.json`.
- Voice (opsional): `audio/voice/<heroId>_<line>.<ext>`.

---

## 4. Audio Unlock

### 4.1 Masalah Autoplay Policy

Browser modern (Chrome 71+, Safari, iOS) **memblokir audio otomatis**. `AudioContext` dibuat dalam state `suspended` dan **hanya boleh `resume()` dari dalam event gesture user** (pointerdown, keydown, touchstart). Memanggil `resume()` di luar gesture akan diabaikan.

```ts
const ctx = new AudioContext();
console.log(ctx.state); // 'suspended' — belum boleh bunyi
```

### 4.2 Strategi Unlock: Pertama Kali Gesture

Kita pasang **listener sekali-pakai** di `BootScene` atau `MainMenuScene` yang memanggil `ctx.resume()` saat user pertama kali berinteraksi. Setelah berhasil, lepas listener.

```ts
// src/audio/AudioUnlock.ts
export class AudioUnlock {
  private static done = false;

  static attach(ctx: AudioContext, onUnlocked?: () => void): void {
    if (AudioUnlock.done) return;

    const unlock = async () => {
      if (ctx.state === 'suspended') {
        try { await ctx.resume(); } catch { /* abaikan */ }
      }
      if (ctx.state === 'running') {
        AudioUnlock.done = true;
        onUnlocked?.();
        // lepas semua listener (sekali pakai)
        window.removeEventListener('pointerdown', unlock);
        window.removeEventListener('keydown', unlock);
        window.removeEventListener('touchstart', unlock);
      }
    };

    // Coba juga saat focus (beberapa browser perlu)
    window.addEventListener('pointerdown', unlock, { once: false });
    window.addEventListener('keydown', unlock, { once: false });
    window.addEventListener('touchstart', unlock, { once: false });
  }

  static get isUnlocked(): boolean {
    return AudioUnlock.done;
  }
}
```

### 4.3 Integrasi dengan Phaser

Phaser punya mekanisme unlock internal (`sound.on('unlocked')`), tapi kita **tetap pasang unlock eksplisit** agar deterministik dan terukur (terutama di PWA/iOS). Gabungan:

```ts
// src/scenes/BootScene.ts
create(): void {
  const sm = this.sound as Phaser.Sound.WebAudioSoundManager;
  const ctx = sm.context; // AudioContext bawaan Phaser

  AudioUnlock.attach(ctx, () => {
    console.log('[audio] unlocked & running');
    // optional: mulai BGM menu di sini, atau biarkan scene yg urus
  });

  // Lanjut ke Preload (jangan mainkan audio sebelum unlock!)
  this.scene.start('Preload');
}
```

### 4.4 Aturan Emas: JANGAN Mainkan Sebelum Unlock

```ts
// Salah — bisa diam/error di iOS
this.sound.play('bgm_menu'); // dipanggil sebelum gesture

// Benar — guard dengan flag unlock
playBgm(scene: Phaser.Scene, key: string): void {
  if (!AudioUnlock.isUnlocked) {
    // antre: mainkan setelah unlock
    pendingBgm = key;
    return;
  }
  scene.sound.play(key, { loop: true });
}
```

> **Pola antrean:** simpan intent BGM/SFX yang terjadi sebelum unlock, lalu jalankan saat `onUnlocked`. Untuk SFX UI sebelum unlock (tombol pertama), cukup diabaikan (user belum dengar apa-apa).

### 4.5 Unlock di PWA / iOS

- iOS Safari **sangat strikt**: gesture harus "native" (pointerdown/keydown riil), tidak dari `setTimeout`/`Promise` terpisah.
- Jika game di-load dari **add-to-home PWA**, `resume()` tetap butuh gesture pertama.
- Hindari memanggil `resume()` di dalam `requestAnimationFrame` tanpa gesture — diabaikan.

---

## 5. Pre-decode & Preload

### 5.1 Decode Saat Preload → play() Instan

Semua audio **harus** di-load & di-decode di `PreloadScene` (atau di scene terkait secara lazy). Phaser otomatis mendecode saat `loader.audio` selesai. Hasil decode disimpan di cache internal Phaser (`sound` manager cache), sehingga `play()` berikutnya instan.

```ts
// src/scenes/PreloadScene.ts
export class PreloadScene extends Phaser.Scene {
  constructor() { super('Preload'); }

  preload(): void {
    const bar = this.add.graphics();
    this.load.on('progress', (p: number) => { /* update bar */ });

    // --- BGM per scene ---
    for (const s of ['mainmenu', 'battle', 'gacha', 'guild', 'market', 'herodetail']) {
      this.load.audio(`bgm_${s}`, [`audio/bgm/${s}.ogg`, `audio/bgm/${s}.mp3`]);
    }

    // --- SFX kritis (juga di AudioSprite di bawah) ---
    // ...

    // --- AudioSprite: satu file banyak marker ---
    this.load.audiosprite('sfx', [
      'audio/sfx/sprite.ogg',
      'audio/sfx/sprite.mp3',
    ], null, 'audio/sfx/sprite.json');
  }

  create(): void {
    // semua sudah didecode → pindah ke MainMenu
    this.scene.start('MainMenu');
  }
}
```

### 5.2 AudioSprite untuk SFX Kecil

**AudioSprite** = satu file audio berisi banyak SFX kecil yang dipisahkan oleh *marker* (json). Keuntungan:

- **Satu HTTP request** untuk puluhan SFX → hemat koneksi 3G (A2).
- **Satu decode** untuk seluruh set → hemat CPU.
- Marker dijson menyimpan `start`/`end` (detik) tiap SFX.

```json
// audio/sfx/sprite.json
{
  "resources": ["sprite.ogg", "sprite.mp3"],
  "spritemap": {
    "ui_click":       { "start": 0.000, "end": 0.120, "loop": false },
    "ui_hover":       { "start": 0.140, "end": 0.260, "loop": false },
    "ui_confirm":     { "start": 0.280, "end": 0.520, "loop": false },
    "gacha_open":     { "start": 0.540, "end": 1.200, "loop": false },
    "gacha_reveal_r": { "start": 1.220, "end": 2.000, "loop": false },
    "gacha_reveal_l": { "start": 2.020, "end": 3.000, "loop": false },
    "fire_slash":     { "start": 3.020, "end": 3.640, "loop": false },
    "water_slash":    { "start": 3.660, "end": 4.280, "loop": false },
    "earth_slash":    { "start": 4.300, "end": 4.920, "loop": false },
    "wind_slash":     { "start": 4.940, "end": 5.560, "loop": false },
    "light_slash":    { "start": 5.580, "end": 6.200, "loop": false },
    "dark_slash":     { "start": 6.220, "end": 6.840, "loop": false },
    "hit_crit":       { "start": 6.860, "end": 7.300, "loop": false },
    "level_up":       { "start": 7.320, "end": 8.600, "loop": false },
    "coin":           { "start": 8.620, "end": 8.900, "loop": false }
  }
}
```

Memainkan marker dari sprite:

```ts
// this.sound.play('sfx', { sprite: 'fire_slash' })
scene.sound.play('sfx', { sprite: 'gacha_reveal_l', volume: 0.9 });
```

### 5.3 Pre-decode Manual (Opsional, untuk SFX Super-Kritis)

Bila ingin kontrol penuh (decode di luar loader Phaser), gunakan `decodeAudioData` langsung dan simpan ke `Map`:

```ts
// src/audio/DecodeCache.ts
export class DecodeCache {
  private buffers = new Map<string, AudioBuffer>();
  constructor(private ctx: AudioContext) {}

  async decode(key: string, url: string): Promise<void> {
    if (this.buffers.has(key)) return;
    const res = await fetch(url);
    const arr = await res.arrayBuffer();
    const buf = await this.ctx.decodeAudioData(arr);
    this.buffers.set(key, buf);
  }

  get(key: string): AudioBuffer | undefined { return this.buffers.get(key); }
}
```

> **Rekomendasi:** untuk MVP, cukup andalkan `loader.audiosprite` + `loader.audio` (Phaser sudah decode otomatis). `DecodeCache` manual hanya bila butuh SFX di-generate prosedural atau dimuat sangat dinamis.

### 5.4 Budget & Kategori Preload

| Kategori | Kapan di-load | Contoh | Budget (A3/A4) |
|----------|--------------|--------|----------------|
| Audio kritis (initial) | `PreloadScene` | 1 BGM menu, SFX sprite (ui/gacha dasar) | < 3 MB |
| BGM per scene | lazy saat masuk scene | `bgm_battle` saat `BattleScene` | < 2 MB/scene |
| SFX battle dinamis | saat `BattleScene` create | slash tiap elemen, crit | dalam sprite |
| Voice (opsional) | saat `HeroDetail`/dialog | hanya bila fitur aktif | on-demand |

---

## 6. Buses & Volume Matrix

### 6.1 Pembagian Gain Node per Bus

Kita buat **5 `GainNode`**: 1 master + 4 bus (Music, SFX, UI, Voice). Tiap suara di-rute ke busnya via opsi `sound.add(key, { ... })` atau dengan mengatur output node.

Phaser `WebAudioSoundManager` tidak expose bus gain secara langsung per-key, sehingga kita **buat node graph sendiri** dan sambungkan lewat `detune`/`rate`? Tidak. Cara bersih: kita kelola volume per kategori lewat `sound.volume` global dan `sound.setVolume` per-sound, lalu terapkan matrix kita sendiri. Namun untuk ducking & bus sejati, kita bangun graph Web Audio ekstra:

```ts
// src/audio/BusGraph.ts
import Phaser from 'phaser';

export type BusId = 'master' | 'music' | 'sfx' | 'ui' | 'voice';

export class BusGraph {
  readonly ctx: AudioContext;
  private gains: Record<BusId, GainNode>;

  constructor(ctx: AudioContext) {
    this.ctx = ctx;
    // buat node
    const master = ctx.createGain();
    const music  = ctx.createGain();
    const sfx    = ctx.createGain();
    const ui     = ctx.createGain();
    const voice  = ctx.createGain();

    // bus -> master
    music.connect(master);
    sfx.connect(master);
    ui.connect(master);
    voice.connect(master);

    // master -> compressor -> destination (anti clipping, lihat §7)
    const comp = ctx.createDynamicsCompressor();
    master.connect(comp);
    comp.connect(ctx.destination);

    this.gains = { master, music, sfx, ui, voice };
  }

  setVolume(bus: BusId, v: number): void {
    // ramping halus (20ms) untuk hindari "pop"
    const g = this.gains[bus];
    g.gain.setTargetAtTime(Phaser.Math.Clamp(v, 0, 1), this.ctx.currentTime, 0.02);
  }

  getVolume(bus: BusId): number { return this.gains[bus].gain.value; }
}
```

> **Catatan integrasi Phaser:** Phaser `WEB_AUDIO` membuat `AudioContext` sendiri dan mengalirkan ke `destination` langsung. Untuk bypass ke bus kita, opsi paling andal di MVP: **kelola volume di level Phaser** (`this.sound.volume` = master; tiap `sound` object punya `.volume` untuk bus) dan gunakan `BusGraph` hanya sebagai metadata matrix + ducking logic yang mengatur `this.sound` volume. Detail kawatan di §6.3.

### 6.2 Volume Matrix (Default)

| Bus | Default Volume | Bisa Mute | Ducking | Keterangan |
|-----|---------------|-----------|---------|------------|
| **Master** | `0.80` | Ya | — | Pengali total semua bus. |
| **Music (BGM)** | `0.50` | Ya | Di-duck ke `0.25` saat SFX penting / Voice | BGM per scene (§11). |
| **SFX (battle/gacha)** | `0.90` | Ya | Memicu duck Music saat aktif | Pool 16 instance (§7). |
| **UI** | `0.70` | Ya | Tidak | Click/hover/confirm. |
| **Voice (opsional)** | `1.00` | Ya | Duck Music ke `0.20` saat bicara | Non-MVP (A12). |

```ts
// Nilai default tersimpan di Settings (§9)
export const DEFAULT_VOLUMES: Record<BusId, number> = {
  master: 0.80,
  music:  0.50,
  sfx:    0.90,
  ui:     0.70,
  voice:  1.00,
};
```

### 6.3 Implementasi Matrix di Phaser (Tanpa Node Ekstra)

Karena kita pakai Phaser `WEB_AUDIO`, pendekatan praktis untuk MVP:

```ts
// src/audio/Bus.ts
export class Bus {
  private _volume: number;
  private _muted = false;
  constructor(
    private manager: Phaser.Sound.WebAudioSoundManager,
    public readonly id: BusId,
    defaultVolume: number,
  ) { this._volume = defaultVolume; }

  // Terapkan ke seluruh suara di bus ini
  apply(sounds: Phaser.Sound.WebAudioSound[]): void {
    const v = this._muted ? 0 : this._volume;
    for (const s of sounds) if (s.busId === this.id) s.setVolume(v);
  }

  set volume(v: number) { this._volume = v; }
  get volume(): number { return this._muted ? 0 : this._volume; }
  set muted(m: boolean) { this._muted = m; }
  get muted(): boolean { return this._muted; }
}
```

Atau, lebih sederhana untuk MVP: gunakan **satu master volume global** (`this.sound.volume`) + **volume relatif per `play()`**:

```ts
// Contoh mainkan SFX dengan volume bus SFX (0.9) * master (0.8)
playSfx(scene: Phaser.Scene, key: string, marker?: string): void {
  const master = settings.master;   // 0.8
  const sfxV   = settings.sfx;      // 0.9
  scene.sound.play(key, {
    sprite: marker,
    volume: master * sfxV,
  });
}
```

### 6.4 Ducking (Pelemahan BGM saat SFX/Voice Penting)

**Ducking** = menurunkan volume BGM sementara saat ada SFX penting (crit, gacha reveal) atau Voice. Tujuannya: fokus pendengaran ke event krusial tanpa mematikan musik.

```ts
// src/audio/Ducking.ts
export class Ducking {
  private timer?: number;

  constructor(
    private bus: BusGraph,
    private musicDefault = 0.50,
    private musicDucked  = 0.25,
  ) {}

  duck(reason: 'sfx' | 'voice', holdMs = 600): void {
    const target = reason === 'voice' ? 0.20 : this.musicDucked;
    this.bus.setVolume('music', target);

    if (this.timer) clearTimeout(this.timer);
    this.timer = window.setTimeout(() => {
      this.bus.setVolume('music', this.musicDefault);
    }, holdMs);
  }
}
```

> **CDP trigger ducking:** `gacha_reveal_l` (Legendary reveal), `hit_crit`, line `voice` (bila diaktifkan). BGM kembali naik setelah ~600 ms (ramp halus via `setTargetAtTime`).

---

## 7. Pooling & Concurrency

### 7.1 Batas Instance SFX Bersamaan

Mainkan terlalu banyak SFX sekaligus → **clipping/distorsi** & beban CPU. Kita batasi instance SFX aktif maksimal **16** (CDP asumsi). Melebihi → instance terlama di-recycle (stop paksa) atau diabaikan (SFX prioritas rendah seperti `coin`).

```ts
// src/audio/SfxPool.ts
export class SfxPool {
  private active = new Map<string, Phaser.Sound.WebAudioSound>();
  constructor(private scene: Phaser.Scene, private limit = 16) {}

  play(key: string, config: Phaser.Types.Sound.SoundConfig = {}): void {
    if (this.active.size >= this.limit) {
      // recycle: stop SFX tertua (bukan yang sedang critical)
      const oldest = this.active.keys().next().value;
      if (oldest) this.stop(oldest);
    }
    const s = this.scene.sound.add(key, { ...config, volume: config.volume ?? 0.9 });
    s.once('complete', () => this.release(key + s.name));
    s.play();
    this.active.set(key + (s as any).name, s);
  }

  private release(id: string): void { this.active.delete(id); }
  stop(id: string): void { this.active.get(id)?.stop(); this.active.delete(id); }
}
```

### 7.2 Recycle & Hindari Kliping

- **Recycle**: instance SFX yang selesai (`complete`) dikembalikan ke pool, bukan dibuang. Phaser `WEB_AUDIO` sudah reuse `AudioBufferSourceNode` secara internal; kita cukup hindari `add()` berulang tanpa `destroy()`.
- **Limiter**: pasang `DynamicsCompressorNode` di master (§6.1) agar puncak gabungan SFX+BGM tidak melebihi 0 dBFS (cegah clipping hardware/software).
- **Voice chat**: **TIDAK** ada di MVP maupun v1 (A13). Tidak perlu kode WebRTC/MediaStream.

```ts
// Compressor di master (anti clipping)
const comp = ctx.createDynamicsCompressor();
comp.threshold.value = -10;  // dB
comp.knee.value      = 20;
comp.ratio.value     = 12;
comp.attack.value    = 0.003;
comp.release.value  = 0.25;
master.connect(comp);
comp.connect(ctx.destination);
```

### 7.3 Prioritas SFX

| Prioritas | SFX | Bila pool penuh |
|-----------|-----|-----------------|
| Tinggi | `hit_crit`, `gacha_reveal_*`, `level_up` | Recycle SFX lama segera |
| Sedang | `fire_slash`..`dark_slash` (6 elemen) | Recycle bila perlu |
| Rendah | `coin`, `ui_hover`, `ui_click` | Abaikan (drop) bila penuh |

---

## 8. Latency Config

### 8.1 latencyHint 'interactive'

Gunakan `latencyHint: 'interactive'` agar browser memprioritaskan latency rendah (bukan kualitas buffering).

```ts
// Phaser membuat context; untuk kontrol penuh, kita override opsi context:
const ctx = new AudioContext({ latencyHint: 'interactive' });
```

Di Phaser, kita akses context yang sudah dibuat dan **tidak bisa ubah latencyHint setelah konstruksi**, jadi pastikan config `audio.type = WEB_AUDIO` (default sudah interactive-ish). Untuk menjamin, kita bisa membuat context sendiri dan injeksi — namun di MVP cukup andalkan default + pre-decode (§5).

### 8.2 Jangan Streaming File Besar

- **JANGAN** pakai `<audio>`/HTML5 untuk BGM besar secara streaming → latency + jeda.
- Gunakan `AudioBufferSourceNode` (buffer di RAM) yang sudah di-decode penuh di preload.
- BGM di-loop via `sound.play(key, { loop: true })` — Phaser menangani loop di buffer, bukan stream.

### 8.3 Schedule Tepat Waktu

Mainkan SFX **segera** (`src.start(ctx.currentTime)`). Untuk sinyal tempo (beat BGM sync), pakai `currentTime` + offset:

```ts
// Jadwal SFX pada waktu presisi (contoh: SFX seirama dengan beat)
const when = ctx.currentTime + 0.0; // sekarang
src.start(when);
```

Jangan menunda `play()` dengan `setTimeout` longgar untuk SFX responsif — selalu panggil langsung dari event handler (yang juga memicu unlock, §4).

### 8.4 Cara Mengukur Latency

```ts
function measureLatency(ctx: AudioContext): void {
  const base = (ctx as any).baseLatency ?? 0;          // latency dalam pipeline
  const out  = (ctx as any).outputLatency ?? 0;        // latency ke hardware
  const total = (base + out) * 1000;                   // ms
  console.log(`[audio] estimated latency: ${total.toFixed(1)} ms`);
  // Target: < 20 ms (A14). Bila > 40 ms, periksa latencyHint & device.
}
```

| Metrik | Sumber | Arti |
|--------|--------|------|
| `baseLatency` | `AudioContext` | Waktu proses (decode→output) internal. |
| `outputLatency` | `AudioContext` | Waktu ke speaker/hardware. |
| `currentTime` | `AudioContext` | Jam audio untuk scheduling. |

---

## 9. Settings & Persistence

### 9.1 Skema Save Data (localStorage)

Setelan audio disimpan bersama save data player di `localStorage`. Kunci: `rpg.save.v1`.

```json
{
  "audio": {
    "master": 0.80,
    "music":  0.50,
    "sfx":    0.90,
    "ui":     0.70,
    "voice":  1.00,
    "muted":  false
  }
}
```

```ts
// src/audio/AudioSettings.ts
export interface AudioSettings {
  master: number;
  music:  number;
  sfx:    number;
  ui:     number;
  voice:  number;
  muted:  boolean;
}

const KEY = 'rpg.save.v1';

export function loadAudioSettings(): AudioSettings {
  try {
    const raw = localStorage.getItem(KEY);
    if (raw) {
      const data = JSON.parse(raw);
      if (data?.audio) return { ...DEFAULT_VOLUMES, ...data.audio };
    }
  } catch { /* corrupt → default */ }
  return { ...DEFAULT_VOLUMES, muted: false };
}

export function saveAudioSettings(s: AudioSettings): void {
  try {
    const raw = localStorage.getItem(KEY);
    const data = raw ? JSON.parse(raw) : {};
    data.audio = s;
    localStorage.setItem(KEY, JSON.stringify(data));
  } catch { /* quota/disabled → abaikan */ }
}
```

### 9.2 Terapkan Saat Boot

```ts
// src/scenes/BootScene.ts (create)
const settings = loadAudioSettings();
audioManager.applySettings(settings); // set semua bus + master.muted
saveAudioSettings(settings);          // pastikan persist
```

### 9.3 Mute & Respek OS / Tab Mute

- **Mute global**: `settings.muted = true` → semua bus volume 0 (tanpa mengubah nilai tersimpan tiap bus, agar unmute mengembalikan).
- **Tab mute (browser)**: bila user men-mute tab lewat UI browser, kita **tidak bisa** mendeteksi langsung, tapi kita bisa mengamati `ctx.state` berubah menjadi `suspended` tak terduga → anggap "dimute eksternal", hentikan BGM, tunggu resume.
- **OS volume**: di luar kendali JS; kita hanya hormati `ctx.state`.

```ts
// Respek perubahan state context (tab mute / OS)
ctx.onstatechange = () => {
  if (ctx.state === 'suspended') {
    // bisa jadi tab dimute / backgrounded → hentikan scheduling BGM
    audioManager.onContextSuspended();
  } else if (ctx.state === 'running') {
    audioManager.onContextResumed();
  }
};
```

---

## 10. Error Handling & Edge Case

### 10.1 Audio Context Terblokir

- **Gejala**: `ctx.state === 'suspended'` terus meski sudah gesture.
- **Penanganan**: pasang ulang listener unlock (§4.2), tunggu gesture berikutnya. Jangan crash game.
- **Fallback**: bila `WEB_AUDIO` gagal total (device langka), Phaser otomatis fallback ke `HTML5_AUDIO` bila `disableWebAudio: false` dan Web Audio tidak ada. Kita tetap set `type: WEB_AUDIO` tapi tangkap error.

```ts
try {
  const sm = this.sound as Phaser.Sound.WebAudioSoundManager;
  if (!sm.context) throw new Error('no web audio');
} catch (e) {
  console.warn('[audio] Web Audio unavailable, HTML5 fallback', e);
}
```

### 10.2 Headphone Unplug

- **Fakta**: tidak ada event JS standar untuk "headphone dicabut" (kecuali sedikit lewat `AudioContext` state berubah di beberapa device).
- **Penanganan**: andalkan behavior browser (audio biasanya pindah ke speaker). Kita **tidak** menghentikan audio. Bila `ctx.state` berubah aneh, ikuti §10.1.
- Catatan: jangan asumsikan bisa deteksi; jangan tambah kode kompleks yang tidak reliable.

### 10.3 Tab Background (Suspend untuk Hemat)

Idle game sering di-background. `requestAnimationFrame` di-throttle, tapi `AudioContext` tetap jalan → boros baterai.

```ts
// src/audio/VisibilityHandler.ts
export function attachVisibility(ctx: AudioContext, onSuspend: () => void, onResume: () => void): void {
  document.addEventListener('visibilitychange', () => {
    if (document.hidden) {
      // background → suspend context (hemat CPU/baterai)
      ctx.suspend().catch(() => {});
      onSuspend();
    } else {
      // foreground → resume (perlu gesture? tidak, resume dari visibility diizinkan
      // bila sebelumnya sudah pernah di-unlock)
      ctx.resume().catch(() => {});
      onResume();
    }
  });
}
```

- Saat `hidden`: `suspend()` → BGM & SFX berhenti (tidak terdengar di background, sesuai harapan user).
- Saat `visible` kembali: `resume()` → lanjutkan BGM (Phaser loop otomatis lanjut).
- **Penting**: idle accrual (GAME_STATE_MACHINE A18) tetap jalan di server/client timer, tidak bergantung audio.

### 10.4 Context Loss / Closed

- **Gejala**: `ctx.state === 'closed'` (beberapa browser menutup context saat tab lama ditidurkan) atau suara tiba-tiba mati.
- **Penanganan**: deteksi `closed`, buat `AudioContext` baru, rebuild `BusGraph`, reload buffer dari cache Phaser.

```ts
// src/audio/ContextGuard.ts
export async function ensureContext(oldsm: Phaser.Sound.WebAudioSoundManager): Promise<AudioContext> {
  const ctx = (oldsm as any).context as AudioContext;
  if (ctx.state === 'closed') {
    const fresh = new (window.AudioContext || (window as any).webkitAudioContext)({ latencyHint: 'interactive' });
    (oldsm as any).context = fresh;
    return fresh;
  }
  if (ctx.state === 'suspended') await ctx.resume().catch(() => {});
  return ctx;
}
```

### 10.5 Ringkasan Edge Case

| Skenario | Deteksi | Tindakan |
|----------|---------|----------|
| Context blocked (autoplay) | `state==='suspended'` | unlock di gesture (§4) |
| Headphone unplug | tidak detectable | biarkan browser, jangan crash |
| Tab background | `visibilitychange.hidden` | `suspend()` context |
| Tab foreground | `visibilitychange.visible` | `resume()` context |
| Context closed/lost | `state==='closed'` | buat context baru, rebuild bus |
| Tab muted (browser) | `state` anomali | hentikan scheduling, tunggu resume |
| Decode gagal (file corrupt) | `catch` di loader | log + skip SFX, jangan crash game |
| OGG tidak didukung | `canPlay('ogg')===false` | Phaser fallback MP3 otomatis |

---

## 11. Hubungan Dokumen Lain

Pipeline audio terhubung erat dengan dokumen desain lain. Tabel ini memetakan integrasi:

| Dokumen | Hubungan dengan Audio Pipeline |
|---------|-------------------------------|
| **ASSET_LOADING.md** | Preload audio (§5) & AudioSprite (§5.2) mengikuti strategi preload/lazy di sana. Budget audio (A3/A4) diambil dari sana. Cache busting & PWA precache audio kritis dikelola di sana. |
| **SPRITESHEET_LAYOUT.md** | AudioSprite adalah **analog audio** dari spritesheet: satu file + marker/json, bukan banyak file. Penamaan marker mengikuti konvensi nama di sana. |
| **GAME_STATE_MACHINE.md** | BGM dipilih **per state** (Boot/MainMenu/Battle/Gacha/dst, A7). Transisi state → ganti BGM + (opsional) ducking. Pause/Maintenance → `suspend()` context (§10.3). |
| **GAME_LOOP.md** | `scene.update` memanggil `audioManager.tick()` untuk schedule SFX tepat waktu (§8.3) & cek pool (§7). Loop dijeda saat Paused → audio ikut `suspend()`. |
| **PRD.md / SRS.md** | Menetapkan kebutuhan: 6 elemen (SFX slash per elemen), gacha reveal, voice opsional. Menjadi sumber daftar event suara. |
| **ERD.md / TDD.md** | Save data audio (§9) tersimpan di struktur player save (localStorage). Test audio ada di TDD (matcher `play()` tanpa error, unlock flow). |

### 11.1 Pemetaan Event Suara (CDP)

Berdasarkan 6 elemen (A5) & rarity (A6):

| Event | Marker AudioSprite | Bus | Ducking? |
|-------|-------------------|-----|----------|
| UI click | `ui_click` | UI | Tidak |
| UI hover | `ui_hover` | UI | Tidak |
| UI confirm | `ui_confirm` | UI | Tidak |
| Gacha open | `gacha_open` | SFX | Tidak |
| Gacha reveal R/E | `gacha_reveal_r` | SFX | Tidak |
| Gacha reveal L | `gacha_reveal_l` | SFX | **Ya** (duck BGM) |
| Serangan Fire | `fire_slash` | SFX | Tidak |
| Serangan Water | `water_slash` | SFX | Tidak |
| Serangan Earth | `earth_slash` | SFX | Tidak |
| Serangan Wind | `wind_slash` | SFX | Tidak |
| Serangan Light | `light_slash` | SFX | Tidak |
| Serangan Dark | `dark_slash` | SFX | Tidak |
| Critical hit | `hit_crit` | SFX | **Ya** |
| Level up | `level_up` | SFX | Tidak |
| Coin / reward | `coin` | SFX | Tidak |
| BGM MainMenu | `bgm_mainmenu` | Music | — |
| BGM Battle | `bgm_battle` | Music | — |
| BGM Gacha | `bgm_gacha` | Music | — |
| BGM Guild/Market/HeroDetail | `bgm_guild`/`bgm_market`/`bgm_herodetail` | Music | — |
| Voice hero (opsional) | `voice_<id>` | Voice | **Ya** (duck BGM) |

---

## 12. Lampiran: Kode Lengkap & Glosarium

### 12.1 `AudioManager.ts` (Singleton Pembungkus)

```ts
// src/audio/AudioManager.ts
import Phaser from 'phaser';
import { AudioUnlock } from './AudioUnlock';
import { AudioSettings, loadAudioSettings, saveAudioSettings, DEFAULT_VOLUMES } from './AudioSettings';
import { BusGraph, BusId } from './BusGraph';
import { Ducking } from './Ducking';

export class AudioManager {
  private static _instance: AudioManager;
  readonly bus: BusGraph;
  readonly ducking: Ducking;
  settings: AudioSettings;
  private pendingBgm?: string;
  private currentBgm?: Phaser.Sound.WebAudioSound;

  private constructor(private scene: Phaser.Scene) {
    const sm = scene.sound as Phaser.Sound.WebAudioSoundManager;
    const ctx = sm.context;
    this.bus = new BusGraph(ctx);
    this.ducking = new Ducking(this.bus);
    this.settings = loadAudioSettings();
    this.applySettings(this.settings);

    AudioUnlock.attach(ctx, () => {
      if (this.pendingBgm) { this.playBgm(this.pendingBgm); this.pendingBgm = undefined; }
    });
  }

  static get inst(): AudioManager { return AudioManager._instance!; }
  static init(scene: Phaser.Scene): void {
    if (!AudioManager._instance) AudioManager._instance = new AudioManager(scene);
  }

  applySettings(s: AudioSettings): void {
    this.settings = s;
    (this.bus as any).setVolume('master', s.muted ? 0 : s.master);
    (this.bus as any).setVolume('music',  s.music);
    (this.bus as any).setVolume('sfx',    s.sfx);
    (this.bus as any).setVolume('ui',     s.ui);
    (this.bus as any).setVolume('voice',  s.voice);
    saveAudioSettings(s);
  }

  setBusVolume(bus: BusId, v: number): void {
    (this.settings as any)[bus] = Phaser.Math.Clamp(v, 0, 1);
    (this.bus as any).setVolume(bus, this.settings[bus as keyof AudioSettings] as number);
    saveAudioSettings(this.settings);
  }

  setMuted(m: boolean): void {
    this.settings.muted = m;
    (this.bus as any).setVolume('master', m ? 0 : this.settings.master);
    saveAudioSettings(this.settings);
  }

  playSfx(key: string, marker?: string, opts: Phaser.Types.Sound.SoundConfig = {}): void {
    if (!AudioUnlock.isUnlocked) return; // abaikan SFX sebelum unlock
    const v = this.settings.master * this.settings.sfx;
    this.scene.sound.play(key, { sprite: marker, volume: v, ...opts });

    // ducking untuk event krusial
    if (marker === 'gacha_reveal_l' || marker === 'hit_crit') this.ducking.duck('sfx');
  }

  playBgm(key: string): void {
    if (!AudioUnlock.isUnlocked) { this.pendingBgm = key; return; }
    this.currentBgm?.stop();
    const v = this.settings.master * this.settings.music;
    this.currentBgm = this.scene.sound.add(key, { loop: true, volume: v }) as Phaser.Sound.WebAudioSound;
    this.currentBgm.play();
  }

  stopBgm(): void { this.currentBgm?.stop(); this.currentBgm = undefined; }

  onContextSuspended(): void { this.stopBgm(); }
  onContextResumed(): void { /* BGM akan di-play ulang oleh state machine */ }
}
```

### 12.2 `main.ts` (Game Config)

```ts
// src/main.ts
import Phaser from 'phaser';
import { BootScene } from './scenes/BootScene';
import { PreloadScene } from './scenes/PreloadScene';
import { MainMenuScene } from './scenes/MainMenuScene';
// ... scene lain

const config: Phaser.Types.Core.GameConfig = {
  type: Phaser.AUTO,
  scale: { mode: Phaser.Scale.FIT, width: 720, height: 1280 },
  audio: {
    disableWebAudio: false,
    type: Phaser.WEB_AUDIO,
    noAudio: false,
  },
  scene: [BootScene, PreloadScene, MainMenuScene /*, ... */],
};

new Phaser.Game(config);
```

### 12.3 Contoh Pemanggilan di Battle (6 Elemen)

```ts
// src/scenes/BattleScene.ts
onAttack(element: 'fire'|'water'|'earth'|'wind'|'light'|'dark', isCrit: boolean): void {
  AudioManager.inst.playSfx('sfx', `${element}_slash`);
  if (isCrit) AudioManager.inst.playSfx('sfx', 'hit_crit');
}
```

### 12.4 Glosarium

| Istilah | Arti |
|---------|------|
| **AudioContext** | Engine audio browser (Web Audio API). Satu per game. |
| **AudioBuffer** | Data PCM audio yang sudah didecode, siap dimainkan (di RAM). |
| **AudioBufferSourceNode** | Node sumber yang memutar `AudioBuffer`. |
| **GainNode** | Node penguat/volume. |
| **Bus** | Jalur audio terpisah (Music/SFX/UI/Voice) dengan gain sendiri. |
| **Ducking** | Melemahkan sementara satu bus (BGM) saat bus lain aktif. |
| **Pre-decode** | Mendecode audio di awal (preload) agar `play()` instan. |
| **AudioSprite** | Satu file audio + json marker untuk banyak SFX. |
| **Autoplay policy** | Kebijakan browser memblokir audio tanpa gesture user. |
| **Unlock** | Memanggil `ctx.resume()` dari gesture untuk mengizinkan audio. |
| **latencyHint** | Petunjuk latency ke browser (`interactive`/`balanced`/`playback`). |
| **Compressor** | `DynamicsCompressorNode` anti-clipping di master. |
| **Pool** | Kumpulan instance SFX yang di-recycle (maks 16). |
| **CDP** | Canonical Design Parameters — parameter desain wajib. |

### 12.5 Checklist Pra-Release

- [ ] `audio.type = Phaser.WEB_AUDIO` di config.
- [ ] Semua audio di-load di `PreloadScene` (decode otomatis).
- [ ] AudioSprite untuk SFX kecil (satu file + json).
- [ ] Unlock listener sekali-pakai di `BootScene`.
- [ ] Tidak ada `play()` sebelum `AudioUnlock.isUnlocked`.
- [ ] 4 bus + master dengan volume matrix & ducking.
- [ ] Pool SFX maks 16 + compressor di master.
- [ ] `latencyHint: 'interactive'`.
- [ ] Setelan audio persist di `localStorage`.
- [ ] `visibilitychange` → `suspend()`/`resume()`.
- [ ] Penanganan `ctx.state==='closed'` (rebuild context).
- [ ] OGG + MP3 fallback via array loader.
- [ ] Voice opsional = bus terpisah, voice chat = TIDAK ada (A13).

---

> **Akhir dokumen** — `AUDIO_PIPELINE.md` v1.0. Seluruh parameter konsisten dengan CDP (6 elemen, rarity C/R/E/L, scene list, OGG+MP3, BGM per scene, autoplay unlock, target delay < 20 ms). Voice opsional, voice chat dihindari.
